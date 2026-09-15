# Hiring — Job Descriptions and Postings

## Why

`project.md`'s core workflow starts with two boxes — **Job Description → Job Posting** — and every
one of the eleven boxes after it hangs off the second. `domain-model.md` is explicit that the
posting is what the spine attaches to: an `Application` is *candidate × posting*, and a
`RankingScore` is a function of `jd_version` among other things. Until a posting exists, there is
nothing for a resume to be submitted against, nothing for a ranking to be computed for, and nothing
for a vacancy slot to belong to.

**This is the first feature that produces a hiring record rather than a substrate.** Every Phase 1
feature built a mechanism and registered no domain content — `platform-core`'s workflow engine ships
with "no posting, Application, offer, or closure machine defined," `ai-platform-governance`'s
registry holds ten prompt families whose text is a stub, and `access-control-and-admin`'s
`assigned-postings` scope predicate filters over `job_postings.recruiter_ids[]`, a column that does
not yet exist. This feature is where those three become exercisable against something real.

**Why it must come before `candidate-intake` and `matching-and-ranking`, not alongside them.** Three
downstream items name an artifact this feature owns as their enabler: `TS-BL-047` (posting-level
duplicate backstop) cannot scope anything to a posting until postings exist, `TS-BL-052` (AI Ranking
Engine) ranks against a posting, and `TS-BL-075` (resurfacing) is triggered by a posting opening
(`JOB-010`). The dependency is not a sequencing preference; it is what those items are about.

**Nothing here is built.** Sprint 0 of `talentsphere-wave-1-foundation` shipped platform-layer code
only — its own handover states, in those words, that "**no hiring feature exists yet, by design**."
Unlike the five Phase 1 features, this one inherits nothing at all: `talentsphere-wave-1-foundation`
has no `hiring/` delta spec, and no `D1`–`D18` decision from its `design.md` was ever allocated here
(`platform-core` `design.md` D11 distributed all eighteen across the five Phase 1 features). See
`design.md` D10.

## What Changes

Eight backlog items, `TS-BL-033` through `TS-BL-040`, from
[D.10](../talentsphere/exploration-notes.md#d10--phases-2-5-backlog-decomposition-and-the-feature-list-is-now-finalized-2026-08-25).
None is built.

- **`TS-BL-033` JD drafting workspace — the Practice Manager drafts, manually.** The
  `job_descriptions` and `job_description_versions` tables, the typed structured-input schema behind
  `JOB-002`'s mandatory fields, and the JD Workspace screen (§14.2) built on `design-system`'s form
  controls and page templates. **The Recruiter is not in this flow** — `C-12` removed them, per §8's
  listing of JD Workspace users as Practice Managers and Recruitment Managers. Every free-text
  section carries a declared length bound from the moment the schema exists, not from the moment AI
  starts filling it (`S.5`, and `ai-platform-governance` D7). Depends on `access-control-and-admin`'s
  `TS-BL-018`.
- **`TS-BL-034` AI-drafted JD generation.** The `job_description_generation` prompt content and
  output contract, invoked through `ai-platform-governance`'s gateway using **the exact envelope its
  `design.md` D3 already fixed** — `jd_draft_ref` and bounded `notes` in, `run_ref`/`status`/output
  or typed failure out. This feature writes the payload inside that envelope, which D3 explicitly
  left open: *"no JD schema is fixed, and `hiring-postings` remains free to refine its own input
  shape."* Output is labeled AI-generated until a human approves it (`AI-001`, `UI-004`) and can
  never be the thing that approves it (`JOB-005`). Depends on `TS-BL-033` and `TS-BL-030`.
- **`TS-BL-035` JD approval workflow — the Recruitment Manager approves.** The RM review queue with
  an approve / request-changes loop, per `C-12`'s resolution of `D12`. Carries the **fallback rule
  that resolution made mandatory**: where no Recruitment Manager is assigned, approval falls to
  another Practice Manager — **never the drafter**, because that is the self-approval option `D12`
  rejected wearing a different hat. Depends on `TS-BL-034`.
- **`TS-BL-036` JD versioning and fork-on-edit.** Editing an **approved** JD creates a new version
  rather than mutating one, and version comparison covers AI-generated against human-edited content
  (`JOB-006`). Live postings stay pinned to the version they were published against — the rule
  `D12` retained through `C-12`, and the reason it exists is that edit-in-place would silently
  invalidate every ranking score computed against that JD. Depends on `TS-BL-035`.
- **`TS-BL-037` Job Posting creation.** The `job_postings` table, `vacancy_type: FINITE(n) |
  EVERGREEN` (`D14`), the link to exactly one approved JD version (`JOB-007`), owner and recruiter
  and panel assignment, and the posting metadata `R.4` enumerated (`work_mode`, `employment_type`,
  `priority`, `target_start_date`). **This item is what makes `access-control-and-admin`'s
  `assigned-postings` scope predicate exercisable** — its authorization spec cites
  `job_postings.recruiter_ids[]` by name. Depends on `TS-BL-036`.
- **`TS-BL-038` Posting status lifecycle.** §11.1's machine — `draft → pending_approval → open →
  screening → interviewing → scorecard_review → selection → offer → onboarding → filled`, plus
  `paused`, `cancelled`, `closed` — registered declaratively against `platform-core`'s workflow
  engine, which ships with none. `JOB-009`'s open-validation checklist, and `JOB-010`'s
  posting-opened event emitted with **no handler registered**. **Every transition this item
  registers is actor-initiated**, which is precisely the room `EVERGREEN` needs — see below. Depends
  on `TS-BL-037`.
- **`TS-BL-039` Posting text stored separately from the internal JD.** `JOB-008`/`G-08`: the
  external posting text is a distinct artifact with its own lifecycle, not a rendering of the JD.
  Carries the `job_posting_generation` prompt content as well as the separation, because `G-08`'s
  own stated rationale for the separation is that it "gives the AI a second clearly-scoped
  generation target" — see `design.md` D11. Depends on `TS-BL-037`.
- **`TS-BL-040` Compliance text handling.** `G-09`: `compliance_text_status: missing | valid |
  not_required`, checked by posting-open validation, **defaulting to `not_required`** so nothing is
  blocked while Legal works `OD-005`. The default is a runtime-configuration value rather than a
  code constant — §29 item 11 already lists "compliance disclosure text for job postings" as
  configurable without code changes — which is what makes `G-09`'s "the default flips and the gate
  begins enforcing" a configuration change rather than a rebuild. Depends on `TS-BL-039`.

**Explicitly not in this change:**

- **No evergreen lifecycle rules.** `D14`'s manual-pause/close-only behavior, the no-5-cap
  consequence, the staleness nudge, and per-hire time-to-fill are `decision-and-offers`'
  `TS-BL-070`, which depends on `TS-BL-038` for exactly this reason. `TS-BL-037` introduces
  `EVERGREEN` as a value and `TS-BL-038` leaves room for it; neither builds its lifecycle. See
  `design.md` D5.
- **No closure rules.** "Onboarded + valid Hubble ID" (`G-13`) is `TS-BL-069`. This feature registers
  `closed` as a state and no automatic transition into it.
- **No vacancy slots.** §12.2's `vacancy_slots` are reserved and filled by priority selection
  (`TS-BL-064`) and the offer path. `TS-BL-037` records `vacancy_count`; it creates no slot rows.
- **No candidate matching, ranking, or resurfacing.** `TS-BL-038` emits `JOB-010`'s event;
  `matching-and-ranking` and `resurfacing-and-communications` register the handlers.
- **No workflow engine, no permission evaluator, no AI gateway, no prompt registry.** All four are
  consumed. This feature registers a state machine, declares permission requirements, calls the
  gateway, and promotes a registered family's template version — it builds none of the four.
- **No new design-system components.** The JD Workspace and Posting Detail screens are assembled
  from `design-system`'s form controls, dense data table, page templates, and AI-disclosure label.

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability below is new.
Unlike the five Phase 1 features, none of these paths is preserved from
`talentsphere-wave-1-foundation`: that change has no `hiring/` delta spec at all.

- `hiring/jd-workspace`: the job description record, its typed structured-input schema with a
  declared bound on every free-text section, and the drafting surface the Practice Manager works in.
  *(`TS-BL-033`)*
- `hiring/jd-ai-drafting`: AI-assisted drafting of a job description through the governed gateway —
  the request payload, the output contract, and the marking that survives until a human approves.
  *(`TS-BL-034`)*
- `hiring/jd-approval`: the separation-of-duties gate — Recruitment Manager approves, the drafter
  never does, and the fallback when no Recruitment Manager is assigned. *(`TS-BL-035`)*
- `hiring/jd-versioning`: immutable approved versions, fork-on-edit, AI-versus-human comparison, and
  the pinning that keeps a live posting's ranking scores meaningful. *(`TS-BL-036`)*
- `hiring/job-posting`: the posting record — one approved JD version, vacancy mode, ownership and
  assignment, and the posting-level metadata that scoping, filtering and reporting read.
  *(`TS-BL-037`)*
- `hiring/posting-lifecycle`: the registered state machine, the open-validation checklist, and the
  posting-opened event. *(`TS-BL-038`)*
- `hiring/posting-text`: the external posting artifact, separate from internal JD content by
  storage and by lifecycle, and its own generation target. *(`TS-BL-039`)*
- `hiring/posting-compliance`: the compliance-text status field, its configured default, and the
  gate that reads it at open. *(`TS-BL-040`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**No overlap to resolve.** The five Phase 1 features each carried a redistribution of inherited
`talentsphere-wave-1-foundation` requirements, and `ai-platform-governance` recorded itself as "the
last of the five to draw its inheritance across." That work is finished, and none of it reached
here: `talentsphere-wave-1-foundation`'s delta specs cover `access-control/`, `ai-platform/`,
`design-system/`, `identity/` and `platform/`, and nothing under `hiring/`.

## Impact

**New application code** — `backend/app/hiring/` containing the job-description aggregate and its
version model, the structured-input schema and its bound validator, the posting aggregate, the
posting state-machine definition registered against `platform-core`'s workflow service, the
open-validation checklist, and the two prompt-family payload builders. `backend/app/api/routes/hiring/`
for the eleven endpoints §13.2 enumerates under "Job Description and Posting". Frontend: the **JD
Workspace** and **Job Posting Detail** screens (§14.2), a Recruitment Manager review queue, and a
posting list — all assembled from `design-system` components, adding none.

**New tables** — `job_descriptions`, `job_description_versions`, `job_postings`, `job_posting_texts`,
and the JD review/comment records the `C-12` loop needs. All by reversible migration, applied by the
migration identity. `vacancy_slots` is **not** created here (see Explicitly not in this change).

**Existing capability consumed, not modified** — `platform-core`'s workflow engine (the first
registered state machine in the product), its async dispatch pattern (for `JOB-010`), its audited
runtime-configuration path (for the compliance-text default, posting aging thresholds, and the closed
practice vocabulary — §29 items 9 and 11, and `design.md` D13) and its notification engine (for the
RM review queue); `access-control-and-admin`'s
permission evaluator and its `assigned-postings` scope predicate; `ai-platform-governance`'s gateway,
prompt registry, run log and AI-disclosure marking; `design-system`'s forms, dense data table, page
templates and evidence/AI labels.

**Two prompt families move from stub to authored.** `TS-BL-030` registered all ten families with
stub text on purpose — *"writing ten real prompts against schemas Phase 2 will refine guarantees
rework."* Phase 2 is here for two of them. `job_description_generation` and `job_posting_generation`
each get real content, promoted as a **new template version** through the registry's own versioning
and promotion gate rather than by editing a version in place. A consequence worth stating plainly:
the promotion gate refuses to activate a template version with no passing corpus run recorded, so
neither family goes live in production until `TS-BL-032`'s harness carries cases for it. See
`design.md` D3.

**Downstream features that block on this one** — `candidate-intake`'s `TS-BL-047`,
`matching-and-ranking`'s `TS-BL-052`, and `decision-and-offers`' `TS-BL-070` name `TS-BL-037` or
`TS-BL-038` directly. Less visibly, `access-control-and-admin`'s `TS-BL-080` (ownership reassignment)
enumerates "assigned postings" as the first ownership relation it must hand to a successor, and
`insight-and-reporting`'s dashboards count postings by state — both read the columns this feature
creates.

**External constraints carried, not solved here** — **`OD-005` (legal sign-off on candidate-facing
and jurisdictional disclosure text) is open**, which is exactly why `G-09` decided the field and the
gate ship now with the default set to `not_required`. **`OD-003` (the AI model provider) is open**;
this feature calls the gateway and never a provider, so the decision costs it nothing. **The
per-field conciseness numbers this feature declares are provisional**, carrying the machine-readable
provisional provenance `ai-platform-governance` D7 requires — `S.5` states the numbers were never
given and should not be invented, and a wrong-but-enforced bound is fixable configuration while an
absent one is an unenforceable contract.
