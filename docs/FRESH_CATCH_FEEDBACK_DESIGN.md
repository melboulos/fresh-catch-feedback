# 🎣 Fresh Catch Feedback — Technical Design Specification

**Owner:** Mel Boulos (Couchbase)
**Workflow kind:** Agentflow (Rox)
**Status:** Production, verified end-to-end
**Last updated:** 2026-09-11
**Version:** 2.1 (post-run-naming, post-webhook-URL-swap)

---

## 1. Purpose

Capture per-rep feedback on Fresh Catch account and contact recommendations, so the next day's Fresh Catch research run can personalize its output: bias toward liked account patterns, hard-exclude blocked domains, and skip DQ'd contacts.

Feedback is emitted as HTTP clicks from pill buttons rendered in Fresh Catch emails and Home reports, and is persisted in a shared org-scoped custom store key that both Fresh Catch (reader) and a downstream stats agent consume.

---

## 2. System context

```
┌──────────────────────────┐        emits feedback pill URLs
│  🎣 Fresh Catch           │────────────────────────────────────┐
│  (research + report)     │                                    │
│                          │◀──── reads fresh_catch_feedback ───┤
└──────────────────────────┘   (org-scoped custom store)        │
                                     ▲                          │
                                     │ writes                   │
              ┌──────────────────────┴───────────────┐          │
              │  🎣 Fresh Catch Feedback              │          │
              │  (this workflow — webhook receiver)  │◀── POST ─┼──┐
              └──────────────────────────────────────┘          │  │
                                     ▲                          │  │
                                     │ hourly health check      │  │
              ┌──────────────────────┴─────────────┐            │  │
              │  🩺 Fresh Catch Webhook Monitor    │            │  │
              │  (cron, POSTs with test=1)         │            │  │
              └────────────────────────────────────┘            │  │
                                                                │  │
              ┌─────────────────────────────────────────────────┘  │
              │                                                    │
              ▼                                                    │
     ┌─────────────────────┐              ┌──────────────────────┐│
     │ Rep clicks 👍/👎/   │              │ 📊 Stats agent        ││
     │ 🎯/🚫 pill in email │──POST──▶     │ (reads feedback,     │◀┘
     │ or Home report      │  webhook     │ rolls up per rep)    │
     └──────────────────────┘             └──────────────────────┘
              │
              │ (via GitHub Pages intermediary)
              │
              ▼
     ┌─────────────────────────────────────────┐
     │ melboulos.github.io/fresh-catch-feedback │
     │ - Shows decorative confirmation UI       │
     │ - POSTs to webhook with JSON body       │
     └─────────────────────────────────────────┘
```

Four workflows/systems, one shared store:

- **Fresh Catch** — producer of recommendations, consumer of feedback for personalization
- **🎣 Fresh Catch Feedback** — this workflow, writes feedback to store
- **🩺 Fresh Catch Webhook Monitor** — hourly health check
- **📊 Stats agent** — rolls up per-rep click counts

Store: Rox custom store, key `fresh_catch_feedback`, scope `org`.

---

## 3. This workflow — 🎣 Fresh Catch Feedback

### 3.1 Metadata

| Field | Value |
|---|---|
| Name | 🎣 Fresh Catch Feedback |
| Kind | Agentflow |
| Trigger type | webhook |
| Webhook auth | disabled (`use_auth: false`) |
| Webhook URL | `https://webhooks.backend.rox.com/webhooks/w/workflow-webhook-003fa3e8` |
| Timezone | America/Los_Angeles |
| Run name template (settings) | null — runs are named dynamically in Step 0 via `set_run_name` |

### 3.2 Tools attached

```json
[
  { "package": "rox_actions", "action": "custom_store_get" },
  { "package": "rox_actions", "action": "custom_store_set" }
]
```

Built-in runtime tools also used:

- `set_run_name` — sets per-run display name; not listed in tools because built-in

### 3.3 Variables

None. The webhook URL is hardcoded in the GitHub Pages intermediary (`index.html`) rather than parameterized here.

### 3.4 Webhook contract

**HTTP method:** POST-only. GET requests return HTTP 404 with `{"message":"Not Found"}` because API Gateway routes are verb-specific.

**Headers required:** `Content-Type: application/json`

**Body:** JSON object with the fields below. `trigger_data.payload` in the runtime populates from this body.

**Synchronous response** (to caller):

- HTTP 200 with body `{"event_id":"<uuid>","status":"accepted"}` — this is the fire-and-forget ack, not the workflow's output
- The workflow's actual output ("Thanks — 👍 logged for...", etc.) is visible only in the trace, never in the HTTP response to the caller

### 3.5 Payload fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `rating` | string | ✅ | One of: `good`, `bad`, `opportunity`, `not_relevant`, `dq_contact` |
| `domain` | string | ✅ | Company domain; workflow lowercases before persisting |
| `rep` | string | ✅ | Rep's email; identifies which rep the click is attributed to |
| `test` | string | Conditional | If `"1"`, short-circuits — validates and returns confirmation but does NOT persist |
| `contact_name` | string | Required iff `rating=dq_contact` | Contact being disqualified |
| `contact_email` | string | Optional | Contact identifier for downstream matching |
| `contact_linkedin` | string | Optional | Contact identifier for downstream matching |

### 3.6 Rating semantics

| Rating | Meaning | Fresh Catch effect on next run |
|---|---|---|
| `good` | 👍 Rep likes this account pattern | 5× weight toward similar accounts |
| `opportunity` | 🎯 Strong buying signal, may become a deal | 5× weight (strongest positive signal) |
| `not_relevant` | 🚫 Wrong account for this rep | Hard-exclude domain forever |
| `dq_contact` | 👎 Wrong contact, right account | Account stays; specific contact never suggested again |
| `bad` | 👎 Legacy; deprecated but honored for old email links | Logged; Fresh Catch ignores during research |

### 3.7 Execution flow

The runtime agent follows a strict linear sequence. No LLM reasoning latitude — this is a persistence workflow.

```
┌──────────────────────────────────────────────────────────────┐
│ Step 0 — Name the run (set_run_name)                         │
│   - test=1               → "🩺 Health check"                  │
│   - rating=dq_contact    → "{rep} → dq_contact: {name} @ {d}"│
│   - anything else        → "{rep} → {rating} @ {domain}"     │
│   Runs BEFORE validation so every run is identifiable.       │
├──────────────────────────────────────────────────────────────┤
│ Step 1 — Read + validate payload                             │
│   test=1?               → return confirmation text, STOP     │
│   rating not in set?    → return error, STOP                 │
│   dq_contact w/o name?  → return error, STOP                 │
├──────────────────────────────────────────────────────────────┤
│ Step 2 — custom_store_get(fresh_catch_feedback, org)         │
│   Missing → treat as []                                      │
├──────────────────────────────────────────────────────────────┤
│ Step 3 — Build entry, append to array                        │
│   Entry shape depends on rating type (see 3.8)               │
├──────────────────────────────────────────────────────────────┤
│ Step 4 — custom_store_set(fresh_catch_feedback, org, array)  │
├──────────────────────────────────────────────────────────────┤
│ Step 5 — Return plain-text confirmation                      │
│   (visible only in workflow trace, not to caller)            │
└──────────────────────────────────────────────────────────────┘
```

### 3.8 Persisted entry shapes

**Account-level ratings** (`good`, `opportunity`, `not_relevant`, `bad`):

```json
{
  "rating": "good",
  "domain": "acme.com",
  "rep": "mel.boulos@couchbase.com",
  "timestamp": "2026-09-11T22:00:06+00:00"
}
```

**Contact-level DQ** (`dq_contact`):

```json
{
  "rating": "dq_contact",
  "domain": "acme.com",
  "contact_name": "Jane Doe",
  "contact_email": "jane.doe@acme.com",
  "contact_linkedin": "https://linkedin.com/in/jane-doe",
  "rep": "mel.boulos@couchbase.com",
  "timestamp": "2026-09-11T22:00:06+00:00"
}
```

Consumers discriminate on `rating == "dq_contact"` to route between shapes.

### 3.9 Response strings

Returned to the workflow trace (not the HTTP response body seen by the caller):

| Path | Response |
|---|---|
| `test=1` | `Test click received — no feedback was saved and production state was not modified.` |
| `good` | `Thanks — 👍 logged for {domain}. Tomorrow's Fresh Catch will find more accounts like this.` |
| `opportunity` | `🎯 Nice — opportunity logged for {domain}. This is the strongest positive signal we track.` |
| `not_relevant` | `🚫 Got it — {domain} added to the block list. We won't surface this company again.` |
| `dq_contact` | `👎 Got it — {contact_name} at {domain} is DQ'd. The account stays on future lists, we'll try different contacts.` |
| `bad` | `👎 Legacy feedback logged.` |
| invalid | `Error: {specific reason}` |

---

## 4. Upstream — pill link generation and GitHub Pages intermediary

### 4.1 Why an intermediary exists

The Rox webhook returns a plain-text response body meant for the workflow trace, not for end users. A direct browser click on the webhook URL would show raw text like `Thanks — 👍 logged for softwriters.com…` in a new tab — technically functional but poor UX.

A GitHub Pages static page (`melboulos.github.io/fresh-catch-feedback/`) sits between the pill click and the webhook:

1. Rep clicks pill in email → browser opens the GitHub Pages page with click params in URL
2. Page reads params, shows a styled confirmation card
3. Page's inline JavaScript POSTs to the Rox webhook with the click data as a JSON body
4. Fire-and-forget — page doesn't read the webhook response

### 4.2 Pill URL format (current — legacy)

Fresh Catch currently emits pills as:

```
https://melboulos.github.io/fresh-catch-feedback/?type={rating}&domain={domain}&rep={rep}[&test=1][&contact_name=...&contact_email=...&contact_linkedin=...]
```

Note: current pills use `type=` (legacy param name). GitHub Pages accepts both `type=` and `rating=` for backward compatibility.

### 4.3 Pill URL format (target — after Fresh Catch prompt update)

```
https://melboulos.github.io/fresh-catch-feedback/?rating={rating}&domain={URL_ENCODE(domain)}&rep={URL_ENCODE(rep)}[&test=1][&contact_name=...&contact_email=...&contact_linkedin=...]
```

URL encoding required for all values: `@` → `%40`, ` ` → `%20`, `:` → `%3A`, `/` → `%2F`, etc.

### 4.4 GitHub Pages intermediary — index.html

Static single-file page. Full source is committed to the `melboulos.github.io/fresh-catch-feedback` repo. Behavior summary:

- Parses query string params (`rating` OR legacy `type`)
- Validates rating against known set: `good`, `bad`, `opportunity`, `not_relevant`, `dq_contact`
- Builds JSON payload with all present fields (lowercases domain)
- Fires `fetch('POST', WEBHOOK_URL, {mode: 'no-cors', headers: {'Content-Type':'application/json'}, body: <payload>})` — fire-and-forget
- Renders emoji + title + body-with-highlights based on rating
- Status text: `"Test click — nothing was saved"` if `test=1`, else `"You can close this tab"`

`WEBHOOK_URL` constant in the page currently points to: `https://webhooks.backend.rox.com/webhooks/w/workflow-webhook-003fa3e8`

### 4.5 Fresh Catch consumption

Fresh Catch reads `fresh_catch_feedback` at the start of each run. Per-rep filtering and weighting happens inside Fresh Catch:

- 90-day window for `good` and `bad`
- Forever retention for `opportunity`, `not_relevant`, `dq_contact` (strongest signals persist)
- Top banner of each report: `"Personalized to your taste — {N} 👍, {M} 🎯, {K} 👎"`

Weighting in researcher prompt:

- 👍 `good` → 5× weight toward similar patterns
- 🎯 `opportunity` → 5× weight (same as `good`, kept separate for reporting)
- 👎 `bad` → deprioritize similar patterns (legacy; being phased out)
- 🚫 `not_relevant` → hard-exclude domain from candidate pool
- 👎 `dq_contact` → account remains eligible; contact filtered from suggestion pool

---

## 5. Downstream — stats agent

A separate workflow (📊 stats agent, not owned by this doc) reads `fresh_catch_feedback` and produces per-rep aggregate rollups (click counts by rating, leaderboards, etc.).

Contract preserved by this workflow: entries have stable `rep`, `rating`, `domain`, `timestamp` fields on every write, and `contact_*` fields on `dq_contact` writes. Any schema change here must be coordinated with the stats agent's read logic.

Health-check runs write nothing (short-circuit on `test=1`), so stats are not polluted by monitor traffic.

---

## 6. Shared state — fresh_catch_feedback

| Attribute | Value |
|---|---|
| Storage | Rox custom store |
| Key | `fresh_catch_feedback` |
| Scope | `org` (shared across all workflows in the organization) |
| Value type | JSON array of feedback entries |
| Growth | Append-only from this workflow; overwritten in full on every `custom_store_set` |
| Concurrency | Last-write-wins; no atomic append primitive |

### 6.1 Scope choice — why org and not workflow

- Custom store keys are namespaced by scope
- `workflow` scope isolates keys per workflow (each workflow would see a different `fresh_catch_feedback`)
- `org` scope is the only scope where multiple workflows (Fresh Catch, this workflow, stats agent, monitor) can read/write the same key
- Trade-off: any other workflow in the org can also read/write this key. Acceptable given internal use.

### 6.2 Concurrency model

- Every write is a full `[…]` overwrite (no append primitive)
- Two clicks within the same ~second race — the later write wins
- Rare in practice (per-rep clicks are human-paced)
- If concurrency corruption becomes measurable: migrate to per-event keys (`fresh_catch_feedback:{uuid}`) with a periodic reducer workflow

---

## 7. Health check — 🩺 Fresh Catch Webhook Monitor

Separate workflow (not owned by this doc, but part of the same system).

| Attribute | Value |
|---|---|
| Trigger | cron `0 * * * *` (hourly) |
| Timezone | America/Los_Angeles |
| Tools | `http.request_http`, `rox_actions.send_notification` |
| Notify recipient | Mel Boulos (`db14b483-5501-44aa-9181-64c8e45f9f1e`) |

Per-run behavior:

- POST synthetic payload to webhook: `{"rating":"good","domain":"healthcheck.internal","rep":"healthcheck@fresh-catch.internal","test":"1"}`
- Check response: healthy = HTTP 200 with body containing `"status":"accepted"`
- If unhealthy → send Rox notification with status code, response body, timestamp, and debug steps
- If healthy → silent success (no notification)

Trace signature in this workflow: hourly runs at `:00:0x` seconds, named `🩺 Health check`, 1 tool call (`set_run_name`), no store activity, ~3s runtime.

Why hourly: cheap enough to be trivial (~72s/day total runtime, 0 store writes), fast enough to catch outages same-day, silent when healthy so no notification fatigue.

---

## 8. Failure modes

| Failure | Detection | Behavior |
|---|---|---|
| Missing required param (`rating`, `domain`, `rep`) | Step 1 validation | Returns error, no store write, run named with partial info if possible |
| Invalid rating value | Step 1 validation | Returns error, no store write |
| `dq_contact` without `contact_name` | Step 1 validation | Returns error, no store write |
| `custom_store_get` returns `found: false` | Step 2 | Treated as `[]`; workflow proceeds normally (first-ever write) |
| Concurrent clicks | None (accepted risk) | Later write wins; earlier entry may be lost |
| Forwarded email — Alex clicks Mel's pill | None | Logged as Mel's click. Attribution error accepted (reps rarely forward per operational assumption) |
| Direct browser hit to webhook URL with GET | API Gateway | Returns HTTP 404 with `{"message":"Not Found"}`. Webhook is POST-only. Not a bug. |
| GitHub Pages page fails to load | Browser | Rep sees browser error page. Click is lost. |
| GitHub Pages `fetch()` fails (network, CORS, webhook down) | None | Rep sees success card as a lie. Click silently lost. This is what caused the Aug 24 → Sep 3 outage. Mitigated by the health-check monitor. |
| Webhook URL changes (trigger reset in Rox UI) | Health check within 1 hour | Notification fires; must update `WEBHOOK_URL` in `index.html` |
| Webhook returns 5xx | Health check within 1 hour | Notification fires |
| Anonymous URL manipulation | None | Anyone with the URL can post arbitrary `rep=` values. Low-impact given internal-only use. |

---

## 9. Incident history

| Date | Event |
|---|---|
| 2026-08-22 | Initial build. 4 ratings (good/bad/opportunity/not_relevant). Account-level entries only. |
| 2026-08-23 | Added `dq_contact` rating for contact-level DQ. Second entry shape introduced. |
| 2026-08-23 | Added `test=1` short-circuit (Step 1.5 → later folded into Step 1). Verified with sentinel domain. |
| 2026-08-24 | Last successful real pill click before Aug 24 → Sep 3 outage. Cause: unknown change to Fresh Catch's pill URLs or GitHub Pages page severed the pipeline. Symptoms: zero live runs for ~10 days. |
| 2026-09-03 | Discovered outage by manual click test. Diagnosed: webhook route stopped responding to GET; had been POST-only from the start but this was undocumented. Rox engineer clarified. |
| 2026-09-03 | Trigger swap (webhook → manual → webhook) issued new URL `003fa3e8`; old URL `c952284c` deprecated. |
| 2026-09-03 | `index.html` updated: POST with JSON body, dual `rating`/`type` param acceptance, new webhook URL. |
| 2026-09-03 | Built 🩺 Fresh Catch Webhook Monitor (hourly cron health check with `test=1`). |
| 2026-09-03 | Junk data cleanup: 3 `xxx@xxx.com` test entries removed from `fresh_catch_feedback`. |
| 2026-09-11 | Added Step 0 (`set_run_name`) so health checks and real clicks are distinguishable in the runs list. |
| 2026-09-11 | This design doc updated to v2.1. |

---

## 10. Operational notes

### 10.1 Testing

**From workflow builder chat:**
- Ask for a test payload → click Start on the proposed test-run widget

**From Rox UI:**
- Test/Run button → paste JSON `{"payload": {...}}` → Start

**From terminal (curl):**

```bash
curl -i -X POST https://webhooks.backend.rox.com/webhooks/w/workflow-webhook-003fa3e8 \
  -H "Content-Type: application/json" \
  -d '{"rating":"good","domain":"test.com","rep":"me@test.com","test":"1"}'
```

**From browser (real end-to-end):**
- Open `https://melboulos.github.io/fresh-catch-feedback/?rating=good&domain=test.com&rep=me@test.com&test=1`
- Should see 👍 card with "Test click — nothing was saved" status
- Confirms both GitHub Pages page and webhook are healthy

⚠️ Every test writes to the org store for real unless `test=1` is included.

### 10.2 Clearing the store

Custom store cannot be edited via RQL (RQL is read-only and doesn't cover custom store).

Options:

- Ask the general Rox chat assistant to overwrite the key with `[]` (or a selectively pruned array). Has direct `custom_store_*` access outside the workflow builder.
- Build a one-shot manual-trigger workflow with `custom_store_get` + `custom_store_set` (name suggestion: 🧹 Clear Fresh Catch Feedback).

### 10.3 Debugging

**"Nothing happens when I click a pill":**

1. Right-click pill → Copy link. Confirm URL starts with `melboulos.github.io/fresh-catch-feedback/`
2. Visit URL directly in a browser. Should show the confirmation card.
3. Check this workflow's Runs list within 30s. Should see a `type=live` run named after the rep+rating.
4. If steps 2 succeeds but no run in step 3 → GitHub Pages `fetch()` failed silently. Test with curl POST directly (see 10.1).
5. If curl POST succeeds but pill click doesn't → check GitHub Pages `index.html` deploy is current.

**Trace interpretation:**

- Empty trace + `status=failed` + ~1s duration → request rejected at webhook ingress before agent started. Almost always malformed request.
- Full trace + validation-rejection response → agent ran, validation caught it. Read the final output for the specific reason.
- 1 tool call (`set_run_name`) + short-circuit response → healthy `test=1` short-circuit path.
- 3+ tool calls (`set_run_name` + `custom_store_get` + `custom_store_set`) → healthy real-click path.

**"Store entries not appearing":** verify Fresh Catch is reading at `scope=org` (not `workflow`).

### 10.4 Runs list interpretation

Runs in this workflow now have named identifiers:

| Run name pattern | What it is |
|---|---|
| 🩺 Health check | Hourly monitor ping. Expected every hour on the hour. ~3s duration. 1 tool call. No store writes. |
| `{email} → {rating} @ {domain}` | Real account-level click from a rep. |
| `{email} → dq_contact: {name} @ {domain}` | Real contact-level DQ click. |
| Blank / partial name | Pre-2026-09-11 run (before Step 0 was added) OR a validation-rejection with incomplete params. |

---

## 11. Known limitations

- **Forwarding attribution error:** if a rep forwards their Fresh Catch email and someone else clicks, the click is attributed to the original recipient (the `rep=` in the pill URL). No mechanism to detect the actual clicker. Accepted risk per operational assumption that forwarding is rare.
- **Anonymous URL access:** webhook has `use_auth: false`, so anyone with the URL and a JSON POST can inject entries. Low-impact given internal-only use and low-stakes signal.
- **Silent client-side failures:** if `fetch()` on the GitHub Pages page fails (network, CORS, webhook down), the rep still sees the "success" card because `mode: 'no-cors'` swallows all errors. This is what caused the Aug 24 outage to go undetected for 10 days. Mitigated but not eliminated by the health-check monitor.
- **Duplicate clicks:** double-clicks and refresh-and-click-again both write duplicate entries. Stats agent must dedupe on its side if this matters (currently unaddressed).
- **Concurrency:** two rep clicks arriving within ~1s can race; later write wins, earlier entry lost. Rare in practice at human click cadence.
- **Two separate pill URL formats in circulation:** existing emails-in-inboxes use `type=`; future Fresh Catch versions will use `rating=`. GitHub Pages `index.html` handles both via fallback. Once all old emails are stale (~30 days), the fallback can be removed for cleaner code.
- **Stale prompt wording:** Step 1 of this workflow says "trigger payload contains query string params" but the actual delivery is a POST JSON body. Both work because `trigger_data.payload` populates from either — but the wording will confuse a future maintainer. Low priority to fix.

---

## 12. Future considerations

- **Auth on webhook:** enable `use_auth` on the trigger once threat model warrants it. Requires re-rendering all Fresh Catch emails to include the signing key — probably behind a feature flag.
- **Reducer workflow for concurrency:** if lost-write races become measurable, migrate from single-key append to per-event keys plus a reducer.
- **Retention policy enforcement:** currently enforced on the Fresh Catch read side (90-day window for `good`/`bad`). Nothing prunes the store itself. Add a scheduled 🧹 Prune old feedback workflow if entries pile up (>10k).
- **Cross-org sharing:** not supported — custom store `org` scope is per-org.
- **Per-rep private taste profile:** current design mixes all reps into one shared array. If reps want private taste (e.g. confidential DQ reasons), migrate to `fresh_catch_feedback:{rep_email}` per-rep keys.
- **Fresh Catch link generation cleanup:** eliminate `type=` → `rating=` translation once all legacy emails are stale. Track deprecation date.

---

## 13. Cross-workflow dependency matrix

| Component | Consumes | Produces | Owner |
|---|---|---|---|
| Fresh Catch | `fresh_catch_feedback` (read) | pill URLs in emails/reports, personalized report content | Mel |
| GitHub Pages `index.html` | pill URL query params | POST to webhook | Mel (melboulos.github.io repo) |
| 🎣 Fresh Catch Feedback (this workflow) | webhook POST body | `fresh_catch_feedback` entries (append) | Mel |
| 🩺 Fresh Catch Webhook Monitor | (cron) | webhook POST with `test=1`; notifications on failure | Mel |
| 📊 Stats agent | `fresh_catch_feedback` (read) | per-rep click rollups | Mel |

Any change to `fresh_catch_feedback` entry shape must be coordinated across:

- Fresh Catch (read logic)
- This workflow (write shapes in Step 3)
- Stats agent (read logic)

Any change to the webhook URL must be coordinated across:

- GitHub Pages `WEBHOOK_URL` constant in `index.html`
- 🩺 Fresh Catch Webhook Monitor URL constant

Any change to the pill URL format must be coordinated across:

- Fresh Catch (link generation)
- GitHub Pages `index.html` (param parsing — currently accepts both `type=` and `rating=`)

---

*End of document.*
