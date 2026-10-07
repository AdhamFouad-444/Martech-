# Viaffinity RevOps Command Center — Build Plan

Master dashboard for HubSpot portal 48485556. Source of truth for deals, leads, contacts,
companies, meetings, calls, emails, tasks, sequences, workflows, marketing activity, and
deliverability for viaffinity.com + getviaffinity.com.

Audience: Emma (RevOps lead, Malin Advisory), Viaffinity leadership (Mason, Jeff), reps (JB, Foz).

Three questions it must answer at a glance:

1. What is happening across pipeline and GTM right now, and what should a rep do next?
2. Can we trust the data?
3. Are our domains healthy enough to keep sending?

## 1. Stack and shape

Two processes plus one file-backed store, orchestrated by `docker compose up`:

| Piece | Choice | Port | Why |
|---|---|---|---|
| API + sync engine | Python 3.12, FastAPI, APScheduler | 8000 | Workload is ingestion-heavy: HubSpot pagination, DMARC XML/zip/gz parsing, DNS lookups, scheduled snapshots. A service with an in-process scheduler is a cleaner fit than serverless API routes. |
| Frontend | React 18, TypeScript, Vite, TanStack Table + Query, Recharts | 5173 | Executive-grade tables and charts; Vite keeps dev fast. |
| Store | SQLite (WAL) via SQLAlchemy/Alembic | file | Zero-config local run, full history for trends, upgradeable to Postgres later without API changes. |

Single-command dev: `make dev` (or `docker compose up`) runs API + UI. Sits cleanly beside
the existing app on :3000 and n8n on :5678.

**Rejected alternative:** Next.js all-in-one. One codebase and one port is tempting, but the
workload is a sync engine first and a UI second; long-running ingestion jobs, XML parsing and
scheduled snapshotting fit a dedicated Python service better, and keeping UI and sync decoupled
lets us restart/deploy either side independently.

## 2. Auth and access

### HubSpot — OAuth app (required, not a private app)

The sequence performance endpoint (`GET /automation/v4/sequences/{id}/performance`, which
returns per-step sent/opened/clicked/replied/meetings counts and the enrollment summary)
**requires user-level OAuth** and rejects portal-level private-app tokens. Since Tab 1E
(sequence step metrics) is a core acceptance criterion, we build on a HubSpot OAuth app.

Scopes to request (finalized in the Phase 0 spike):

- `crm.objects.contacts.read`, `crm.objects.companies.read`, `crm.objects.deals.read`,
  `crm.objects.leads.read`, `crm.objects.owners.read`
- Engagement objects: meetings, calls, emails, tasks, notes read (all are CRM objects now)
- `crm.objects.tasks.write` — the ONLY write scope; task creation is the single write path
- `crm.schemas.*.read`, `crm.objects.*.sensitive.read` as needed for property discovery and
  workflow enrollment data on flows touching sensitive properties
- `automation` (workflows v4 read), `automation.sequences.read` (sequences + performance)
- Forms/marketing email/analytics read scopes — confirmed against the portal's tier in spike

Refresh token stored in the env/secret store; access tokens refreshed in memory. Guardrail:
the app constructs a write-capable client only inside the task-creation service; every other
module receives a read-only client instance.

### Google — one OAuth consent, two scopes

OAuth client for `emma@malin-advisory.com` with:

- `gmail.readonly` — DMARC aggregate report ingestion only (filter `subject:"Report domain:"`
  + attachment types; never read or store other email content)
- `postmaster.readonly` — Google Postmaster Tools (domain reputation, IP reputation, spam
  rate, SPF/DKIM/DMARC pass rates, encryption, delivery errors)

Operational note: gmail.readonly is a restricted scope. For an internal tool the OAuth client
stays unverified; we set the consent screen to "In production" (not "Testing") so refresh
tokens do not expire after 7 days. Emma accepts the unverified-app warning once.

### DNS

No auth. `dnspython` lookups hourly: SPF (`TXT` on root), DKIM (HubSpot selectors
`hs1._domainkey`/`hs2._domainkey` plus Google selector as configured), DMARC (`_dmarc` TXT,
assert `rua=mailto:emma@malin-advisory.com`), MX — for both domains.

## 3. Sync engine and data model

All tabs read only local data — the UI never calls HubSpot directly, so every number carries
a `last_refreshed` timestamp and every metric ships with a definition tooltip (guardrail).

```
HubSpot/Gmail/Postmaster/DNS -> connectors (rate-limited) -> normalize -> SQLite warehouse
                                                                     -> snapshot jobs
                                                                     -> computed tables
FastAPI /api/* -> React UI
```

Jobs (APScheduler):

| Job | Cadence | Notes |
|---|---|---|
| `hubspot_incremental` | 15 min | Watermark sync per object via `lastModifiedDate` search + associations |
| `snapshot_metrics` | daily | KPI values, fill rates, issue counts — powers all "trend over time" charts |
| `dmarc_poll` | hourly | Gmail inbox scan for new DMARC reports |
| `postmaster_pull` | daily | Postmaster stats per domain |
| `dns_check` | hourly | SPF/DKIM/DMARC/MX for both domains |
| `integrity_scan` | after each hubspot sync | Data-quality flags (Tab 2) |
| `action_queue_rebuild` | after each hubspot sync | MQL queue scoring (Tab 1H) |

**Design note — start accumulation early.** DMARC reports, Postmaster stats and fill-rate
trends are historical. Their ingestion jobs ship inside Phase 1's foundation (before their
UI tabs exist) so Phase 2 and Phase 3 open with real history instead of a day-one cliff.

Tables (SQLite):

- Mirror: `companies`, `contacts`, `deals`, `leads`, `owners`, `assoc_edges`,
  `meetings`, `calls`, `emails`, `tasks`, `notes`
- Sequences: `sequences`, `sequence_enrollments`, `sequence_step_stats`, `sequence_forecast`
- Marketing: `marketing_emails`, `forms`, `form_submissions`, `campaigns`, `workflows`,
  `workflow_stats`
- Ops: `sync_watermarks`, `metric_snapshots`, `fill_rate_snapshots`
- Deliverability: `dmarc_reports`, `dmarc_rows`, `postmaster_stats`, `dns_checks`,
  `deliverability_alerts`
- Computed: `data_issues`, `action_queue` (MQL candidates + scores), `audit_log`
  (every task-creation click recorded)

**Custom-property map.** Logical names in the spec (Account Motion, tier, Ecosystem LOB /
Size / Type, vertical, closed-lost reason, documented trigger) map to real HubSpot property
names in one config file (`config/properties.yaml`), populated by the Phase 0 property
discovery spike. Nothing else in the codebase hard-codes a property name — that keeps
spec-level wording decoupled from whatever the portal actually calls the fields.

## 4. Tab 1 — Revenue & GTM activity (Phase 1)

| Section | Build | HubSpot sources |
|---|---|---|
| Global filters | Filter bar persisted in URL + localStorage; every endpoint accepts date/owner/tier/motion/LOB/size/type/vertical/domain; Named vs Digital split everywhere | n/a |
| 1A Headline KPIs | KPI tiles with period selector (today/7d/30d/QTD/custom) + prior-period delta | deals, leads, meetings, calls, emails, tasks |
| 1B Deals | Open-deals board by stage w/ days-in-stage flag vs stage average; closed-won stage-velocity waterfall; closed-lost count/value/reason trend; segment cross-tabs; auto "Learnings & Recommendations" panel (deterministic insight rules, e.g. fastest-closing segment, bottleneck stage) | deals + pipeline stage history, companies |
| 1C Leads/contacts/companies | By source, lifecycle stage, owner, motion; lead→MQL and MQL→meeting funnels; lifecycle movement over time | leads, contacts, companies |
| 1D Rep activity | Meetings (booked/held/no-show/outcome/source), calls, 1:1 emails, tasks, notes per rep × tier × motion; activity-vs-outcome scatter | engagements, owners |
| 1E Sequences | Per-sequence card: owner, vertical, segment mix, steps, enrolled/completed/unenrolled-by-reason, per-step sent/open/click/reply/bounce/unsub rates, 7-day scheduled-send forecast; daily sequence-send chart stacked by sender + domain with agreed daily limit line | `/automation/v4/sequences*` performance + enrollments |
| 1F Marketing & workflows | Marketing emails/forms/landing pages/campaigns: sends, opens, clicks, submissions, contacts, influenced pipeline; workflow enrollments/completions/errors | marketing email, forms, campaigns, workflows v4 |
| 1G Copy intelligence | Subject/body ranking by reply rate (≥50 sends), "re-evaluate" list below portfolio median, top-5/bottom-5 side-by-side with full text | emails, sequence step stats, marketing emails |
| 1H Action queue | MQL-scored contacts ranked; engagement detail, intent signals, trigger + source link, suggested action; "Create rep task" button → `POST /api/tasks` → HubSpot task for record owner, audit-logged | contacts, companies, engagements |

**MQL rule engine** (all three must hold):

1. FIT — company embeds/could embed insurance, no in-house insurance agency, contact owns
   digital/small-business/strategy or is C-suite (title keyword map in config)
2. INTENT — multi-person engagement at the account, repeat clicks across different sequence
   steps, or a direct reply
3. DOCUMENTED TRIGGER — researched reason stored on the record (property name resolved in
   the access checklist below)

## 5. Tab 2 — Data integrity (Phase 2)

- Fill-rate table for every key property on deals/companies/contacts; daily snapshots give
  the trend.
- Red flags: deals missing amount/company/contact/close date/owner/stage-required fields;
  contacts missing email/company/vertical/tier; companies missing industry/employee
  count/domain/Ecosystem fields/motion; deals whose company is missing any of the above.
- Duplicates: companies by domain, contacts by email.
- Stale: untouched 90+ days; bounced contacts still enrolled/enrollable.
- Every flag row deep-links to the record: `app.hubspot.com/contacts/48485556/record/{object}/{id}`.
- Weighted Data Health Score, split Named Account vs Digital Marketing.

## 6. Tab 3 — Deliverability (Phase 3)

- Status card per domain: Green/Amber/Red with the reason.
- DNS panel: live SPF/DKIM/DMARC/MX, misconfig flags, DMARC `rua=` verification.
- DMARC reports: volume by source IP and sending service, SPF/DKIM alignment, unauthorised
  senders, trend, full report history (dedupe key: domain + org + report_id).
- Postmaster: domain/IP reputation, spam rate (amber >0.1%, red >0.3%), auth rates,
  delivery errors, daily trend. Low-volume domains return sparse data — UI handles empty
  states explicitly.
- HubSpot-side: bounce and unsubscribe rate per domain per sender.
- Alerts log: every status transition with timestamp.

## 7. Guardrails implementation

- Read-only by construction: one write-capable code path (`tasks_service.create_task`),
  reached only from the UI button; no update/delete/enroll code exists.
- Credentials only in env / secret store; never logged; `.env.example` documents names.
- Every metric tooltip shows definition + last-refreshed.
- Every chart gets a CSV export (`GET /api/export?chart=…` streams the underlying query).

## 8. Phases

| Phase | Contents | Exit demo |
|---|---|---|
| 0 — Access & feasibility spike | Create HubSpot OAuth app + Google OAuth client; run property discovery; verify sequences performance, workflows v4, campaigns endpoints return what Tab 1 needs; scaffold repo, docker compose, empty shell app | Verified access matrix + property map committed |
| 1 — Tab 1 | Foundation (sync engine + snapshots + DMARC/Postmaster/DNS jobs collecting silently) then 1A→1H in order; fixture mode (synthetic data) so UI work proceeds before OAuth lands | 10-record reconciliation vs HubSpot; all acceptance criteria for Tab 1 |
| 2 — Tab 2 | Integrity engine, dedupe, health score, trends | Flags link to exact HubSpot records; score reconciles to spot check |
| 3 — Tab 3 | DMARC UI (history already accumulating since P1), Postmaster, DNS, alerts | Both domains show live DNS + DMARC + Postmaster with trends |

Rough sizing: Phase 0 ~a day, Phase 1 is the bulk of the build, Phases 2–3 are each
substantially smaller than Phase 1 and overlap-friendly since their data jobs already run.

## 9. Risks

| Risk | Mitigation |
|---|---|
| Sequence step metrics need user-level OAuth (private app rejected) | Solved in design — OAuth app required; verify in P0 |
| Workflow error detail / enrollment history partially exposed via API | P0 verifies v4 coverage; surface what's available, flag gaps honestly |
| Campaign "influenced pipeline" not fully exposed | Approximate via campaign-associated deals/contacts; label the approximation |
| Postmaster sparse at low volume | Explicit empty states; never fake numbers |
| Google unverified-app refresh-token expiry (testing mode = 7 days) | Consent screen set to production status during setup |
| HubSpot rate limits (100 req/10s burst) | Token bucket, incremental watermarks, all reads cached locally |
| "Documented trigger" may not have a clean home in HubSpot today | Open question — needs a chosen property/note convention |

## 10. Access checklist (needed to start)

1. **HubSpot OAuth app** (or approval to create one) — client ID/secret; Emma or an admin
   authorizes once.
2. **Google Cloud OAuth client** for emma@malin-advisory.com (desktop/installed-app type is
   fine for local) — client ID/secret JSON.
3. **Agreed daily sequence send limit** — the number drawn as the limit line.
4. Property confirmations (or let the spike discover them): Account Motion, tier, Ecosystem
   LOB/Size/Type, vertical, closed-lost reason, documented trigger + its source.
5. Metric definitions to lock: MQL = lifecycle stage or lead status? Reply rate denominator
   = sent or delivered? Meeting rate = booked or held?
6. Where does it run — Emma's machine via docker compose, or a shared box/VPS later?
