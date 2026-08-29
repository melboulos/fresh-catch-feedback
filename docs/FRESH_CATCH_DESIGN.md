# 🎣 Fresh Catch — Technical Design Specification

Document version: 1.0
Design spec version: fc-v1.0
Owner: Mel Boulos (mel@couchbase.com)
Last updated: 2026-08-29

## 1. Purpose

Fresh Catch is an automated daily prospecting agent for Couchbase Account Executives (AEs) and Sales Development Reps (SDRs). Every weekday morning, it emails each AE a ready-to-call brief containing 5–10 net-new US companies that resemble Couchbase's won customers, with verified selling signals, senior contacts, ready-made call openers, and one-click feedback controls.

What Fresh Catch answers for the rep, every morning:

- Who should I call today?
- Why now — what changed at this account?
- What do I say — Couchbase angle and opener?
- How do I make tomorrow's list better?

What it is NOT:

- Not an executive summary or strategy report
- Not a research dashboard
- Not a web app — a tactical SDR call sheet delivered by email

## 2. Runtime environment

| Attribute | Value |
|---|---|
| Platform | Rox Workflows |
| Workflow kind | Agentflow (LLM-driven at runtime) |
| Trigger | Cron `0 8 * * 1-5` in America/New_York (weekdays 8am ET) |
| Timezone | America/New_York |
| Task items on Home | Enabled — Home shows a task per run |
| Run name template | 🎣 Fresh Catch — {{ metadata.run.triggered_at }} |
| Executing identity | Each rep's own workflow copy runs as that rep ({{ metadata.user.email }} / {{ metadata.user.id }}) |

Two deployed copies:

| Copy | Purpose | preview_mode | ui_test_mode |
|---|---|---|---|
| Fresh Catch - Prod | Live daily runs for real reps | false | false |
| Fresh Catch - Test | Safe validation before rolling changes to prod | true | true or false |

The only field that differs between copies is these two flag variables. Everything else is byte-for-byte identical.

## 3. Workflow variables

| Name | Type | Default | Purpose |
|---|---|---|---|
| preview_mode | bool | false (prod) / true (test) | Production safety switch. When true: run generates report but does NOT persist state or add leads. Prefixes email with [TEST]. |
| ui_test_mode | bool | false (prod) / true (test) | Fast UI-only path. Skips all Steps 1–8 real data operations, uses hardcoded sample dataset, generates HTML, sends [UI TEST] email. |
| feedback_webhook_url | str | https://melboulos.github.io/fresh-catch-feedback/ | Where 👍 🎯 🚫 👎 clicks POST. Same across prod and test. |

Note: support_by_rep was previously a variable but has been migrated to org-scoped custom_store to allow central management without per-agent redeployment.

## 4. Tools attached

12 catalog actions. Additional RQL data-read tools are built-in at runtime (not explicitly attached).

| Package | Action | Usage |
|---|---|---|
| agent_outputs | generate_agent_response | Candidate research, signal verification, contact research |
| agent_outputs | generate_webpage | HTML call sheet generation |
| rox_actions | lookup_accounts_by_domain | Batch Salesforce dedup |
| rox_actions | find_contact | Compliance check (DNC/opt-out) |
| rox_actions | enrich_person_details | Person profile enrichment |
| rox_actions | enrich_phone | Batch phone waterfall |
| rox_actions | enrich_email | Batch email waterfall (backup) |
| rox_actions | create_people_list | First-run Lookalike Prospects list creation |
| rox_actions | add_lead_to_people_list | Add cleared leads |
| rox_actions | custom_store_get | Load config, state, feedback, support_by_rep |
| rox_actions | custom_store_set | Persist state (live only) |
| email | send_email_as_user | Send the daily brief from the AE's own account |

## 5. Org-scoped state (custom_store)

All shared state lives at org scope so it works across every rep's copy of the workflow.

| Key | Purpose | Written by | Read by |
|---|---|---|---|
| fresh_catch_config | Central tuning knobs (see §6) | Admin (Rox chat or manager agent) | Every run, Step 1 |
| previously_suggested_domains | Domains ever suggested — hard-exclude list | Live runs, Step 4 | Every run, Step 1 |
| fresh_catch_feedback | 👍 🎯 🚫 👎 clicks captured by the Fresh Catch Feedback webhook service | External feedback service (see §11) | Every run, Step 1 |
| lookalike_prospects_list_id | Public ID of the shared People list for cleared leads | First live run that creates it | Every live run, Step 8 |
| fresh_catch_runs | Audit log of runs | Live runs, Step 11 | Ad-hoc analytics, weekly recap (future) |
| support_by_rep | AE → [SE_email, SDR_email] CC mapping | Admin manually or via manager agent | Every run, Step 1 → used in Step 10 |

Preview and UI test runs never write to any of these.

## 6. Tunable config (fresh_catch_config)

| Key | Default | What it controls |
|---|---|---|
| fit_threshold | 7 | Min Couchbase fit score (1–10) to keep a candidate |
| final_count | 10 | Companies in the final brief |
| min_candidates_target | 25 | Working-list floor before stopping research passes |
| max_research_passes | 5 | Step 3 pass cap |
| signal_recency_days | 90 | Signal freshness window |
| feedback_opportunity_weight | 5 | Weight multiplier: 🎯 opportunity vs 👍 thumbs-up |
| feedback_lookback_days | 90 | Non-opportunity account feedback retention |
| email_subject_template | 🎣 Fresh Catch — {N} leads · {M} companies · call {top_company} first | Placeholders |
| icp_a_extra / icp_a_exclude | [] | Append/remove ICP-A defaults |
| icp_b_extra / icp_b_exclude | [] | Append/remove ICP-B defaults |
| config_version | "default" | Version tag logged to fresh_catch_runs per run |

## 7. Per-rep CC mapping (support_by_rep)

Structure:

```json
{
  "ae@couchbase.com": ["se@couchbase.com", "sdr@couchbase.com"]
}
```

Current mapping — 27 AEs across Strategic West/East and Enterprise West/East, plus 3 SDR overlays (Poret Kyesmu, Wendy Litwak, Preston Cattanach).

Defensive normalization rules (Step 10): lowercase everything, coerce shapes, filter valid emails, dedupe, never self-CC, never invent entries, empty on any exception — never fail the run.

## 8. ICP anchor model

- **ICP-A** — ~130 marquee US Couchbase wins, by industry
- **ICP-B** — ~25 regional/mid-tier/digital-native companies
- **REP anchor** — dynamic, from RQL query of the running AE's committed forecast-stage opportunities

Anchor mode telemetry: `rep+icp`, `icp_only`, or `rql_failed`.

## 9. Quality bar (final list gates)

US-based · net-new · fit ≥ threshold · verified signal · not previously suggested · not blocked · corporate-stable · has a compliance-cleared contact. Never pad the list — zero-lead days send a "no qualifying leads" card.

## 10. Corporate instability filter

Disqualifying events within 180 days: M&A, rebrand, bankruptcy, major restructuring, layoffs >20%, loss of primary business, going-private. Non-disqualifying: new exec hires, funding/IPOs, layoffs <20%, product sunsets, non-acquisition partnerships. Err on the side of keeping if unverified.

## 11. Feedback model

| Button | Semantics | Persisted as | Effect |
|---|---|---|---|
| 👍 Good account | Rep likes this pattern | type=good | Weight 1× |
| 🎯 Opportunity | Real pipeline created | type=opportunity | Weight 5×, kept forever |
| 🚫 Never suggest | Hard exclude | type=not_relevant | Domain blocked for this rep |
| 👎 DQ (contact) | Wrong person for outreach | type=dq_contact | Contact DQ'd org-wide, forever — company stays eligible |

Preview mode appends `test=1` to all feedback URLs so the webhook ignores the click.

### External dependency: Fresh Catch Feedback service

| Attribute | Value |
|---|---|
| Deployed (production) URL | https://melboulos.github.io/fresh-catch-feedback/ |
| Local source repo | `/Users/melboulos/git/fresh-catch-feedback` (Mel's machine — GitHub Pages source for the deployed URL above) |
| Function | Receives GET requests from email button clicks, writes feedback events to `fresh_catch_feedback` org-scoped custom_store |
| Uptime | Outside this agent's control |
| Contract | Query params `type`, `domain`, `rep`, `email` (for DQ), optional `test=1` |
| Publish path | Push to the repo's default branch (e.g. `main` or `gh-pages`, per repo's Pages config) to update the live site |

## 12. Runtime flow — 11 steps

1. Load config, state, feedback, list ID, support_by_rep
2. RQL query rep's committed opportunities
3. Research passes — generate candidates, score, dedup vs SF
4. Persist scored domains [LIVE ONLY]
5. Verify signals + corporate stability, take top N
6. Research senior contacts, skip DQ contacts
7. Batch enrich phones + emails
8. Compliance screen, add cleared leads [LIVE ONLY]
9. Generate email-safe HTML call sheet
10. Resolve CC list, send email
11. Log run metadata [LIVE ONLY]

## 13. UI Test Mode

Skips Steps 1–8, uses hardcoded 3-company sample (TrailForge Retail, AeroGrid Systems, Finovo Payments), runs Step 9 normally, sends `[UI TEST]` email, skips Step 11. Precedence: ui_test_mode > preview_mode > live.

## 14. Email + report structure (Step 9 output)

Design lock: `design_spec_version: fc-v1.0`. 9-section HTML layout, frozen design tokens, Gmail-safe (inline styles, table layout, no `<style>`/scripts/external images/web fonts).

## 15. Deployment model

Two-agent topology (Prod shared to all reps, Test owner-only). Sync rule: only the two flag variables ever differ. Config/CC changes need no redeploy; structural/HTML changes do.

## 16. Observability

Per-run: Home task, email, workflow traces. Cumulative: `fresh_catch_runs` audit log with full funnel counts and effective config.

## 17. Data flow diagram

See runtime flow (§12) — custom_store reads at Step 1, RQL at Step 2, research/verification at Steps 3–6, enrichment at Step 7, compliance + list writes at Step 8 (live only), HTML at Step 9, email at Step 10, audit log at Step 11 (live only). Feedback clicks flow asynchronously from the deployed feedback service back into `fresh_catch_feedback`, consumed by the next day's Step 1.

## 18. Rules and invariants

Design lock, org-scope-only state, contact DQ doesn't affect account, never pad list, always send email, tunables live in config not instructions, CC mapping lives in support_by_rep, defensive normalization never fails the run.

## 19. Known limitations and future work

- Weekly recap agent (proposed, not built)
- Per-lead context columns on People list (proposed)
- `fresh_catch_daily_briefs` persistence (not implemented)
- No admin UI for support_by_rep (edited via raw JSON in Rox chat)
- No canary rollout mechanism (config changes hit all reps at once)
- Feedback service is a single point of failure outside agent control, with silent failure on the rep's end
- UI test mode uses hardcoded data only
- RQL fallback to icp_only is silent to the rep

## 20. Glossary

AE, SE, SDR, ICP, RQL, Custom store, People list, Design spec version, Preview mode, UI test mode — see full definitions in original spec.

## 21. Change log

| Version | Date | Change |
|---|---|---|
| fc-v1.0 | 2026-08-29 | Initial locked design spec |
| fc-v1.0 (doc update) | 2026-08-29 | §11 updated with local source repo path for Fresh Catch Feedback service |
