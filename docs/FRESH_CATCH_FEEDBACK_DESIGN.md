# 🎣 Fresh Catch Feedback — Technical Design Document

Owner: Mel Boulos (Couchbase)
Status: Deployed, verified end-to-end
Last updated: 2026-08-29

## 1. Purpose

Capture per-rep feedback on Fresh Catch account/contact recommendations so the next day's Fresh Catch run can personalize its research: bias toward patterns the rep liked, deprioritize patterns they disliked, hard-exclude domains they blocked, and skip contacts they DQ'd.

Feedback is emitted by rep clicks in a Fresh Catch email or Home report and is persisted in an org-shared key-value store that Fresh Catch reads at the start of each run.

## 2. System context

Two workflows, one shared store:

- **Fresh Catch** (producer of recommendations + reader of feedback)
- **🎣 Fresh Catch Feedback** (this workflow — writer of feedback)

Store: Rox custom store, key `fresh_catch_feedback`, scope `org`.

Rep clicks 👍/👎/🎯/🚫 in an email/Home report → hits the Fresh Catch Feedback webhook agent → writes to the shared org-scope store → Fresh Catch reads it on its next run. Rep sees a plain-text confirmation in the browser tab.

## 3. This workflow — 🎣 Fresh Catch Feedback

### 3.1 Metadata

| Field | Value |
|---|---|
| Name | 🎣 Fresh Catch Feedback |
| Kind | Agentflow |
| Trigger | Webhook (no auth, no output schema) |
| Timezone | America/Los_Angeles |
| Webhook URL | https://webhooks.backend.rox.com/webhooks/w/workflow-webhook-c952284c |
| Tools | rox_actions.custom_store_get, rox_actions.custom_store_set |

### 3.2 Trigger contract

Requests arrive as GET-style webhook calls with query string parameters, delivered to the agent as `trigger_data.payload`.

| Param | Type | Required | Notes |
|---|---|---|---|
| rating | string | ✅ | One of: good, bad, opportunity, not_relevant, dq_contact |
| domain | string | ✅ | Company domain; agent lowercases + URL-decodes |
| rep | string | ✅ | Rep's email; agent URL-decodes |
| test | string | ⚠️ conditional | If "1", short-circuit: acknowledge without persisting |
| contact_name | string | Required iff rating=dq_contact | Contact being disqualified |
| contact_email | string | Optional | Contact identifier for downstream matching |
| contact_linkedin | string | Optional | Contact identifier for downstream matching |

**Rating semantics:**

| Rating | Meaning | Fresh Catch effect on next run |
|---|---|---|
| good | 👍 Rep likes this account pattern | 5× weight toward similar accounts |
| opportunity | 🎯 Strong buying signal, may become a deal | 5× weight (strongest positive signal) |
| not_relevant | 🚫 Wrong account for this rep | Hard-exclude domain forever |
| dq_contact | 👎 Wrong contact, right account | Account stays; specific contact never suggested again |
| bad | 👎 Legacy; deprecated but honored for old email links | Logged but ignored during research |

### 3.3 Behavior

Execution follows a fixed sequence. The runtime agent has no LLM reasoning latitude — this is a persistence workflow, in and out.

1. **Step 1** — Read query params from `trigger_data.payload`
2. **Step 1.5** — Short-circuit: if `test=1`, return "Test click received…" and stop (NO custom store calls)
3. **Step 2** — Validate: rating ∈ {good, bad, opportunity, not_relevant, dq_contact}? else reject. If rating=dq_contact: contact_name required
4. **Step 3** — `custom_store_get(key=fresh_catch_feedback, scope=org)`. Missing → treat as `[]`
5. **Step 4** — Build entry, append to array (shape depends on rating type — see 3.4)
6. **Step 5** — `custom_store_set(key=fresh_catch_feedback, scope=org, value=<array>)`
7. **Step 6** — Return friendly plain-text confirmation (rendered as browser response body)

### 3.4 Entry shapes

Account-level ratings (good, opportunity, not_relevant, bad):

```json
{
  "rating": "good",
  "domain": "acme.com",
  "rep": "mel.boulos@couchbase.com",
  "timestamp": "2026-08-29T17:48:00+00:00"
}
```

Contact-level DQ (dq_contact):

```json
{
  "rating": "dq_contact",
  "domain": "acme.com",
  "contact_name": "Jane Doe",
  "contact_email": "jane.doe@acme.com",
  "contact_linkedin": "https://linkedin.com/in/jane-doe",
  "rep": "mel.boulos@couchbase.com",
  "timestamp": "2026-08-29T17:48:00+00:00"
}
```

Fresh Catch discriminates on `rating == "dq_contact"` to route between the two shapes.

### 3.5 Response strings

Returned as plain-text HTTP response body (rendered in the browser tab the rep clicked from):

| Rating | Response |
|---|---|
| good | Thanks — 👍 logged for {domain}. Tomorrow's Fresh Catch will find more accounts like this. |
| opportunity | 🎯 Nice — opportunity logged for {domain}. This is the strongest positive signal we track. |
| not_relevant | 🚫 Got it — {domain} added to the block list. We won't surface this company again. |
| dq_contact | 👎 Got it — {contact_name} at {domain} is DQ'd. The account stays on future lists, we'll try different contacts. |
| bad | 👎 Legacy feedback logged. |
| test=1 | Test click received — no feedback was saved and production state was not modified. |
| invalid | Error: {reason} (e.g. missing param, invalid rating) |

## 4. Shared state — fresh_catch_feedback

- **Storage:** Rox custom store
- **Key:** `fresh_catch_feedback`
- **Scope:** org (shared across all workflows in the org)
- **Value type:** JSON array of feedback entries
- **Growth:** Append-only from this workflow; overwritten in full on every `custom_store_set` call

**Why org scope, not workflow scope:** custom store keys are namespaced by scope. Workflow scope isolates keys per workflow (Fresh Catch and Fresh Catch Feedback would see different `fresh_catch_feedback` values). Org scope is the only scope both workflows can read/write to as a shared surface.

**Concurrency model:** every write is a full `[...]` overwrite (no atomic append/patch primitive). Two clicks within the same ~second could race — reader-side of `custom_store_get` doesn't lock. Rare in practice (per-rep clicks are human-paced); acceptable given the workflow's tolerance for occasional dropped entries.

**Future improvement:** `custom_store_increment` exists for counters but there is no append-primitive for arrays. If lost writes become a real problem, migrate to a keyed schema (one custom store key per rating event) with a separate reducer.

## 5. Fresh Catch integration (upstream)

Fresh Catch is the counterpart workflow. It is not owned or edited by this doc, but this workflow's contract is meaningful only in relation to it.

### 5.1 Link generation

For each account card in a Fresh Catch email/report, four buttons are rendered:

```
Base URL: {{ variables.feedback_webhook_url }}
         = https://webhooks.backend.rox.com/webhooks/w/workflow-webhook-c952284c

Account-level:
  👍 Good fit    → ?rating=good&domain={URL_ENCODE(domain)}&rep={URL_ENCODE(rep)}
  🎯 Opportunity → ?rating=opportunity&domain={URL_ENCODE(domain)}&rep={URL_ENCODE(rep)}
  🚫 Not relevant → ?rating=not_relevant&domain={URL_ENCODE(domain)}&rep={URL_ENCODE(rep)}

Contact-level (one per suggested contact):
  👎 DQ contact → ?rating=dq_contact
                  &domain={URL_ENCODE(domain)}
                  &rep={URL_ENCODE(rep)}
                  &contact_name={URL_ENCODE(name)}
                  [&contact_email={URL_ENCODE(email)} if present]
                  [&contact_linkedin={URL_ENCODE(url)} if present]
```

URL encoding rules: standard percent-encoding — `@` → `%40`, ` ` → `%20`, `:` → `%3A`, `/` → `%2F`, etc.

**Test-mode gating:** if Fresh Catch is rendering a test email, append `&test=1` to every link. This triggers this workflow's short-circuit (Step 1.5) so test clicks don't pollute production feedback.

**Empty-variable fallback:** if `feedback_webhook_url` is empty, Fresh Catch skips button rendering entirely rather than producing broken links.

### 5.2 Consumption

Fresh Catch reads `fresh_catch_feedback` at the start of each run (org scope). Per-rep filtering and bucketing happens inside Fresh Catch:

- 90-day window for good/bad
- Forever retention for opportunity (strongest signal)
- Forever retention for not_relevant (block list is permanent)
- Forever retention for dq_contact (contact blocklist per account)
- Signals surfaced in the top banner of the next report: "Personalized to your taste — {N} 👍, {M} 🎯, {K} 👎"

### 5.3 Weighting in the researcher

Passed into Fresh Catch's research prompt as taste signal:

- 👍 good accounts → 5× weight toward similar patterns
- 🎯 opportunity accounts → 5× weight (functionally treated same as good, kept as separate label for reporting)
- 👎 bad → deprioritize similar patterns (legacy; being phased out)
- 🚫 not_relevant → hard-exclude domain from candidate pool
- 👎 dq_contact → account remains eligible; specific contact filtered from suggestion pool

## 6. Failure modes

| Failure | Detection | Behavior |
|---|---|---|
| Rep clicks link with missing param | Step 2 validation | Returns `Error: Missing required query parameters: …`, no store write |
| Rep clicks link with typo rating (`?rating=goood`) | Step 2 validation | Returns `Error: Invalid rating value`, no store write |
| dq_contact link without contact_name | Step 2 validation | Returns `Error: dq_contact requires contact_name`, no store write |
| `custom_store_get` returns `found: false` | Step 3 | Treated as `[]`; workflow proceeds normally (first-ever write) |
| Concurrent clicks | None (accepted risk) | Later write wins; earlier entry may be lost |
| `test=1` from a production email | Step 1.5 | Short-circuits; production data never touched. Rep sees "Test click received" — surprising, but zero data risk |
| Direct browser hit to webhook URL (no query params) | Trigger ingress before agent starts | Failed run, 1s duration, no traces. Distinguishable from validation rejection: no `set_run_name` call, `status=failed` not `completed` |
| Webhook URL leaked externally | Trigger has `use_auth=false` | Anyone with the URL can inject entries. Mitigation options: (a) enable auth on the webhook trigger — issues a signing key, breaks all existing email links until re-rendered; (b) accept the risk — signal-quality attacks against a private feedback store have low ROI |

## 7. Verified test coverage

All scenarios verified via `type=test` runs during build-out. Every trace available via `get_workflow_run_traces`.

| Scenario | Run ID | Outcome |
|---|---|---|
| 👍 account-level good | earlier runs | ✅ Appended; friendly ack returned |
| 👎 dq_contact full payload | prior test | ✅ Contact-level entry with all optional fields; ack returned |
| test=1 short-circuit | 2096282b-8074-4d1f-bafa-277b3d37fde9 | ✅ Only `set_run_name` in trace; zero store calls; short-circuit ack |
| Missing all params | 66acd92a-9a07-4da7-8b9e-5f68e886a053 | ✅ Rejected with error; zero store calls |

## 8. Operational notes

### 8.1 Testing

- From workflow builder chat: request a payload and click Start on the proposed test-run widget
- From Rox UI: Test/Run button → paste `{"payload": {…}}` → Start
- From real webhook: paste the URL with query string in a browser tab

⚠️ All test methods write to the org store for real unless `test=1` is included in the payload.

### 8.2 Clearing the store

Custom store cannot be edited via RQL (RQL is read-only, and custom store isn't in the RQL object catalog).

Two options to clear `fresh_catch_feedback`:

1. Ask the general Rox chat assistant to overwrite the key with `[]` (outside the workflow builder — it has direct `custom_store_*` access)
2. Build a one-shot manual-trigger workflow with `custom_store_get` + `custom_store_set` (planned as 🧹 Clear Fresh Catch Feedback)

### 8.3 Debugging

- Empty trace + `status=failed` + ~1s duration: request rejected at webhook ingress before agent started. Almost always malformed URL or missing query params.
- Full trace + validation-rejection response: agent ran, validation caught it. Check `set_run_name` for context and the final output for the specific rejection reason.
- Missing store entries: check trace for `custom_store_set success: true`. If yes and entry still missing, verify Fresh Catch is reading at `scope=org` (not workflow).

## 9. Change history

| Date | Change |
|---|---|
| 2026-08-22 | Initial build. 4 ratings (good/bad/opportunity/not_relevant). Account-level only. |
| 2026-08-22 | Added dq_contact rating for contact-level DQ. Two entry shapes. |
| 2026-08-23 | Added test=1 short-circuit (Step 1.5). Verified with sentinel-domain test run. |
| 2026-08-29 | This design doc. |

## 10. Future considerations

- **Auth on webhook:** enable `use_auth` on the trigger once the leakage risk model warrants it. Requires re-rendering all Fresh Catch emails to include the signing key header — probably do this behind a feature flag on Fresh Catch's link generator.
- **Reducer workflow for concurrency:** if lost-write races become measurable, migrate from single-key append to per-event keys + periodic reducer.
- **Retention policy enforcement:** currently enforced on the Fresh Catch read side (90-day window for good/bad). Nothing prunes the store itself. Add a scheduled 🧹 Prune old feedback workflow if entries pile up materially (>10k).
- **Cross-org sharing:** not supported — custom store org scope is per-org. If Couchbase spins up sibling orgs, they'd each have their own `fresh_catch_feedback`.
- **Per-rep private taste profile:** current design mixes all reps into one shared array with per-entry rep field. If reps want private taste (e.g. confidential DQ reasons), migrate to `fresh_catch_feedback:{rep_email}` per-rep keys.
