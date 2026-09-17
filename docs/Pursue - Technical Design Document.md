# Pursue — Technical Design Document

2026-09-16 · @u_oYMPV326oShi3ZruCZsG4w

This doc covers the Pursue agent, split out from the Fresh Catch design doc. Two contracts are shared between the two agents and kept identical on both sides — the ownership boundary and the pill/webhook integration — marked below as mirrored; Fresh Catch's copy lives in its own design doc.

## 1. Purpose and scope

*Placeholder — to be filled in with Pursue's own design (trigger, steps, storage, email/draft behavior). This stub currently holds only the two contracts shared with Fresh Catch, so the two agents can be edited independently without losing the shared interface between them.*

Known from the Fresh Catch side: Pursue owns deep person research, LinkedIn history mining, signal-progression tracking, Couchbase customer-reference research (evidence-gated: DIRECT\_MATCH / PARTIAL\_MATCH / NO\_MATCH + one-example-max), and email drafting from persisted lead context — triggered when a rep clicks the 🎯 Pursue pill in a Fresh Catch email.

## 2. Fresh Catch ↔ Pursue ownership boundary (FROZEN)

*Mirrored from the Fresh Catch design doc — update both sides together.*

Both agents share one record per contact, keyed by `pursuit_id`.

**Fresh Catch-owned fields** (Pursue reads, never writes): `pursuit_id`, `run_id`, `rep_email`, `run_date`, `account.*` (name, domain, fit\_score, tier, most\_similar\_to), `signal.*` (emoji, headline, source\_name, date, url), `why_now`, `couchbase_angle` (category-level only, no customer names), `implication`, `call_opener` (no customer names), `contact.*` (name, title, email, phone, linkedin\_slug), `status` (initialized to `open` by Fresh Catch on new records).

**Pursue-owned fields** (written only here): `why_you`, `customer_evidence.*` (qualification, customer\_name, use\_case, relevance, source\_url), `conversation_thesis`, and `status` transitions `open → drafted → sent`. All Couchbase customer-reference research lives here — Fresh Catch never names a customer in `couchbase_angle` or `call_opener`, so any customer-name mention in outreach copy must originate from Pursue's own evidence-gated research.

**Same-day rerun contract:** if Fresh Catch reruns same-day (same rep + domain + contact + run\_date), it reuses the existing `pursuit_id` and refreshes only its own fields — it never clobbers `drafted` or `sent` back to `open`. Pursue can rely on its own fields surviving a Fresh Catch rerun untouched.

## 3. Integration Contract — 🎯 Pursue pill / webhook (FROZEN)

*Mirrored from the Fresh Catch design doc — update both sides together.*

Fresh Catch renders every 🎯 Pursue pill href as:

```
https://rox-deep-dive-proxy.mel-boulos-97e.workers.dev/pursue-click?pursuit_id={pursuit_id}&rep_email={rep_email_urlencoded}&contact_email={contact_email_urlencoded}
```

When a rep clicks it: the Cloudflare Worker at that host validates the params, looks up the rep's Pursue webhook URL by `rep_email` (via org admin KV), signs the POST if a signing key is configured, and forwards to that rep's Pursue instance — returning an HTML confirmation page to the browser. This is Pursue's inbound entry point: a POST carrying `pursuit_id`, `rep_email`, and (unless the contact is LinkedIn-only) `contact_email`.

On receipt, Pursue is expected to: look up the shared record by `pursuit_id` in the per-rep lead-context store (`fresh_catch_lead_context:{rep_email_lower}`), read the Fresh Catch-owned fields (§2) as its research inputs, and write its own fields (`why_you`, `customer_evidence.*`, `conversation_thesis`) plus advance `status` `open → drafted` (and later `→ sent`) — without touching any Fresh Catch-owned field.

Rules that constrain what Pursue can assume about inbound requests: `rep_email` is always `{{ metadata.user.email }}` lowercased, URL-encoded; `contact_email` is omitted entirely (not empty, not null) for LinkedIn-only contacts — Pursue must handle that case by falling back to the LinkedIn slug already on the shared record; the pill (and therefore this webhook) is never invoked from preview or UI-test runs — every inbound hit corresponds to a real live-mode contact; `pursuit_id` is always the exact ID already persisted by Fresh Catch, never a value Pursue should regenerate or reconcile.

Persistence independence: Fresh Catch writes lead context for every compliance-cleared contact regardless of whether the rep is currently mapped in the Worker's webhook map — so a `pursuit_id` may exist and be readable by Pursue even before that rep is enabled for Pursue at all.
