# 🎣 Fresh Catch — Full Design Spec

*Daily prospecting brief agentflow for Couchbase sellers & SDRs*

---

## 1. Purpose & Positioning

**What it is:** A daily, email-first, ready-to-call prospecting brief for Couchbase sellers and SDRs.

**What it answers for the rep, every morning:**
- Who should I call today?
- Why now — what changed at this account?
- What do I say — what's the Couchbase angle and opener?
- How do I give feedback so tomorrow's list gets smarter?

**What it is NOT:**
- Not an executive summary
- Not a strategy dashboard
- Not a research report
- Not a web app — it's a tactical SDR call sheet delivered by email

**Scope:** US-based, net-new (not in Salesforce/Rox), high-fit lookalikes of Couchbase's won customers, with verified selling signals and compliance-screened senior contacts.

---

## 2. Runtime Environment

| Setting | Value |
|---|---|
| Kind | Agentflow (LLM-driven at runtime) |
| Trigger | `cron_schedule`, `0 8 * * *` in `America/New_York` (daily 8am ET) |
| Timezone | America/New_York |
| Task items | Enabled — Home shows a task per run |
| Run name | `🎣 Fresh Catch — {{ metadata.run.triggered_at }}` |
| Runs as | The rep who owns the workflow copy (`{{ metadata.user.email }}` / `{{ metadata.user.id }}`) |
| Two live copies | Fresh Catch - Prod (`preview_mode=false`) and Fresh Catch - Test (`preview_mode=true`) |

---

## 3. Variables

| Name | Type | Prod value | Test value | Purpose |
|---|---|---|---|---|
| `preview_mode` | bool | `false` | `true` | Production safety switch — the ONLY field that differs between prod and test |
| `feedback_webhook_url` | str | `https://melboulos.github.io/fresh-catch-feedback/` | same | Where 👍 🎯 🚫 👎 clicks POST |

---

## 4. Tools Attached (12)

| Package | Action | Used for |
|---|---|---|
| `agent_outputs` | `generate_agent_response` | Candidate research, signal verification, contact research |
| `agent_outputs` | `generate_webpage` | Email-first HTML call sheet |
| `rox_actions` | `lookup_accounts_by_domain` | Batch Salesforce dedup |
| `rox_actions` | `find_contact` | Compliance check (DNC/opt-out) |
| `rox_actions` | `enrich_person_details` | Contact enrichment |
| `rox_actions` | `enrich_phone` | Phone waterfall (batch) |
| `rox_actions` | `enrich_email` | Email waterfall backup (batch) |
| `rox_actions` | `create_people_list` | First-run Lookalike Prospects list creation |
| `rox_actions` | `add_lead_to_people_list` | Add cleared leads |
| `rox_actions` | `custom_store_get` | Load config, state, feedback |
| `rox_actions` | `custom_store_set` | Persist state (live only) |
| `email` | `send_email_as_user` | Send the daily brief |

> RQL data-read tools are built-in at runtime — never attached explicitly.

---

## 5. Org-Scoped State (`custom_store`)

All shared state lives at org scope so it works across every rep's copy.

| Key | Purpose | Written by | Read by |
|---|---|---|---|
| `fresh_catch_config` | Central tuning knobs (see §6) | You / admin | Every run, Step 1 |
| `previously_suggested_domains` | Domains ever suggested — hard-exclude list | Live runs, Step 4 | Every run, Step 1 |
| `fresh_catch_feedback` | 👍 🎯 🚫 👎 clicks captured by the feedback webhook | Fresh Catch Feedback (separate service) | Every run, Step 1 |
| `lookalike_prospects_list_id` | Public ID of the shared People list | First live run that creates it | Every live run, Step 8 |
| `fresh_catch_runs` | Audit log of runs, config versions, funnel counts | Live runs, Step 11 | Ad-hoc analytics |

> **Important:** Preview runs never write to any of these.

---

## 6. Tunable Config (`fresh_catch_config`)

Edit this one org-scoped key to change behavior across every rep's copy. No redeploy.

| Key | Default | Effect |
|---|---|---|
| `fit_threshold` | `7` | Min Couchbase fit score (1–10) |
| `final_count` | `10` | Companies in the brief |
| `min_candidates_target` | `25` | Working-list floor before stopping research |
| `max_research_passes` | `5` | Step 3 pass cap |
| `signal_recency_days` | `90` | Signal freshness window |
| `feedback_opportunity_weight` | `5` | 🎯 vs 👍 multiplier |
| `feedback_lookback_days` | `90` | Non-opportunity account feedback retention |
| `email_subject_template` | `🎣 Fresh Catch — {N} leads · {M} companies · call {top_company} first` | Placeholders: `{N}`, `{M}`, `{top_company}` |
| `icp_a_extra` | `[]` | APPEND to ICP-A defaults |
| `icp_a_exclude` | `[]` | REMOVE from ICP-A defaults |
| `icp_b_extra` | `[]` | APPEND to ICP-B defaults |
| `icp_b_exclude` | `[]` | REMOVE from ICP-B defaults |
| `config_version` | `"default"` | Logged to `fresh_catch_runs` per run |

> **Redeploy required (not tunable via config):** the ICP-A/ICP-B default customer lists themselves, Couchbase ICP context, quality bar rules, HTML layout, feedback model.

---

## 7. ICP Anchor Model

**ICP-A defaults (primary anchor — marquee US wins)**
Bucketed by industry: financial services, telecom, media/adtech, retail/F&B, travel/hospitality, gaming, healthcare, industrial/manufacturing/aerospace, enterprise software/business services, transportation/logistics/utilities/government. ~130 companies total.

**ICP-B defaults (secondary anchor — regional/mid-tier/digital-native)**
Same industry buckets, smaller-scale companies. ~25 companies total.

**REP anchor (optional, per-run)**
Committed forecast-stage opportunities owned by the running rep. Adds territory-shaped candidates without lowering the score threshold.

**Ordering rules**
- Prefer ICP-A over ICP-B on ties
- REP match never lowers the score threshold
- Effective lists = (defaults ∪ `*_extra`) − `*_exclude`, deduped case-insensitively

---

## 8. Couchbase ICP Context (fed to research LLM)

**Strong fit signals:**
- High-scale, low-latency operational workloads (personalization, catalogs, session stores, gaming leaderboards, IoT telemetry, payments, transactions)
- Mobile / edge / offline-first sync
- DB modernization from Oracle, DB2, early MongoDB, Cassandra, custom caching
- Cloud-native microservices, polyglot persistence
- AI / vector search + operational data
- Industries: financial services, retail/ecommerce, travel/hospitality, gaming, telco, healthcare, adtech, industrial, aerospace/defense, logistics

**Poor fit:** analytics-only, warehouse-only, small companies without engineering, competitor-locked shops without modernization triggers.

---

## 9. Feedback Model

**Account-level (bottom of each company card)**

| Button | Meaning | Effect |
|---|---|---|
| 👍 Good account | Rep likes this pattern | Weighted 1× as positive taste signal for future runs |
| 🎯 Opportunity | Real pipeline created | Weighted `feedback_opportunity_weight`× (default 5×). Kept forever. |
| 🚫 Never suggest | Hard exclude | Domain added to `blocked_domains` for this rep |

**Contact-level (next to each contact row)**

| Button | Meaning | Effect |
|---|---|---|
| 👎 DQ | This person is wrong | Contact added to `dq_contacts` org-wide, forever. Does NOT affect the account — the company remains eligible with different people. |

**Feedback retention**
- Account feedback: current rep only (`rep == {{ metadata.user.email }}`), last `feedback_lookback_days` (default 90d) except opportunity which is forever
- `dq_contact` entries: all reps, forever
- Legacy bad entries: ignored (do not use for research)

**Preview mode feedback**
All feedback URLs get `test=1` appended so the webhook service ignores the click. Prevents test runs from polluting production feedback.

---

## 10. Selling Signals

**What qualifies as a verified signal**
- Specific to the company
- Recent (within `signal_recency_days`, default 90)
- Named source + date
- Tied to a Couchbase talking point

**Acceptable sources:** Dated news, press releases, SEC filings, earnings-call transcripts, company blogs, product pages, active job listings, investor decks, dated executive LinkedIn posts, engineering blogs, GitHub repos, conference talks.

**Rejected sources:** Vague "industry sources," "widely reported," undated claims, generic growth claims, inferred scaling without a source.

---

## 11. Quality Bar (Final List Gates)

Every company in the final brief must pass ALL of these:

- ✅ US-based (HQ or substantial US engineering/commercial ops)
- ✅ Net-new (not in Salesforce/Rox org-wide, verified via `lookup_accounts_by_domain`)
- ✅ Couchbase fit score ≥ `fit_threshold` (default 7/10)
- ✅ At least one verified selling signal
- ✅ Not previously suggested
- ✅ Not in `blocked_domains` (🚫 feedback)
- ✅ Has at least one compliance-cleared contact with phone or email

> **Rule:** never pad the list. Zero-lead days are acceptable and still send an email.

---

## 12. Runtime Flow (11 Steps)

| Step | Action |
|---|---|
| 1 | Load config + state (`custom_store`: config, dedup, feedback, list ID) |
| 2 | RQL query rep's committed opportunities (territory anchor) |
| 3 | Research passes (up to N) — generate candidates, score, dedup vs SF |
| 4 | Persist scored domains to dedup list (LIVE ONLY) |
| 5 | Verify signals, drop signal-less, take top N by fit score |
| 6 | Research 2–3 senior contacts per final company, skip DQ contacts |
| 7 | Batch enrich phones + emails |
| 8 | Compliance screen (DNC/opt-out), add cleared leads to People list (LIVE ONLY) |
| 9 | Generate email-safe HTML call sheet |
| 10 | Email the report to the rep |
| 11 | Log run metadata (LIVE ONLY) |

**Anchor modes (Step 2 output)**

| Mode | Meaning |
|---|---|
| `rep+icp` | RQL query returned committed opps |
| `icp_only` | RQL succeeded but zero rows (no committed opps) |
| `rql_failed` | RQL errored, fell back to pure ICP research (run does not fail) |

**Batching rules**
- Domain dedup: one `lookup_accounts_by_domain` call per pass (batch)
- Phone enrichment: one `enrich_phone` call with people array
- Email enrichment: one `enrich_email` call with people array
- Contact research: one `generate_agent_response` for all final companies
- Never enrich per person

---

## 13. Preview Mode (Production Safety Switch)

Controlled ONLY by `{{ variables.preview_mode }}`. Trigger metadata is never used.

| Behavior | `preview_mode = false` (prod) | `preview_mode = true` (test) |
|---|---|---|
| Write to `previously_suggested_domains` | ✅ | ❌ |
| Write to `fresh_catch_runs` | ✅ | ❌ |
| Create/update Lookalike Prospects list | ✅ | ❌ |
| Add leads to People list | ✅ | ❌ |
| `test=1` on feedback URLs | ❌ | ✅ |
| `[TEST]` email subject prefix | ❌ | ✅ |
| Footer preview disclaimer | ❌ | ✅ |
| Still researches, enriches, generates report, emails | ✅ | ✅ |

---

## 14. Email + Report Structure (Step 9 Output)

**Delivery**
- `to`: `{{ metadata.user.email }}`
- `subject`: rendered from `email_subject_template`, `[TEST]` prefix if preview
- Zero-lead override: `🎣 Fresh Catch — no qualifying leads today`
- `body_format`: html
- Body = full HTML from `generate_webpage`
- Also saved to Home as a task item via `save_to_homepage="save"`

**HTML requirements**
- Gmail-safe: inline styles only, table layout, no CSS blocks, no scripts, no CDNs
- Headline: 🎣 Fresh Catch
- Subtitle:
  - Live: *N new leads at M US companies. Modeled on Couchbase won customers, deduped against Salesforce, verified signals only, compliance-checked.*
  - Preview: *N leads at M US companies (test preview — nothing added to Lookalike Prospects).*
- Preview footer: *Test run — no leads were added to Lookalike Prospects, no feedback clicks will be persisted, and production state was NOT modified.*
- Stat tiles: 2–3 max (Leads, Companies, Compliance blocked if >0)
- Personalization line: committed opp anchor / feedback taste counts / RQL fallback
- Immediately followed by company cards — no exec summary, no TOC, no strategy section, no funnel dashboard

**Company card contents**
- Fit badge
- Company name + domain
- Most similar to + tier (A/B/REP)
- Why-now signal headline
- Evidence with named source + date
- Couchbase angle
- Highlighted call opener
- Contact rows (name, title, phone, email, LinkedIn)
- Per-contact 👎 DQ button
- Account-level buttons: 👍 Good account, 🎯 Opportunity, 🚫 Never suggest

> ⚠️ **Known drift:** button visual styling isn't pinned in the prompt today, so it varies run-to-run. Fix is to add explicit button CSS spec to Step 9.

---

## 15. Deployment Model

**Two-agent topology**

| Agent | Purpose | `preview_mode` | Rep exposure |
|---|---|---|---|
| Fresh Catch - Prod | Live daily runs | `false` | Shared to real reps |
| Fresh Catch - Test | Safe validation before pushing changes | `true` | Owner only |

**Sync rule:** Any change to prod → paste identical instructions/tools/settings into test. Only `preview_mode` ever differs.

**Change types**

| Change | Redeploy required? |
|---|---|
| ICP-A/B customer list tweaks | ❌ Config only (`icp_*_extra` / `_exclude`) |
| Fit threshold, final count, research budget, signal recency, feedback weight, subject template | ❌ Config only |
| New tool, new step, changed logic, HTML redesign, new feedback type | ✅ Redeploy to each rep |

**Rollout workflow**
1. Edit prod agent instructions
2. Copy instructions to test agent
3. Run test agent, review email
4. If good → save prod, re-share to reps
5. If bad → iterate on test only

---

## 16. Observability

- Per-run task item on Home for the rep
- `fresh_catch_runs` org-scoped log captures: run date, rep, anchor mode, `committed_opps_count`, funnel counts (researched, cleared fit, SF hits, non-US drops, low-fit drops, signals dropped, DQ skipped, compliance blocks), `final_companies_count`, top_company, leads_created, feedback_signals_used, `config_version`, all `effective_*` config values
- Workflow run traces available via `get_workflow_run_traces` for debugging

---

## 17. Known Open Items

- **Button styling not pinned** — Step 9 prompt describes buttons semantically but not visually, so design drifts. Recommend pinning explicit CSS spec.
- **No email prompt template** — Step 9 rebuilds HTML from scratch every run. Consistency comes from prompt precision, not a template. Alternative: pin a full HTML skeleton in the instructions.
- **RQL fallback is silent** — if the deal query fails, the run continues in `icp_only` mode but the email doesn't tell the rep that happened. Could surface this in the personalization line.
- **No canary rollout mechanism** — config changes hit every rep on their next run. If you want staged rollout, add an `enabled_for_reps` key to config and have the agent check it in Step 1.
- **Fresh Catch Feedback is a separate service** (`melboulos.github.io/fresh-catch-feedback/`) — it's what writes `fresh_catch_feedback` to `custom_store` when buttons are clicked. Its uptime and behavior are outside this agent's control.
