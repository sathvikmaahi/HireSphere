## Why

Ten features have produced governed records — postings, Applications, rankings, notes, scorecards,
selections, offers, closures, audit rows, AI runs. **Nothing reads any of them in aggregate.** Every
surface built so far answers *"what happened to this record"*; this feature is the first that answers
*"how are we doing"*, which `access-control-and-admin`'s `access-control/audit-review` spec names as a
deliberately separate concern rather than an extension of audit search.

It is also **already owed**. Three sibling features deferred work here in their own verified
artifacts, none of it visible in `D.10`'s four one-line titles:

- `hiring-postings` `design.md` non-goal: *"No aging or SLA behavior. §29 item 9's posting aging
  thresholds are configuration this feature can read; **the nudges and dashboards that act on them are
  `insight-and-reporting`**."*
- `interview-pipeline` task 2.14: seed the interview and feedback aging thresholds, and assert no
  transition is driven by one — *"**aging drives `insight-and-reporting`'s SLA views**, not the state
  machine."*
- `matching-and-ranking` task 5.11 and `interview-pipeline` task 6.14 both verify divergence is
  computable from stored override records *"which is what makes **`insight-and-reporting`'s
  AI-agreement-rate dashboard** possible"*.

This feature exists to consume those substrates, not to rebuild them.

## What Changes

**Four dashboards covering §20.1's five rows.** `§20.1` specifies five; `D.10` funds four items. The
fifth is not missing — it is **already built elsewhere**, and one specified dashboard is absorbed:

| §20.1 row | Lands as |
|---|---|
| Practice Manager Dashboard | `TS-BL-071` |
| Recruitment Manager Dashboard | `TS-BL-072` |
| **Candidate Timeline** | **Not this feature** — `candidate-intake`'s `TS-BL-046` already builds it (its tasks 6.17 and 6.20) |
| AI Audit Dashboard | `TS-BL-074` |
| **SLA Aging Dashboard** | **Absorbed into `TS-BL-073`**, which becomes closure-readiness *and* aging |

No new backlog item is created, and no item moves between features. Both moves are item-boundary
refinements under `D.11`, recorded in `design.md` D1.

**The reporting read model is aggregate-only by construction.** This is the feature where candidate
PII would leak, so the boundary is structural and tested rather than documented: reporting queries
execute against a projection that holds no candidate-identifying column at all, drill-through is a
navigation evaluated on the target record rather than a report-side join, aggregates over cohorts
below a configured floor are suppressed together with the complements that would recover them by
subtraction, and an export is generated from the same projection under the same filters as the screen
that offered it. `D16`, `C-02` and `G-02` are the constraints; `design.md` D3 is the mechanism.

**Report Export is §25's existing eleventh row, registered — not invented.** *"Report Export | User
action | Retry export build only when safe. | Export file and audit event."* runs on `platform-core`'s
`TS-BL-006` dispatch chain as a job-type registration, with `SEC-012`'s classification label,
`§20.2` item 16's audit event, `PRV-007`'s permission gate and `§29` item 13's export limits. §25 stays
at eleven.

**These dashboards carry no charts.** Decided explicitly, not by omission: they are numbers, ranked
lists and tables on `design-system`'s `TS-BL-009` dense-data-table. `design.md` D2 records the
reasoning and the condition under which that should be revisited — a charting primitive would need a
categorical color family, which `design-spec.md` §1.6 forbids inventing at a call site and which
therefore belongs to `design-system`'s `TS-BL-007`, not here.

**Two metrics from an unratified recommendation are treated as unratified.** The *"Signature
dashboards"* row sits in `exploration-notes.md`'s **Recommendations made, not yet formally decided**
table — the same status class as `D23a`, closure semantics and reason-on-every-disposition. It is
**not** cited here as settled:

- **Resurfacing yield** is specified and built. Its substrate is `G-10`'s
  `candidate_posting_applications.source_type`, created in **Phase 2** by `candidate-intake`'s
  `TS-BL-046` (its task 6.12, its `design.md` D13) — not by the Phase 5 resurfacing engine. The metric
  therefore needs **no dependency edge to `TS-BL-075`**, and `D.10`'s omission of one is correct
  rather than an oversight. Until `TS-BL-075` populates the value, the surface renders a distinct
  *"no resurfaced applications yet"* state rather than a misleading `0%`. `design.md` D6.
- **AI agreement rate** ships whole, but its two halves rest on different substrates. *"How often
  approved scorecards diverge from AI drafts"* is computable from the override records
  `interview-pipeline`'s task 6.14 verifies. *"How often humans shortlist the AI's top picks"* is **not
  an override at all** — a Practice Manager who shortlists rank seven overrode nothing — so it is a
  join needing the AI rank as it stood at disposition time. No ranking record carries an activation
  timestamp, but **every ranking activation is an audited event that does**: `platform/audit-trail`
  records it with previous and new values and an immutable timestamp, `access-control/audit-review`
  makes that log searchable by target, action and time range, and `matching/ranking-score` returns a
  prior version's *"full entry set... as it stood"*. Because audit records are append-only at the
  database, a past period's rate is **immutable** once computed. `design.md` D7.

**One gap found in a verified sibling.** `access-control-and-admin`'s `TS-BL-022` seeds one page-catalog
entry per `§14.2` screen, and `§14.2` carries a single **Reports** screen whose capabilities include
both *"practice dashboard"* and *"AI audit"*. One page key cannot express the posture `C-02` and `D16`
require — granting the Auditor View to reach `TS-BL-074` would also grant them `TS-BL-071`'s
candidate-derived aggregates, while denying it makes `TS-BL-074` unreachable for the only role meant to
read it. Recorded as a cross-feature obligation, not worked around. `design.md` D4.

## Capabilities

### New Capabilities

- `reporting/read-model`: what a reporting query may return and what it may never return — the
  aggregate-only projection, small-cohort suppression and its differencing complement, drill-through
  by reference evaluated on the target, and the interactive-versus-export threshold.
- `reporting/report-export`: `§25`'s Report Export job type — registration on the existing dispatch
  chain, retry-only-when-safe, column-set parity with the on-screen view, classification label,
  export audit event, and configured limits.
- `reporting/workload-dashboard`: `§20.1`'s Practice Manager Dashboard — open postings, candidates by
  stage, pending interviews, scorecards due, selected candidates, offer status, closure blockers.
- `reporting/pipeline-oversight`: `§20.1`'s Recruitment Manager Dashboard — recruiter workload,
  postings, candidates submitted, shortlist conversion, interview scheduling, offer progress, aging —
  plus resurfacing yield.
- `reporting/closure-and-aging`: `§20.1`'s SLA Aging Dashboard and the cross-posting closure-readiness
  view, sharing one aging-and-blocker projection computed from `decision-and-offers`' closure
  predicate rather than a second one.
- `reporting/ai-audit-reporting`: `§20.1`'s AI Audit Dashboard for the Auditor — runs, prompt and model
  versions, overrides, failure rates, flagged outputs — over `ai-platform-governance`'s existing run
  log and `access-control-and-admin`'s existing audit search, plus the scorecard-divergence half of AI
  agreement rate.

### Modified Capabilities

None. Every substrate this feature reads is consumed at its existing contract; nothing this feature
needs requires a sibling's *shipped* requirements to change — including shortlist concordance, which
`design.md` D7 shows is answerable from requirements already written. The gaps found (`design.md` D4's
page-catalog granularity, D7's imprecise mechanism sentence in `matching-and-ranking`'s own D4, and
D12's `practice` vocabulary) are recorded as cross-feature obligations against their owning changes
rather than as delta specs here — `AGENTS.md`'s per-change rule.

## Impact

**Endpoints.** `§13.2`'s *Reports and Audits* block is claimed here except its two log-search routes:
`GET /api/reports/practice-dashboard` (`TS-BL-071`), `GET /api/reports/recruiter-workload`
(`TS-BL-072`), `GET /api/reports/aging` (`TS-BL-073`), `POST /api/reports/export` (`TS-BL-071`).
`GET /api/audit-logs` remains `access-control-and-admin`'s and `GET /api/ai-run-logs` remains
`ai-platform-governance`'s; `TS-BL-074` aggregates over them rather than replacing them. Two routes
`§13.2` does not name are added with reasoning in `design.md` D8: a cross-posting closure-readiness
route (§13.2 has only the per-posting `GET /api/postings/{postingId}/closure-checklist`) and an
AI-audit aggregate route. This also closes this feature's share of the residual risk
`interview-pipeline`'s `design.md` records — that *"§25's eleven job types and §13.2's endpoint list"*
had never been audited feature-by-feature against their owners.

**Consumed, not rebuilt.** `platform-core` `TS-BL-005` (aging thresholds as audited runtime
configuration, its task 5.11) and `TS-BL-006` (dispatch, retry, idempotency);
`access-control-and-admin` `TS-BL-018` (evaluator), `TS-BL-021` (audit search and its already-specified
audited, classification-labelled export), `TS-BL-022` (page catalog);
`ai-platform-governance` `TS-BL-028` (run log, override records, disclosure records — whose
`Run log access and search` requirement already names *"the reporting surface built on this search is
`insight-and-reporting`'s `TS-BL-074`"*); `design-system` `TS-BL-007`/`TS-BL-009`;
`decision-and-offers` `TS-BL-069` (the closure predicate, whose spec requires the checklist and the
guard share **one** predicate — so this feature adds a third reader, never a second predicate).

**Roles.** Recruiter, Practice Manager and Recruitment Manager read the operational dashboards;
Auditor reads only `TS-BL-074`; both Administrator roles read none of the candidate-derived
aggregates (`D16`, and `TS-BL-022` task 5.8's seeded no-View-on-candidate-personal-data posture).

**No application code changes in this change.** Planning only.

## Reconciliation with `talentsphere-wave-1-foundation`

Checked, as every feature does. **Clean, as expected but confirmed rather than assumed.**
`sprint-0-outcome.md` and `KNOWN_ISSUES.md` assign no old `D1`–`D18` decision to this feature, and
none of Sprint 0's seven carry-forward items touch reporting. Two entries are adjacent and worth
carrying:

- Sprint 0's *"Task 2.7 depends on a Sprint 6 capability"* note — an audited, permission-gated export
  built before an evaluator exists to gate it — is the same shape as this feature's export, and is why
  `access-control-and-admin` added its `TS-BL-021` → `TS-BL-018` edge. This feature's export inherits a
  finished evaluator and does not repeat the problem.
- `KNOWN_ISSUES.md`'s note that runtime-configuration audit events currently reach the structured log
  rather than `audit_logs` is resolved long before Phase 4; the aging thresholds this feature reads are
  audited configuration by then.

**`OD-003` is not in `KNOWN_ISSUES.md`.** The undecided AI model provider is `reference/spec.md`
`§35`'s `OD-003`, and `ai-platform-governance`'s task 1.18 is the still-unbuilt thing that will record
it there — the open obligation the cross-feature table already tracks after four features miscited it.
It is named here from `§35`, its actual source.
