# Matching and Ranking — Embedding, Retrieval and the AI Ranking Board

## Why

`hiring-postings` built what a candidate is measured *against*. `candidate-intake` built the
candidate and the resume. This feature is where those two meet and the product does the thing it
exists to do: **`RankingScore = f(resume_version, jd_version, prompt_template_version,
model_version)`**. All four terms of `domain-model.md`'s tuple exist for the first time only once
this feature ships — and until it does, nothing in TalentSphere orders one person against another.

**This is the first feature whose output is a judgement about a person's candidacy.** Everything AI
has produced so far described an artifact: a job description draft, a posting text, a set of skills
read off a resume. A fitment score is a statement about someone's suitability, rendered on a screen
next to eleven other people. That is why `project.md`'s one governing rule — *AI recommends, a human
decides* — has its highest-stakes application here, why `RANK-004` requires the board to say so on
its face, and why `G-01`, `G-03`, `G-06` and `G-07` were all adopted specifically to keep a score
from becoming a verdict.

**It is the third real AI Gateway consumer, and the first to send two references rather than one.**
`ai-platform-governance`'s `design.md` D3 fixed the envelope against JD generation and generalized
it to this feature by name: *"`candidate_ranking` sends posting and resume references."*
`hiring-postings`' and `candidate-intake`'s families each carried one reference; this one carries
`posting_ref` **and** `resume_ref`, and neither carries content. That was anticipated eleven days
before this conversation and is not renegotiable here — the payload inside the envelope is this
feature's, the envelope is not.

**It is also the first consumer of query-time authorization**, the design choice
`access-control-and-admin`'s D4 made specifically for this feature. `C-01` adopted `VEC-003`
(vector retrieval applies the same authorization rules as relational access) and `C-03` made read
scope matrix-configurable; [Interaction A](../talentsphere/exploration-notes.md#interaction-a--permission-changes-now-trigger-vector-re-indexing)
shows that together they would force a `VEC-005` re-index on **every permission edit** if a
resolved verdict were ever baked into a vector record. `access-control-and-admin` wrote the rule
— *"evaluate at query time, never bake a resolved verdict into stored data"* — and named
`TS-BL-050` as the consumer that has to honour it. This is that consumer.

**`C-01` is the decision that shapes the whole feature.** Vectors **retrieve**, the LLM **ranks**,
and the relational database stays authoritative (§12: *"the vector store shall never be the only
source of truth"*). `D07`'s volume assessment still stands — fewer than 5,000 candidates, fewer
than 30 postings — so the vector store is not here for scale; it is here because §24 makes it
mandatory for semantic resurfacing and §17 scopes what gets embedded. `VEC-003` and `VEC-005` were
recorded as **ongoing obligations, not one-time setup**, and this feature is where that bill starts
arriving.

**Nothing here is built, and nothing was inherited.** `talentsphere-wave-1-foundation` has no
`matching/` delta spec — its fourteen delta specs cover `access-control/`, `ai-platform/`,
`design-system/`, `identity/` and `platform/`. Its `design.md` D1–D18 are fully allocated across
the five Phase-1 features (`platform-core` D11 records the split: D1–D7 to
`access-control-and-admin`, D8 to `identity-and-access`, D9–D12 to `ai-platform-governance`,
D13–D14 and D17 to `design-system`, D15/D16/D18 to `platform-core`) and **none of them landed
here.** `sprint-0-outcome.md` records 13 tasks of platform-layer work with "no hiring feature
[existing] yet, by design," and `KNOWN_ISSUES.md` carries no entry about vectors, embeddings,
ranking or matching. Checked rather than assumed — see `design.md` D13.

## What Changes

Seven backlog items, `TS-BL-049` through `TS-BL-055`, from
[D.10](../talentsphere/exploration-notes.md#d10--phases-2-5-backlog-decomposition-and-the-feature-list-is-now-finalized-2026-08-25).
None is built.

- **`TS-BL-049` Vector embedding pipeline (resume → embedding).** `VEC-001`'s embedding of resume
  sections and candidate skill summaries, registered as §25's Embedding Generation job type against
  `platform-core`'s dispatch pattern, writing `vector_index_records` (§12.2) with `VEC-002`'s
  traceback to the authoritative relational record, a pinned `embedding_model`, and `VEC-004`'s
  filter columns. It embeds the `DOC-008` chunks `candidate-intake`'s `TS-BL-043` already produces
  and builds no second chunker. `VEC-005`'s re-index path ships **with** the pipeline rather than
  after it, on the same reasoning `ai-platform-governance` applied to write-before-invoke: a
  re-index path added later is a migration, not an invariant. §12.2's `source_type` enum also names
  `interview_note_section` and `scorecard_summary`; those sources arrive in Phase 3 and are
  registered by `interview-pipeline`, not embedded here — see `design.md` D2. Depends on
  `candidate-intake`'s `TS-BL-044`.
- **`TS-BL-050` Vertex AI Vector Search integration (`C-01`).** Retrieval **stage one**: kNN over
  the index with `VEC-004`'s filters and `VEC-003`'s authorization applied as
  `access-control-and-admin`'s query-time predicate filter, returning `VEC-008`'s source labels and
  `VEC-007`'s minimum required chunks. The backend is Vertex AI Vector Search, not `pgvector` —
  `C-01`'s original resolution named `pgvector` and was **superseded 2026-08-17** by `D09`'s infra
  planning, with everything else in that resolution unaffected. The `authorization_scope_json`
  column carries the **inputs** to an authorization decision (candidate id, posting id, practice),
  never its outcome, so a matrix change never triggers a re-index. Depends on `TS-BL-049`.
- **`TS-BL-051` RankingScore tuple model.** The persisted, versioned ranking record —
  `resume_version`, `jd_version`, `prompt_template_version`, `model_version` — and `G-07`/`RANK-005`'s
  rule that regenerating a ranking creates a **new version while preserving prior versions**. This
  is a table with its own lifecycle, not four columns on `candidate_posting_applications`;
  §12.2's `latest_rank` and `latest_match_score` become a pointer to the active version rather than
  the record itself (`design.md` D4). It is also where **ranking staleness** is represented, which
  is what lets `candidate-intake`'s deliberately-deferred questions be answered — see `design.md`
  D5. Depends on `TS-BL-050` and `ai-platform-governance`'s `TS-BL-030`.
- **`TS-BL-052` AI Ranking Engine (fitment/gap scoring).** Three registered prompt families move
  from stub to authored — `candidate_ranking`, `fitment_summary`, `gap_summary` (§16.3) — each
  invoked through the gateway in the fixed envelope carrying **`posting_ref` and `resume_ref`**
  (`ai-platform-governance` D3). §16.4's eleven permitted ranking signals and ten prohibited ones
  are enforced as a contract, not a guideline. `RANK-001`'s batch trigger, `ERR-007`'s
  candidate-level partial-failure reporting, and `JOB-010`'s `posting.opened` event — which
  `hiring-postings`' `TS-BL-038` already publishes with **no subscriber registered** — are all
  consumed here. `G-01`'s mandatory-criteria asymmetry lands in this item: the AI may propose
  `meets | unclear | manager_review_required` and may **never** assert `does_not_meet`
  (`design.md` D11). Depends on `hiring-postings`' `TS-BL-037`, `TS-BL-051`, and
  `ai-platform-governance`'s `TS-BL-027`.
- **`TS-BL-053` Ranking Board UI.** The AI Ranking Board (§14.2, §19.4) built from
  `design-system`'s dense data table — `RANK-002`'s full column set, `RANK-003`'s criteria
  explanations, `RANK-004`/`UI-006`'s human-review disclaimer, and `RANK-006`/`G-06`'s human
  override with a mandatory reason that never deletes the original AI output. A PM / RM / permitted-
  Recruiter surface per §8 and `C-10`, scoped by `C-03`'s assigned-postings predicate. It is also
  where `candidate-intake`'s `TS-BL-047` advisory duplicate warning renders — the one contract that
  feature deliberately kept to a payload this board consumes. Adds **no new design-system
  component**. Depends on `TS-BL-052`, `design-system`'s `TS-BL-009` and
  `access-control-and-admin`'s `TS-BL-018`.
- **`TS-BL-054` AI context visibility rules for Interviewers (`D05`/`C-10`).** Built against
  **`C-10`'s reversal, not `D05`'s original decision**: *"gaps yes, score no."* Interviewers see
  source-labelled fitment and gaps (`INT-004`, `INT-005`); they do **not** see
  `latest_match_score`, rank position, or other candidates. This item is a **server-side
  projection and its enforcement**, not a screen — the screen it feeds is `interview-pipeline`'s
  Interview Console, which is unproposed (`design.md` D7). It records disclosures through
  `ai-platform-governance`'s existing append-only disclosure record and **builds no second
  disclosure mechanism** (`design.md` D9). Depends on `TS-BL-053`.
- **`TS-BL-055` Insufficiency outputs (`G-03`).** `AI-008`'s insufficiency reasons rather than a
  guess, reusing `candidate-intake`'s already-established shape and vocabulary verbatim — *"missing
  evidence produces an insufficiency marker, not a value,"* with an insufficient field
  distinguishable from one never requested. Applied to ranking output, the load-bearing consequence
  is sharper than it was for enrichment: **absence of evidence must not be scored as weakness.** A
  thin resume produces a withheld or marked score, never a low one. `G-03` calls this "the failure
  mode most likely to erode trust in ranking"; `ai_insights.insight_type` (§12.2) already carries an
  `insufficiency` value. Depends on `TS-BL-052`.

**Explicitly not in this change:**

- **No shortlisting, no dispositions, no stages.** `TS-BL-056` (`interview-pipeline`) is the
  shortlisting decision UI and depends on `TS-BL-053`. This feature ranks and explains; it sets no
  `status`, no `shortlist_reason`, and no `final_outcome`.
- **No resurfacing and no priority lane.** `RANK-007` (existing matches appear before or alongside
  new submissions) and `RANK-008` (selected-not-offered priority lane) are
  `resurfacing-and-communications`' `TS-BL-075`/`TS-BL-076`, which depend on `TS-BL-052`. §25 lists
  Candidate Ranking and Existing Candidate Resurfacing as **two separate job types** on the same
  trigger; this feature registers the first and not the second, which is exactly what
  `hiring-postings` D7 anticipated when it published the event and registered no subscriber.
- **No interview questions, no interview summary, no scorecards.** `interview_questions`,
  `interview_note_summary` and `scorecard_generation` are three of the ten registered families and
  all three belong to `interview-pipeline`. `INT-004`'s AI-suggested questions render on the
  Interview Console; `TS-BL-054` supplies that console's *ranking-context projection* and nothing
  else.
- **No interview-note or scorecard embedding.** `VEC-001` scopes four source types and `C-01`'s
  scope note records interview-history semantic search as an intentional addition. Two of the four
  sources do not exist until Phase 3. `TS-BL-049` builds the pipeline as source-type-driven and
  registers the two resume-derived sources; `interview-pipeline` registers its own (`design.md`
  D2).
- **No new vector backend decision, and no `pgvector`.** `D09` settled Vertex AI Vector Search on
  2026-08-17. `AGENTS.md` names this explicitly as a case where "the exploration notes win" over
  the reference spec, which lists `pgvector` first.
- **No bias-audit tooling.** `project.md` records it as a separate initiative and `PRV-006`
  restricts collecting the data it would need. `RANK-004`/`UI-006` require the board to **avoid
  claiming** bias-free recommendations, which is a disclaimer requirement, not an audit capability.
- **No ranking criteria weight values.** §29 item 8 makes weights configurable *"where product
  approved"* — the mechanism ships, the numbers do not (`design.md` Open Questions).
- **No AI gateway, prompt registry, run log, disclosure record, evidence-label vocabulary,
  permission evaluator, dispatch substrate, audit writer, or design-system component.** All nine
  are consumed.

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability below is
new. None is preserved from `talentsphere-wave-1-foundation`: that change has no `matching/` delta
spec at all.

- `matching/resume-embedding`: `VEC-001`'s embedding of resume-derived sources as a registered job
  type, with `VEC-002`'s traceback, a pinned embedding model, and `VEC-005`'s re-index path.
  *(`TS-BL-049`)*
- `matching/vector-retrieval`: retrieval stage one — filtered kNN with query-time authorization,
  source labels, and minimum-chunk retrieval, over a store that is never the source of truth.
  *(`TS-BL-050`)*
- `matching/ranking-score`: the versioned reproducibility-tuple record, `G-07`'s preserved prior
  versions, the active-version pointer, and ranking staleness. *(`TS-BL-051`)*
- `matching/ranking-engine`: three prompt families invoked through the gateway on `posting_ref` +
  `resume_ref`, §16.4's permitted and prohibited signals, batch execution with candidate-level
  partial failure, and `G-01`'s proposed-not-asserted mandatory-criteria value. *(`TS-BL-052`)*
- `matching/ranking-board`: the PM/RM board — `RANK-002`'s columns, `RANK-003`'s explanations, the
  disclaimer, and the mandatory-reason human override. *(`TS-BL-053`)*
- `matching/ai-context-visibility`: `C-10`'s gaps-yes-score-no projection, `D05`'s still-standing
  per-candidate rule, and the disclosure records both leave behind. *(`TS-BL-054`)*
- `matching/ranking-insufficiency`: `G-03` applied to ranking — insufficiency markers instead of
  inferred values, and absence of evidence never scored as weakness. *(`TS-BL-055`)*

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**No overlap to resolve.** All five Phase-1 features have drawn their inheritance across from
`talentsphere-wave-1-foundation`, and `ai-platform-governance` recorded itself as the last to do
so with "no unaccounted content left." Nothing under `matching/` was ever written there. See
`design.md` D13 for the check.

**One correction was made to a shared document during this conversation**, per `AGENTS.md`'s
convention for genuine gaps found in the documents every feature reads: `exploration-notes.md`'s
`D05` correction block asserted that *"the decision (full context upfront) … stand[s] exactly as
written"* eleven days after `C-10` had reversed it. Fixed at the point of the error with a dated
note quoting the prior wording. See `design.md` D14.

## Impact

**New application code** — `backend/app/matching/` containing the embedding pipeline and its
re-index path, the retrieval client and its filter builder, the ranking record aggregate with its
version lifecycle, the three prompt payload builders, and the visibility projection.
`backend/app/api/routes/matching/` for §13.2's AI Ranking and Insights endpoints:
`POST /api/postings/{postingId}/ai/rank`, `GET /api/postings/{postingId}/ai/rankings`,
`GET /api/applications/{applicationId}/ai/insights`, and
`POST /api/applications/{applicationId}/ai/override`. Frontend: the **AI Ranking Board** (§14.2)
and a per-candidate insight detail view, both assembled from `design-system`'s dense data table,
AI-disclosure and evidence-source-label components, adding none.

**New tables** — `vector_index_records` (§12.2), a ranking-version table with its per-application
entries, and `ai_insights` rows (§12.2) for fitment, gap and insufficiency content. All by
reversible migration, applied by the migration identity. Two existing
`candidate_posting_applications` columns are written here for the first time — `latest_rank` and
`latest_match_score`, which `candidate-intake`'s `TS-BL-046` created and deliberately left unwritten
— plus `mandatory_criteria_status`, whose AI-proposed and human-confirmed halves are split by
`G-01`.

**New infrastructure and one new external dependency** — a Vertex AI Vector Search index and
endpoint by Terraform. The landing zone already enables `aiplatform.googleapis.com` per app, which
is the specific ground on which `D09` chose this backend over `pgvector`. It is the fourth external
service this product calls after Hubble, the model provider and the malware scanner, and
`G-12`/`NFR-004`'s degradation rule applies to it: retrieval unavailability degrades ranking, it
does not break the board, the candidate database, or any non-AI action.

**Three prompt families move from stub to authored.** `candidate_ranking`, `fitment_summary` and
`gap_summary` — three of the ten `TS-BL-030` registered with stub text on purpose — get real
content as **new template versions** through the registry's versioning and promotion gate. The gate
refuses production activation with no passing corpus run recorded, so all three are deployable to
Dev and not to production until `TS-BL-032`'s harness carries cases for them. `AI-015` and `C-09`
make those cases unusually load-bearing here: adverse examples, low-information resumes,
adversarial resumes and conflicting interview notes are named for **ranking templates
specifically**, and `ENG-008` requires prompt/model version review for any production change
affecting ranking behaviour.

**Existing capability consumed, not modified** — `ai-platform-governance`'s gateway (three families,
one envelope, no second egress), run log, **disclosure record**, override record, prompt registry,
evidence-label vocabulary and advisory-only guarantee; `access-control-and-admin`'s permission
evaluator, its **query-time filter contract** and its `assigned-postings` predicate, plus its
durable audit writer; `platform-core`'s dispatch pattern (two new job types — embedding generation
and candidate ranking), its audited runtime-configuration path (§29 items 6, 7, 8, 15) and its
correlation identifier across the async boundary; `hiring-postings`' pinned `jd_version` and its
`posting.opened` event; `candidate-intake`'s `resume_version_ref`, its `DOC-008` chunks and its
application record; `design-system`'s dense table, disclosure and label components.

**Two questions handed to this feature by name are answered here rather than deferred again.**
`candidate-intake`'s `design.md` D8 and its `resume-versioning` spec both record that *"whether a
re-point forces a re-rank is `matching-and-ranking`'s to decide under `G-07`"*, and its Risks
section records that *"`matching-and-ranking` ranking an unenriched candidate needs a defined
answer, and that is its feature's to give."* `design.md` D5 and D6 give both. A third —
`D23a`'s follow-on application-merge rule — is only **half** answerable here and is deliberately
left open, because the other input is `interview-pipeline`'s application stage (`design.md` D5).

**Downstream features that block on this one** — `interview-pipeline`'s `TS-BL-056` (on
`TS-BL-053`) and therefore its whole chain, and `resurfacing-and-communications`' `TS-BL-075` (on
`TS-BL-052`). `TS-BL-054`'s projection is consumed by `interview-pipeline`'s Interview Console; the
contract is deliberately a payload that console renders, mirroring how `candidate-intake` kept its
duplicate warning to a payload this feature's board renders.

**External constraints carried, not solved here** — **`OD-003` (the AI model provider) is open**;
this feature calls the gateway and never a provider, so it costs nothing beyond running on the stub
in Dev. **§29 item 8's ranking criteria weights are "where product approved"** and no product
approval exists; the configuration key ships with provisional provenance and no tuned values.
**`PRV-006`** keeps bias auditing out of scope while `RANK-004` still requires the disclaimer.
**`OD-004`'s retention period** governs how long a superseded ranking version is kept, which
`G-07` requires be preserved — noted, not resolved.
