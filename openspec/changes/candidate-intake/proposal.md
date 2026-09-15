# Candidate Intake — Resumes, Identity and Deduplication

## Why

`hiring-postings` built the left half of `domain-model.md`'s spine. This feature builds the other
half: **`Candidate ──1:N──▶ Resume`**, the pair that every `Application` (candidate × posting)
resolves to. `RankingScore = f(resume_version, jd_version, prompt_template_version,
model_version)` — two of those four terms do not exist until this feature ships, and neither does
anything for a posting to be ranked *against*.

**This is the first feature that ingests content from outside the company.** Every record built so
far was typed by an authenticated employee into a bounded field. A resume is an untrusted binary
uploaded by a recruiter, authored by someone with no system access at all, and read by a language
model. Three substrate guarantees that have only ever been asserted against employee-authored data
become load-bearing here for the first time: the gateway's untrusted-content isolation
(`AI-012`/`SEC-014`), the run log's reference-only rule, and the audit trail's no-personal-data
constraint. `reference/spec.md` §36 names *"prompt injection from resumes"* as a top risk, and
`D18` names the same thing while pointing out that the larger surface is the ranking stage
downstream — which is exactly why the isolation has to be right at this stage, where the untrusted
text first enters the system.

**This is also the first feature that holds candidate personal data.** `D16` made Administrators
config-only, `glossary.md` defines break-glass as the audited exception, and
`access-control-and-admin` built both against a database with no candidate in it. `SEC-003`,
`PRV-002`/`CAN-008` (contact data separated from scoring input) and `API-006` (contact fields
returned only with permission) all first apply to a real row here.

**Identity is decided before enrichment, deliberately.** `D18` chose hybrid parsing on one
specific ground: *"dedup keys come from deterministic logic rather than model output. Candidate
identity should not depend on a model's mood."* That single sentence is the shape of this whole
feature — deterministic extraction and identity are separable from, and prior to, everything the
model contributes.

**Nothing here is built, and nothing was inherited.** Sprint 0 of
`talentsphere-wave-1-foundation` shipped platform-layer code only; its handover states that its
~2,000 lines are "all of it platform-layer; **no hiring feature exists yet**, by design."
That change has no `candidate/` delta spec, no `D1`–`D18` decision from its `design.md` was
allocated here, and `KNOWN_ISSUES.md` carries nothing about resumes, parsing or candidates. Checked
rather than assumed — see `design.md` D12.

## What Changes

Eight backlog items, `TS-BL-041` through `TS-BL-048`, from
[D.10](../talentsphere/exploration-notes.md#d10--phases-2-5-backlog-decomposition-and-the-feature-list-is-now-finalized-2026-08-25).
None is built.

- **`TS-BL-041` Resume upload endpoint and malware scan.** `POST
  /api/postings/{postingId}/candidates/upload-resume` (§13.2), `CAN-002`'s validation set, the
  quarantine-then-promote object-storage path (`DOC-004`, `DOC-005`, `SEC-004`, `SEC-006`), and the
  malware scan registered as its **own job type** against `platform-core`'s dispatch pattern. The
  endpoint returns a job identifier and completes promptly (`API-007`); nothing about an external
  scanner's availability sits on an interactive request. **`CAN-002`'s six checks split five
  synchronous and one asynchronous** — see `design.md` D2, which is also where the scope boundary
  the async substrate's own spec names ("resume parsing") is settled. Depends on `platform-core`'s
  `TS-BL-001`, which is **done**.
- **`TS-BL-042` Resume file hash (`G-04`).** The hash computed over the bytes as uploaded, stored
  and indexed for lookup. `G-04`'s one sentence spans three items and this one carries only the
  first third: it computes and exposes the signal, `TS-BL-046` evaluates it as a row of `D23`'s
  table, and `TS-BL-048` acts on the same-candidate case by *not* creating a version.
  `design.md` D6 records that boundary, because the alternative is three items each half-building
  it. Depends on `TS-BL-041`.
- **`TS-BL-043` Deterministic contact-field extraction (`D18`'s deterministic half).** PDF text
  extraction, then regex and heuristics for email, phone, name, dates and links — plus `DOC-008`'s
  chunking into traceable sections, which §25 lists as an output of resume parsing and which
  `matching-and-ranking`'s embedding pipeline consumes. **This item is the "resume parsing"
  consumer that `platform-core`'s async-orchestration spec and §25 both name**, and it is a
  registered job type for a reason stronger than latency: its trigger is the scan job's clean
  verdict, so there is no interactive request in flight to be synchronous with (`design.md` D2).
  **No AI call occurs in this item.** Depends on `TS-BL-041`.
- **`TS-BL-044` LLM enrichment layer (`D18`'s interpretive half).** Skills, seniority, role
  summaries and domain experience, through the one governed egress, in the envelope
  `ai-platform-governance` `design.md` D3 already fixed — and specifically the case D3 anticipated
  by name: *"`resume_extraction` sends a resume version reference."* The request carries
  `resume_version_ref`, never resume content. **This is the first family whose output makes claims
  about a candidate**, so unlike `TS-BL-034` it must carry evidence references and source labels
  (`G-02`, `TS-BL-031`'s vocabulary) — the half of the contract D3 explicitly did *not* exercise.
  It **does not** register itself as a `TS-BL-006` job type: the gateway dispatches asynchronously
  on its behalf (`ai-platform-governance` D10). Depends on `TS-BL-043` and
  `ai-platform-governance`'s `TS-BL-027`.
- **`TS-BL-045` Extraction confidence scoring (`G-05`).** Per-section confidence and
  insufficient-information markers (`DOC-009`), a record-level roll-up in
  `candidate_resumes.extraction_confidence`, and the recruiter review queue that
  low confidence drives. `G-05` calls this "a third state between parsed and failed" in `D18`'s
  failure model; `design.md` D7 decides it is a **review state on the record, not a fifth
  `parsing_status` value**, because whether a parse ran and whether to trust it are different
  questions. Depends on `TS-BL-044`.
- **`TS-BL-046` Candidate identity model and dedup logic (`D23`).** The `candidates` table,
  normalized-name derivation, `D23`'s five-signal table evaluated as **five independent signals**,
  the review queue, the reversible audited merge, and the not-a-duplicate suppression list. Also
  where `D18`'s hard rule lands: **deterministic extraction failing blocks candidate creation until
  a human enters the contact fields**, because dedup depends on them. Depends on `TS-BL-041`.
- **`TS-BL-047` Duplicate backstop at posting level (`D23a`).** Posting-scoped pairwise comparison
  on parsed signals, surfaced as a **non-blocking advisory warning** on the ranked list, never an
  auto-merge. **`D23a`'s status in `exploration-notes.md` is "recommendation made, not yet
  confirmed" — the only decision in this backlog that is not settled.** `design.md` D1 makes the
  call explicitly rather than building through it: the two-check recommendation is treated as
  confirmed and why; the follow-on application-merge rule is **not**, and is excluded from this
  item's scope pending an owner decision. Depends on `TS-BL-046` and `hiring-postings`'
  `TS-BL-037`.
- **`TS-BL-048` Resume versioning (`resume_version_ref`).** Sequential immutable versions per
  candidate (`DOC-010`, `RET-002`), one active version, and the rule that mirrors
  `hiring-postings` D9 exactly: **an `Application` stays pinned to the resume version it was
  submitted with**, because re-pointing it would silently invalidate every ranking score computed
  against the prior version. Depends on `TS-BL-046`.

**Explicitly not in this change:**

- **No embedding, no vector store, no ranking.** `TS-BL-043` produces `DOC-008`'s traceable chunks;
  `matching-and-ranking`'s `TS-BL-049`/`TS-BL-050` embed them and `TS-BL-052` ranks. `VEC-001`'s
  source identifiers are satisfied by the chunk records this feature writes, not by an index it
  builds.
- **No OCR implementation.** `project.md` and `D18`'s consequences both say readiness only. §25
  lists OCR as its own job type; `TS-BL-043` **detects** a PDF with no extractable text layer,
  records the condition and `DOC-007`'s `ocr_used` flag, and registers no OCR job. §29 item 4's
  OCR-enablement switch exists and stays off.
- **No Application lifecycle, no shortlisting, no stages.** This feature creates the
  candidate-posting link and its source metadata (`CAN-005`, `CAN-007`, `G-10`); its *stage*
  machine, shortlisting and dispositions are `interview-pipeline`.
- **No application-merge resolution.** `D23a`'s follow-on rule — which of two applications on one
  posting survives, whether a re-rank is forced, what happens at differing stages — is deliberately
  out of scope. `design.md` D1 states the invariant this feature *can* commit to (a candidate merge
  never silently merges, withdraws or re-ranks an application) and names the two unproposed
  features that own the resolution.
- **No retention or deletion workflow.** `CAN-006` requires candidates stay searchable "according
  to retention and access rules"; `OD-004` (retention period) and `OD-009` (deletion, access and
  correction requests) are both open. `PRV-003`–`PRV-005`'s workflows are not built here, and
  §29 item 12's retention configuration key is read, not decided.
- **No new design-system components.** The Candidate Resume Intake screen (§14.2), the dedup review
  queue and the low-confidence review queue are assembled from `design-system`'s dense data table,
  forms, page templates, AI-disclosure and evidence-source-label components.
- **No AI gateway, no prompt registry, no permission evaluator, no dispatch substrate, no audit
  writer.** All five are consumed. `TS-BL-044` calls the gateway and authors one family's template
  version; it builds no egress and no queue.

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability below is
new. As with `hiring-postings`, none is preserved from `talentsphere-wave-1-foundation`: that
change has no `candidate/` delta spec at all.

- `candidate/resume-upload`: the upload endpoint, `CAN-002`'s validation set, quarantine and
  promotion of the stored object, and the malware scan as a registered job type. *(`TS-BL-041`)*
- `candidate/resume-hash`: the document-level hash signal — computed over the uploaded bytes,
  stored, and available for lookup. *(`TS-BL-042`)*
- `candidate/resume-extraction`: deterministic extraction of the identity-bearing fields and
  `DOC-008`'s traceable chunks, with no model involved. *(`TS-BL-043`)*
- `candidate/resume-enrichment`: model-produced skills, seniority and experience claims through the
  governed gateway, each carrying an evidence reference and a source label. *(`TS-BL-044`)*
- `candidate/extraction-confidence`: per-section confidence, insufficiency markers, and the review
  state that sits between parsed and failed. *(`TS-BL-045`)*
- `candidate/candidate-identity`: the candidate record, `D23`'s independent match signals, the
  review queue, the reversible merge, and the suppression list. *(`TS-BL-046`)*
- `candidate/duplicate-backstop`: the posting-scoped fallback comparison and its advisory warning.
  *(`TS-BL-047`)*
- `candidate/resume-versioning`: immutable sequential resume versions, the active version, and the
  pinning that keeps a ranking score explicable. *(`TS-BL-048`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**No overlap to resolve.** The five Phase 1 features each redistributed inherited
`talentsphere-wave-1-foundation` requirements, and `ai-platform-governance` recorded itself as the
last of the five to draw its inheritance across. None of it reached here: that change's delta specs
cover `access-control/`, `ai-platform/`, `design-system/`, `identity/` and `platform/`, and nothing
under `candidate/`. See `design.md` D12 for the check.

## Impact

**New application code** — `backend/app/candidates/` containing the resume intake pipeline (upload
validation, quarantine handling, scan job, extraction job), the deterministic extractor and its
chunker, the enrichment payload builder, the confidence model, the candidate aggregate with its
signal evaluator and merge machinery, and the posting-scoped comparison.
`backend/app/api/routes/candidates/` for the six endpoints §13.2 enumerates under Candidate Intake.
Frontend: the **Candidate Resume Intake** screen (§14.2), a candidate profile and timeline, a
resume-version list, a **dedup review queue** and a **low-confidence review queue** — all from
`design-system` components, adding none.

**New tables** — `candidates`, `candidate_resumes`, `candidate_posting_applications`,
`resume_text_chunks`, and the dedup review/suppression/merge records `D23` requires. All by
reversible migration, applied by the migration identity. `candidate_posting_applications` is
created here with its identity, source metadata and resume pin only; its `status`, `latest_rank`,
`latest_match_score`, `mandatory_criteria_status`, `shortlist_reason` and `final_outcome` columns
are written by `matching-and-ranking`, `interview-pipeline` and `decision-and-offers`.

**New object storage and one new external dependency** — a private bucket with a quarantine prefix
and a resume prefix, signed or proxied access only (`SEC-004`), plus the malware scanner whose
endpoint and timeout are §29 item 5 configuration. The scanner is the first external service this
product calls other than Hubble and the model provider, and `G-12`/`NFR-004`'s degradation rule
applies to it: a scanner outage queues uploads, it does not break the candidate database.

**Existing capability consumed, not modified** — `platform-core`'s async dispatch pattern (two new
job types), its audited runtime-configuration path (§29 items 2, 3, 4, 5 and 12), its notification
engine (review queues) and its correlation identifier across the async boundary;
`access-control-and-admin`'s permission evaluator, its `assigned-postings` scope predicate (which
is what `CAN-001` resolves to) and its durable audit writer; `ai-platform-governance`'s gateway,
run log, prompt registry, evidence-labeling substrate and AI-disclosure marking; `hiring-postings`'
`job_postings` record; `design-system`'s table, forms, templates and labels.

**One prompt family moves from stub to authored.** `resume_extraction` — one of the ten
`TS-BL-030` registered with stub text on purpose — gets real content, promoted as a **new template
version** through the registry's versioning and promotion gate. The gate refuses production
activation with no passing corpus run recorded, so the family is deployable to Dev and not to
production until `TS-BL-032`'s harness carries cases for it. Its adversarial cases matter more than
`hiring-postings`' two did: this is the first family whose input is a document the company did not
write.

**Downstream features that block on this one** — `matching-and-ranking`'s `TS-BL-049` (embedding,
on `TS-BL-044`) and everything after it, `interview-pipeline`'s whole chain, and
`resurfacing-and-communications`' `TS-BL-075`. `matching-and-ranking`'s `TS-BL-053` (Ranking Board
UI) is where `TS-BL-047`'s advisory warning renders, which is the one place this feature reaches
into an unproposed feature's surface — recorded in `design.md` D1 as part of the D23a scoping call.

**External constraints carried, not solved here** — **`OD-004` (candidate data retention period,
owned by Product/HR/Legal) and `OD-009` (candidate deletion, access, correction and withdrawal
handling, owned by Legal/HR/Product) are both open**, and this is the feature that creates the data
they govern. Neither blocks: the retention interval is configuration (§29 item 12) and `RET-007`'s
source-linkage requirement — derived data linked back to its source for deletion and re-indexing —
is built now regardless of when the policy lands, because retrofitting it later means finding every
derived record without a link. **`OD-003` (the AI model provider) is open**; this feature calls the
gateway and never a provider, so it costs nothing here. **`D22`'s staleness windows cap
configurable values by retention**, so the two must be set consistently once `OD-004` answers —
noted, not resolved.
