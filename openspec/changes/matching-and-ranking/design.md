# Matching and Ranking — Design

## Context

See `proposal.md` — Why, for motivation, and the seven delta specs under `specs/` for the behaviour
contracts. This document covers only the technical decisions this feature must settle, the two
questions earlier features handed to it by name, and the reconciliation checks behind the
proposal's claim that nothing was inherited.

Constraints that shape the approach:

- **The envelope is already fixed, and this call was named in advance.**
  `ai-platform-governance` `design.md` D3 designed the gateway's uniform request envelope against
  JD generation and generalized it to this feature explicitly: *"`candidate_ranking` sends posting
  and resume references."* This is the third real consumer and the first to send two references.
- **`C-01` divides the work and the backend is settled.** Vectors retrieve, the LLM ranks, the
  relational database stays authoritative. The backend is **Vertex AI Vector Search**, resolved
  2026-08-17 under `D09`, superseding `C-01`'s original `pgvector` naming; everything else in that
  resolution is unaffected by the swap. `VEC-003` and `VEC-005` are recorded as **ongoing
  obligations**, not setup.
- **`C-10` reversed `D05`, and one paragraph of `exploration-notes.md` said otherwise.** `TS-BL-054`
  is built against `C-10`'s "gaps yes, score no." The contradiction was corrected in the shared
  document during this conversation — see D14.
- **Query-time authorization was designed for this feature by another one.**
  `access-control-and-admin` D4 wrote *"evaluate at query time, never bake a resolved verdict into
  stored data"* and named `TS-BL-050` as the consumer that has to honour it.
- **Three of this feature's outputs are governed by adopted reference-spec items rather than by
  free design:** `G-01` (mandatory criteria — AI proposes, human confirms any block), `G-03`
  (insufficiency), `G-06` (override records) and `G-07` (ranking versions). None is reopened here.
- **Two upstream seams exist and are unregistered on purpose.** `hiring-postings`' `TS-BL-038`
  publishes `posting.opened` with **no subscriber**; `candidate-intake`'s `TS-BL-047` produces a
  duplicate warning payload for a surface that did not exist. Both land here.
- **Settled stack** (`D09`): FastAPI on Python, PostgreSQL on Cloud SQL, Terraform on GCP with
  GitLab CI/CD, OpenTelemetry, background work on the landing zone's Pub/Sub → Eventarc → Workflows
  → Cloud Run Job chain, and Vertex AI Vector Search.
- **Only Local and Dev are provisionable**, and no generation provider is configured (`OD-003`), so
  every AI path in this feature runs on `ai-platform-governance`'s deterministic stub adapter until
  that decision lands.

## Goals / Non-Goals

**Goals:**

- A score that can always be explained: the full tuple on every entry, prior versions preserved, and
  an explicit representation of an entry whose inputs have moved on.
- Authorization that costs nothing at index time and is always current at query time — the specific
  property [Interaction A](../talentsphere/exploration-notes.md#interaction-a--permission-changes-now-trigger-vector-re-indexing)
  identified as the thing to get right here.
- Guarantees checkable by absence: no path from AI output to a blocking value, no score in the
  projected response, no second disclosure store, no comparative claim in a per-candidate payload,
  no negative score contribution from an insufficiency marker.
- Answers to the two questions `candidate-intake` handed forward, and an honest statement of the
  third it could only half-answer.
- A ranking surface that is a reading surface: it explains, records divergence, and decides nothing.

**Non-Goals (design-level boundaries beyond the proposal's scope):**

- **No score model tuning.** §29 item 8's weights are configuration with no approved values; a
  tuned weighting needs real postings, real resumes and a real provider, and has none of the three.
- **No relevance evaluation of retrieval.** Whether kNN returns *good* chunks is a quality question
  needing a corpus of real resumes. What ships is that retrieval is filtered, authorized, bounded
  and labelled — properties assertable without a corpus.
- **No embedding model selection tuning.** The model is pinned, recorded per record and changeable
  through the re-index path. Choosing which one is the same shape of open decision as `OD-003`.
- **No second dispatch substrate, gateway, registry, evaluator, audit writer, disclosure store or
  design-system component.** All are consumed.
- **No interview surface.** `TS-BL-054` produces a payload; `interview-pipeline` renders it.

## Decisions

### D1 — Two references, three families, one registered job type

Three separate calls answer three §16.1 rows, and exactly one job type is registered for the lot.
All three parts of that sentence are easy to get wrong in opposite directions, so each is stated.

**Two references, no content.** `ai-platform-governance` D3 fixed the envelope and named this call:
*"`candidate_ranking` sends posting and resume references."* Concretely:

```
request:
  family:        candidate_ranking | fitment_summary | gap_summary
  caller:        { actor, page: AI Ranking Board, action: Run AI, correlation_id }
  input:         { posting_ref: <id>, resume_ref: <id@version> }
response:
  run_ref, status, output | failure: { kind: validation | provider | configuration, detail }
```

The posting reference resolves to its pinned `jd_version` plus the posting's own authoritative
requirement fields; the resume reference resolves to that version's `DOC-008` sections, which
`candidate-intake` D10 already guaranteed carry no structured contact field. *Why references rather
than a composed prompt payload:* the run log's reference-only rule becomes a property of the
architecture rather than a redaction step, because the gateway never held the content — the same
argument D3 made, now with two references instead of one.

**Three families, three runs.** §16.1 lists Candidate Ranking, Fitment Summary and Gap Summary as
three capabilities with different triggers and outputs; §16.3 registers `candidate_ranking`,
`fitment_summary` and `gap_summary` with three different required output formats. Collapsing them
into one call would collapse three registry entries into one, and per-family model configuration,
per-family conciseness bounds and per-family corpus coverage all attach to a *family*. The registry
is fixed at ten and is not this feature's to renegotiate.

*Alternative considered:* one composite ranking call returning score, fitment and gaps together, on
the grounds that it is one conceptual operation and cheaper in provider round-trips. Rejected — it
would make `ENG-008`'s "prompt/model version review for production changes affecting ranking
behavior" apply to a single artifact covering three behaviours, so a wording change to the gap
prompt would force a re-review of the scoring template. Three families keep the review surface as
granular as the change surface. Cost accepted: three runs per candidate per ranking version, which
`D07`'s volumes make affordable.

**One registered job type, and it is the orchestrator.** This is the specific place
`candidate-intake`'s `tasks.md` warned that "getting it wrong in either direction is the most likely
error." Both halves are true simultaneously:

- **No job type per gateway call.** `ai-platform-governance` D10 — *"AI runs register as a job type;
  they do not get a queue"* — means the gateway already dispatches each invocation asynchronously.
  Registering one here would be the second dispatch path `candidate-intake` D2 exists to prevent.
- **One job type for the batch.** §25 lists **Candidate Ranking** as a background job type triggered
  by posting open, and `hiring-postings` D7 publishes `posting.opened` and *registers no subscriber*.
  Something must subscribe. That subscriber — the orchestrator that fans out per-candidate gateway
  calls and assembles a ranking version — is this feature's one registered job type.

### D2 — Two of the four `VEC-001` source types are embedded here; the other two are registered by their owner

§12.2's `source_type` enum names four values: `resume_section`, `candidate_skill_summary`,
`interview_note_section`, `scorecard_summary`. `TS-BL-049` embeds the first two and builds the
pipeline **source-type-driven** so the other two are registrations rather than modifications.

*Why not embed all four now:* two of them do not exist. Interview notes arrive with
`interview-pipeline`'s `TS-BL-058` (whose `INT-007` already lists embeddings among a note's stored
forms) and scorecard summaries with `TS-BL-062`. Embedding a source type before its records exist
would be a code path with no input, which is `platform-core` D5's silent-success class — it passes
by never running.

*Why not defer the pipeline until all four exist:* `C-01` places embedding in the intake/matching
stage explicitly, `TS-BL-049`'s dependency is `candidate-intake`'s `TS-BL-044`, and ranking cannot
retrieve from an index that does not exist. The pipeline is needed now; two of its inputs are not.

*What "source-type-driven" has to mean to be worth claiming:* the registration surface is a real
seam, not a switch statement to extend — a source type declares its own record-producing shape and
its filter attributes, and the enum carries all four values from the first migration so Phase 3 adds
a registration and no schema change. This is the same shape `platform-core` used for job-type
registration and `ai-platform-governance` used for its envelope: build the seam against the real
consumer, do not build the consumer.

*Recorded consequence:* `C-01`'s scope note flagged interview-history semantic search as "a genuine
capability that is **not in the original 19 features**." That capability becomes reachable once
`interview-pipeline` registers its two source types. Nothing here builds it, and nothing here blocks
it.

### D3 — Authorization at query time, and the three `VEC-005` triggers that remain

`VEC-005` names four re-index triggers: parsing logic, embedding model, source content, and
**authorization metadata**. This design implements three and designs the fourth out of existence.

`authorization_scope_json` carries the *inputs* to an authorization decision — candidate id, posting
id, practice — and never a resolved verdict. Retrieval then evaluates through
`access-control-and-admin`'s evaluator and applies the returned filter as a query-time predicate on
the index.

*Why this is the whole point rather than an optimization:* `C-01` adopted `VEC-003` and `C-03` made
read scope matrix-configurable. Interaction A traces the consequence — a permission edit would
invalidate every baked verdict, forcing a full re-index on every matrix change. With ~5,000
candidates that is survivable and still wrong: `AUTHZ`-level evaluation is **uncached specifically so
revocation bites on the next request**, and a re-index queue puts a window back in. A revocation that
takes effect when a batch job finishes is not a revocation.

*The trade-off, stated because no type system enforces it:* every retrieval path must apply the
returned filter. `access-control-and-admin` D4 mitigated the same exposure by routing list endpoints
through a helper that takes the filter as a required argument; retrieval does the same, and the spec
carries a scenario that fails the suite on a retrieval path without one.

*Alternative considered:* bake scope and accept re-indexing, on the grounds that filtered kNN on a
managed index is simpler than post-hoc predicate application. Rejected on the revocation-window
argument above, and because `access-control-and-admin` already wrote the rule as a requirement with
this feature named as its consumer — diverging here would leave a specified requirement with no
implementation while appearing to satisfy it.

### D4 — The tuple is a versioned record; the application's two columns are a pointer

`RankingScore` is a table with its own version lifecycle, not four columns added to
`candidate_posting_applications`. §12.2's `latest_rank` and `latest_match_score` become a derived
pointer at the active version.

*Why:* `G-07`/`RANK-005` require that regenerating a ranking **preserve prior versions**. Columns on
the application can hold exactly one value, so satisfying `G-07` with them means either overwriting
(losing the prior board) or shadow-copying rows somewhere else (a version table by another name,
with no constraints on it). §12.2's own annotation already says what `latest_rank` is — *"active
ranking version output"* — which presupposes versions living elsewhere.

*Why keep the two columns at all:* they are declared in §12.2, `candidate-intake` created them, and
a list of applications wants a score without joining a version table per row. They are a
denormalization with a single writer — activation of a ranking version — and the spec carries a
scenario that fails the suite on any other write path.

*What this makes cheap later:* `insight-and-reporting`'s AI-agreement-rate metric compares human
decisions against AI output over time, which needs the version that was *current when the human
decided*, not the current one. That query is answerable here and unanswerable from two columns — and
it takes **both halves of this decision**. The version table establishes **which boards exist**, and,
because no entry is ever updated in place, that each still reads as it did. It does **not** establish
which board was live at a given moment: no version record carries an activation timestamp. What
supplies *active at time T* is the **audited activation event**, which task 3.13 already mandates for
every ranking activation. `insight-and-reporting`'s `design.md` D7 has the full join.

### D5 — What triggers a re-rank, what only marks staleness, and the one question that stays open

§25 gives three ranking triggers — *posting open, manual trigger, requirement change*. Nothing else
re-ranks automatically. The full table, including the cases earlier features deferred to here:

| Event | Effect |
|---|---|
| Posting opens (`JOB-010`, `posting.opened`) | New ranking version, automatically |
| Authorized user triggers ranking (`RANK-001`) | New ranking version |
| A ranking-relevant posting requirement field changes (§25 "requirement change") | New ranking version |
| Application re-pointed to a different resume version | Entry marked **stale**; no re-rank |
| Candidate merge leaves an application on a superseded input | Entry marked **stale**; no re-rank |
| A new resume version uploaded, application still pinned | Nothing — the application is pinned |
| Permission matrix change | Nothing (D3) |

*Why "requirement change" resolves to the posting's own fields:* `hiring-postings`' pinning
requirement makes an open posting's `jd_version` immutable, and approving a later JD version
explicitly does not change what an open posting references. So the only requirement inputs that can
move under a live posting are the posting's own authoritative fields — which `hiring-postings` D2
establishes it owns copies of, and which §16.4 names as ranking signals ("work location and work mode
constraints where configured").

**This answers `candidate-intake`'s deferred question.** Its `candidate/resume-versioning` spec and
`design.md` D8 both record that *"whether a re-point forces a re-rank is `matching-and-ranking`'s to
decide under `G-07`."* The answer is **no**, for three reasons: §25 does not list a resume change
among the triggers; `RANK-001` makes ranking a user-triggered action, so silently re-ranking on a
recruiter's file-management action produces a board nobody asked for; and `G-07` already provides
the mechanism that makes "no" safe — the old entry is preserved, marked stale, and visibly attributed
to the version it was computed against. A human sees the staleness and triggers the re-rank.

*Alternative considered:* auto-re-rank on re-point, so the board is never stale. Rejected — it makes
a resume re-point (`candidate-intake` `TS-BL-048` task 8.9: permission-gated, mandatory reason, *"no
state transition and no ranking recomputation"*) into an action with a large invisible side effect,
and it would make an authorized re-point a way to change a candidate's score without triggering
ranking.

**What stays open, and why it is not this feature's to close.** `D23a`'s follow-on rule — when two
applications on one posting merge, *which ranking is retained or whether a re-rank is forced* —
is now **half** answerable: the ranking half is "nothing is retained or discarded; both entries are
preserved and the surviving application's entry is marked stale." The other half is *which
application survives*, and that depends on the application stage, which is `interview-pipeline`'s.
`candidate-intake` D1(b) declined to invent it for exactly this reason and asserted the invariant
instead. That invariant is unchanged and is now satisfiable from this side too: staleness gives the
merge a defined ranking outcome that pre-empts nothing.

### D6 — An unenriched candidate is ranked on the evidence that exists

`candidate-intake`'s Risks section recorded that *"`matching-and-ranking` ranking an unenriched
candidate needs a defined answer, and that is its feature's to give."* The answer: **rank them,
using what exists, and record insufficiency for what enrichment would have supplied.** Never exclude,
never score low for the absence.

*Why not exclude:* `D18` made the decoupling deliberately — deterministic extraction failing blocks
the candidate, LLM enrichment failing does not — so an unenriched candidate is a valid expected
state, not an error. Excluding them from the ranking version would make an AI subsystem's failure
into a candidate's disadvantage, which is the closest thing to an AI-made rejection this architecture
could produce without a transition. `AI-010` and `BR-007` both bite.

*Why not score low:* absence of evidence and evidence of absence are different findings, and a single
number cannot carry the difference. This is `G-03`'s stated purpose — insufficiency "attacks the
failure mode most likely to erode trust in ranking" — and `G-01`'s reasoning in miniature, where a
single score conflating "excellent fit but lacks the mandatory clearance" with "mediocre fit, meets
everything" was the argument for a separate field.

*What it costs:* the board shows entries whose score is withheld or qualified, so a Practice Manager
cannot always sort a full column of numbers. That is the correct cost — the alternative sorts a
column where some numbers mean "assessed as weak" and others mean "we could not tell," which is a
worse board that looks like a better one.

### D7 — `TS-BL-054` is a server-side projection, not a screen, and its rule comes from `C-10`

**Built against `C-10`, not `D05`.** `C-10` amends D05 *reversing it*: gaps yes, score no. That is
the requirement. See D14 for the shared-document paragraph that said otherwise and what was done
about it.

**It is a projection because its screen belongs to another feature.** The surface an interviewing
actor actually reads is the Interview Console — `interview-pipeline`'s `TS-BL-058`, unproposed. So
this item builds the *server-side response* and its enforcement, and the console consumes it. This
mirrors `candidate-intake`'s handling of its duplicate warning, kept deliberately to "a warning
payload the list renders, so `TS-BL-053` consumes rather than accommodates it" — the same seam, in
the opposite direction.

*Why the dependency on `TS-BL-053` is real rather than incidental:* a projection is defined by what
it withholds, and what it withholds is defined against the full board payload. `D.10`'s own
dependency-translation note reaches the same conclusion from the other side, recording that
`TS-BL-053` needs `TS-BL-018` because *"`C-10`'s 'gaps yes, score no' display rule for Interviewers
is a permission-gated rendering decision."* Writing the projection first would mean writing it
against a payload shape that does not exist.

**The projection is keyed to the permission verdict, not to a role name.** An actor who lacks the
board action receives the projected response; one who holds it receives the full one. `C-10`'s
allocation is then a **seeded matrix value**, which is what `Interaction B` says it is, rather than a
role comparison compiled into the code. This also makes `INT-003`'s own escape hatch —
"unless broader permission is granted" — work without a code change, and makes `AUTHZ-005`'s direct
denial effective against a user who holds the board action by role.

*Alternative considered:* hard-code the Interviewer and Hiring Panel Member roles as the projected
audience. Rejected — nine roles exist and the matrix is configurable by design (`C-02`, `ADM-005`);
a hard-coded pair would need editing every time the matrix answered a question the code had already
answered, and it would silently ignore an override.

### D8 — Suppressing the score is not enough, and the rule reaches the board's own payload

`D05`'s per-candidate rule survives `C-10` untouched, and it does two things a score suppression
alone does not.

**1. It constrains claim *content*, not just fields.** A gap summary reading "weaker on cloud
experience than most applicants for this posting" leaks the cohort's standing in prose while every
withheld field stays withheld. So the projection rejects comparative claims, and the three families'
output contracts are absolute-by-construction: §16.1's Fitment Summary and Gap Summary input columns
name resume and interview evidence and **no cohort**, so a comparative claim is also a claim with no
evidence reference — already a contract violation under `AI-007` and `G-02`. The rule makes the
enforcement explicit rather than emergent.

**2. It does constrain the board — at the per-candidate endpoint.** The question is worth stating
because the intuitive answer is no. `RANK-002` requires the board to display rank, `§8` gives the
board to Practice Managers and Recruitment Managers, and comparing candidates is their job; `D05`'s
rule is scoped to "AI context shown to an Interviewer." So the board keeps rank.

What it constrains is §13.2's *other* endpoint. There are two:
`GET /api/postings/{postingId}/ai/rankings` (posting-scoped, comparative, board audience) and
`GET /api/applications/{applicationId}/ai/insights` (application-scoped). The second is the natural
thing for a role-restricted surface to reuse, and if comparative data rides along on it then the
projection is one careless reuse away from leaking. So the split is drawn at the payload: **rank,
score and cohort size exist only on the posting-scoped response.** A candidate-detail view opened
from the board reads the application-scoped payload plus the board row it came from, rather than a
richer per-candidate payload that later has to be trimmed.

*Why this is worth a decision rather than an implementation detail:* the alternative — one rich
per-candidate endpoint filtered by role at read time — is exactly the display-filter shape
`config.yaml` rules out for redaction, and it puts the guarantee in the filter rather than in the
data. Cohort size is also derivable from surprisingly little: a per-candidate payload carrying a
percentile, a "top quartile" flag, or a normalized score all reconstruct standing without naming a
rank.

### D9 — Disclosure records are consumed, and what counts as a disclosure is decided here

`ai-platform-governance`'s `ai-platform/ai-run-logging` capability already carries an append-only
disclosure record built for exactly this question. This feature writes to it and **builds no second
mechanism**. Its `design.md` D11 states the ownership split directly: *"`matching-and-ranking`'s
`TS-BL-054` owns the visibility rules; the record they must leave is substrate."*

What the substrate does not decide, and this feature must: **what counts as one disclosure.** The
rule adopted is *one record per retrieval of AI output by an actor*, not one per page render and not
one per session.

*Why not per render:* a board that re-renders on sort, filter and pagination would write dozens of
records for one sitting, and the question the record exists to answer — "was this seen before the
judgement was formed" — is answered identically by the first of them. Volume that adds no answer
degrades the record's usability as evidence, which is its only purpose.

*Why not per session:* a session that opened the board and never opened a candidate discloses
nothing about that candidate. `D05`'s question is per-output, so the record has to be too.

*What the record must carry to be usable, beyond the substrate's four fields:* the context value has
to distinguish **pre-interview** disclosure from any other, because that is the distinction `D05`
asked about and `C-10`'s agreement-rate consequence depends on. The interviewing-actor projection and
the board are therefore distinct contexts on the record rather than one "viewed" context.

*One assertion this buys, worth stating as a requirement rather than a hope:* the disclosure history
can be enumerated to confirm that no score was ever disclosed to an interviewing actor
pre-interview. `C-10` claims the agreement-rate metric "becomes meaningful"; this is what makes that
claim checkable from stored data rather than from reading the projection code.

### D10 — Insufficiency reuses `candidate-intake`'s vocabulary verbatim, and adds one thing

`candidate-intake`'s `candidate/resume-enrichment` capability already established `G-03`'s shape
concretely: a requirement titled *"Missing evidence produces an insufficiency marker, not a value,"*
with the scenario that *"a field marked insufficient is distinguishable from a field that was never
requested."* `TS-BL-055` is the same `G-03` concept applied to ranking output, and reuses that
wording, that shape and that vocabulary rather than inventing parallel terms.

*Why sameness matters more than local fit here:* a resume produces an insufficiency at intake and
the same resume produces an insufficiency at ranking, and both render on surfaces a Practice Manager
reads. Two vocabularies for one concept would make "insufficient" mean subtly different things on
two screens describing the same candidate, and `evidence-labeling`'s closed-vocabulary argument
applies by analogy — the value of a closed enumeration is that exactly one component defines it.

**What ranking adds that enrichment did not need: a third state.** Enrichment needed *insufficient*
distinguishable from *never requested*. Ranking needs both distinguishable from **evaluated as
weak**, because ranking produces a score and the other two do not. That third distinction is where
`G-03`'s trust argument actually lands, and it is why `matching/ranking-insufficiency` asserts that
an insufficiency marker makes no negative contribution to a score. Without it, "we found no
evidence" and "we found weak evidence" collapse into the same number, and `G-03` is satisfied on the
display and defeated in the arithmetic.

### D11 — `G-01`'s mandatory-criteria asymmetry splits across two items, and both halves are asserted

`mandatory_criteria_status` (§12.2) is on `candidate_posting_applications`; `candidate-intake`
created the column and left it unwritten. `D.10` gives it no item of its own, so it lands inside this
feature's two natural halves, recorded here under `D.11`'s permission to refine internal item
boundaries:

- **`TS-BL-052` proposes.** A ranking run may output `meets | unclear | manager_review_required` and
  the contract **rejects** `does_not_meet` as a violation. `G-01`: *"only a human may set a blocking
  value, with a reason, audited."*
- **`TS-BL-053` confirms.** The blocking value is set by a human on the board, with a mandatory
  reason, audited, with the AI-proposed value still visible beside it.

*Why not one item:* the two halves have different dependencies (a contract clause versus a permission
-gated audited UI action) and different failure modes. More practically, they are separately
deployable in the order they are needed: the proposal half is useless without the engine, and the
confirmation half is useless without a surface.

*Why the prohibition is a contract rejection rather than a UI omission:* `G-01`'s stated reason is
that otherwise "the AI would be gating a decision about a person." A value the AI can emit and the UI
declines to show is still a value in the database that a later feature could read — and `G-01` itself
names the concrete downstream reader: `BR-015`'s priority lane grants precedence *subject to mandatory
criteria*, so an AI-asserted block would silently disqualify a candidate from resurfacing. Rejecting
at the contract means the value never exists.

*The related boundary:* `matching/ranking-insufficiency` carries the specific case where absent
evidence tempts a model toward `does_not_meet`, because absent and disqualifying evidence look alike
to a model and this is where `G-01` and `G-03` meet.

### D12 — Item-boundary refinements, recorded rather than assumed

Under `D.11`'s permission for a feature's propose conversation to refine its internal item
boundaries. None changes a dependency edge and none moves work between features.

- **`TS-BL-049` carries `VEC-005`'s re-index path**, which `D.10`'s one-line title ("resume →
  embedding") does not mention. It ships with the pipeline for the reason
  `ai-platform-governance` gave for write-before-invoke ordering: a re-index path added later is a
  migration against existing records, not an invariant. `C-01` also records `VEC-005` as an
  **ongoing** obligation, which is a property of the pipeline rather than a follow-on item.
- **`TS-BL-052` authors three template versions, not one.** D1. `D.10`'s title says "fitment/gap
  scoring," which names all three outputs while reading like one job.
- **`TS-BL-052` and `TS-BL-053` split `G-01`.** D11.
- **`TS-BL-051` carries ranking staleness** as well as the tuple and its versions. Staleness is a
  property of a version's relationship to current inputs, so it has nowhere else to live, and it is
  what makes D5's answer to `candidate-intake` representable.
- **`TS-BL-053` renders `candidate-intake`'s duplicate warning.** That feature's `TS-BL-047` task
  7.10 already assigned this and kept the contract to a payload; recorded here so the board's scope
  includes it rather than discovering it during apply.

### D13 — What Sprint 0 built and decided about matching: nothing, and no decision was allocated here

The same reconciliation check the other seven proposed features ran, and the same result
`hiring-postings` and `candidate-intake` reported. Checked, not assumed.

**Sprint 0 built nothing relevant.** `sprint-0-outcome.md` records 13 of 196 Wave-1 tasks, all
platform-layer, with "no hiring feature exist[ing] yet, by design." The backend's directories are
`core/`, `api/`, `db/`, `audit/`, `features/`, `runtime_config/`, `middleware/` and `migrations/` —
there is no matching module, no `vector_index_records` table, no ranking table, and no Vertex AI
resource in Terraform.

**No `D1`–`D18` decision was allocated here.** `platform-core` D11 records the full split — D1–D7 to
`access-control-and-admin`, D8 to `identity-and-access`, D9–D12 to `ai-platform-governance`, D13–D14
and D17 to `design-system`, D15/D16/D18 to `platform-core` — which accounts for all eighteen with
none left over. Consistent with `ai-platform-governance`'s statement that all five Phase-1 features
have drawn their inheritance across and that change "has no unaccounted content left."

**No `matching/` delta spec exists** in `talentsphere-wave-1-foundation`. Its fourteen delta specs
cover `access-control/`, `ai-platform/`, `design-system/`, `identity/` and `platform/`. So unlike the
Phase-1 features, there is nothing to redistribute and no interim double-description to accept.

**`KNOWN_ISSUES.md` carries no entry about vectors, embeddings, ranking or matching.** Recorded
because an empty section can otherwise read as a clean bill of health rather than as an absence of
subject matter. **One** entry there does bear on this feature without being about it: only Local and
Dev are provisioned, which means every retrieval path here is exercised against a Dev-scale index.

**A second constraint belongs beside it but is not in that file, and this design previously said it
was.** All three of this feature's prompt families run against the deterministic stub because the
model provider is undecided — which is `reference/spec.md` §35's **`OD-003`**, reasoned in
`ai-platform-governance`'s `design.md` D1, **not** a `KNOWN_ISSUES.md` entry: that file carries no
mention of a provider, `OD-003`, or a stub. The entry is owed but **unwritten** — it is
`ai-platform-governance`'s task 1.18, *"Record the unresolved provider decision (`OD-003`) in
`KNOWN_ISSUES.md`, in the same form the unconfirmed Hubble contract (`OD-001`) already takes,"* which
lands with `TS-BL-027` and is not built. Found during `interview-pipeline`'s verification, which hit
the same misattribution in its own D13 and fixed it there; the constraint is unchanged and still
applies, only its citation was wrong.

**Two carry-forward items from `sprint-0-outcome.md` touch this feature indirectly.** The
`INSERT`/`SELECT`-only grant pattern (proven on Cloud SQL by `access-control-and-admin`'s
`TS-BL-020`) is what makes the disclosure records this feature writes trustworthy; and the
hand-granted schema ownership means standing up the Vertex AI index in a re-provisioned Dev inherits
the same manual step every other resource does.

### D14 — A shared-document gap found and fixed: `D05`'s correction block re-asserted a reversed decision

**Corrected in `exploration-notes.md` during this conversation**, per `AGENTS.md`'s convention for
genuine gaps in the four documents every feature reads, and the precedent of `D09`'s background-jobs
row, `S.6`, and `domain-model.md`'s mis-cited AI-governance reasoning.

**What was wrong.** `D05`'s ✔ CORRECTED block, added 2026-08-25 by
`ai-platform-governance`'s propose conversation, closes by listing what its fix leaves untouched:
*"**The rest of D05 is unaffected**: the decision (full context upfront), the accepted anchoring
trade-off, the structured-note mitigation, and the per-candidate/no-rank rule all stand exactly as
written."*

**The decision does not stand.** `C-10` reversed it on 2026-08-14 — eleven days earlier — in a
resolution whose first line reads *"Amends D05, reversing it"* and whose outcome is gaps yes, score
no. A reader arriving at D05 through that block rather than through the Decision index reads a
reversed decision as current.

**Why this mattered enough to fix rather than route around.** `TS-BL-054` is the item that has to
build one of the two readings, so this feature could not proceed without picking. Picking silently
would leave the next reader — `interview-pipeline`, which renders this projection — to pick again
from the same contradiction.

**What the fix says**, in `exploration-notes.md` at the point of the error: the prior wording quoted
in full, then D05's five elements resolved individually — the decision reversed, the anchoring
trade-off moot for Interviewers and still live for the board's own audience, the logging mitigation
doubly superseded, and the structured-note and per-candidate rules standing. It also records that the
**disclosure record is unaffected**: its reasoning was motivated by D05's mitigation but does not
depend on it, because disclosure records answer "who saw this AI output, when" for any output and any
actor — the board's score shown to a Practice Manager and the fitment and gaps shown to an
Interviewer under `C-10` alike. Only the motivating example is stale.

**One artifact outside this change carries the stale premise and was flagged, not edited.**
`ai-platform-governance`'s `ai-platform/ai-run-logging` spec sources its disclosure-record
requirement to *"`D05`, whose accepted trade-off — full AI context shown to an Interviewer before the
interview."* The requirement itself is correct and this feature consumes it unchanged; only its
citation describes a reversed decision. `AGENTS.md` is explicit that correcting another feature's
artifacts is `/opsx:update`'s job, one change at a time, with user confirmation — so it is recorded
here and in the `exploration-notes.md` block, and not hand-edited.

## Risks / Trade-offs

- **[The first AI output that ranks people is built with no real provider and no tuned weights.]**
  Every score produced in Dev comes from `ai-platform-governance`'s deterministic stub, and §29 item
  8's weights ship provisional and unapproved → Mitigated by scoping what is actually being asserted:
  the contracts, the provenance, the prohibitions and the insufficiency behaviour are all testable
  against a stub, and `C-09`/`AI-015`'s corpus is what tests output quality once a provider exists.
  Residual risk accepted, and it is larger here than for the two earlier AI consumers — a JD draft
  read by its author is self-correcting; a score read by a Practice Manager is not.
- **[Three families multiply the corpus obligation, and `AI-015` names four adversarial input classes
  for ranking specifically.]** Adverse examples, low-information resumes, adversarial resumes and
  conflicting interview notes, across three families → Mitigated by the promotion gate refusing
  production activation without passing cases, so the obligation cannot be quietly skipped. Not
  mitigated: Dev runs all three before those cases exist, which `ai-platform-governance` D8 argues is
  the correct order and which is still a window where a bad prompt is reachable in Dev.
- **[The fourth `VEC-005` trigger is designed out, so nothing exercises the "authorization changed"
  path — and if a resolved verdict is ever stored later, nothing will notice.]** → Mitigated by an
  assertion rather than a comment: the spec requires that stored authorization metadata contain no
  resolved outcome and that a matrix change require no re-index, both checkable. This is the exact
  failure `access-control-and-admin` D4 said was "cheap to avoid here and expensive there."
- **[Retrieval quality is unmeasured, and a filtered kNN that returns the wrong chunks produces a
  confident ranking on poor evidence.]** → Partly mitigated: evidence references make the inputs to
  every claim visible on the board, so a Practice Manager can see *what* the score rested on even
  when nobody has measured whether it rested on the best available. Not mitigated: relevance itself
  needs a corpus of real resumes and postings, which does not exist. Named as a Non-Goal rather than
  left implicit.
- **[Withheld and qualified scores make the board's score column non-uniform.]** D6's consequence: a
  column where some cells are numbers and some are "insufficient evidence" is harder to sort and
  scan → Accepted deliberately as the lesser harm; the alternative is a uniform column where some
  numbers are assessments and others are artifacts of missing data. Mitigated by the insufficiency
  rendering on the row rather than behind a detail view, so the non-uniformity is legible instead of
  surprising.
- **[The projection's guarantee is only as strong as the number of paths that respect it.]**
  `TS-BL-054` withholds score and rank from one response shape; `interview-pipeline` will build the
  surface, and a future feature could add a third path to per-candidate insight data → Mitigated
  structurally by D8's payload split (comparative data exists only on the posting-scoped response, so
  a new per-candidate consumer cannot leak what it never receives) and by the disclosure-history
  assertion, which fails if a score is ever disclosed to a projected-view actor.
- **[Staleness is a state that can be ignored indefinitely.]** D5 chooses marking over re-ranking, so
  a board can sit stale for a long time and still look authoritative → Mitigated by the staleness
  rendering on the row with the version it was computed against, which is strictly more information
  than the alternative gave. Not mitigated: nothing forces a re-rank, by design. If stale boards turn
  out to be common in practice, an aging threshold is the natural answer and §29 item 9 already
  provides the configuration shape for one.
- **[Two questions were answered for another feature's benefit, against a score model that does not
  exist yet.]** D5 and D6 both commit this feature on behalf of `candidate-intake`'s deferred
  decisions → Mitigated by both answers being *refusals to act* (no auto-re-rank, no exclusion, no
  penalty) rather than mechanisms that could be wrong in detail. A refusal is cheap to revise; a
  built-in behaviour is not.
- **[Vertex AI Vector Search is a fourth external dependency and the first one with its own
  Terraform-provisioned index.]** → Mitigated by `G-12`/`NFR-004` degradation being a spec
  requirement with its own scenario, including the specific case that a retrieval failure must fail
  the ranking run rather than produce a ranking from an empty evidence set. Residual: standing up the
  index in a re-provisioned Dev inherits Sprint 0's hand-granted-ownership carry-forward.
- **[This feature writes two columns another feature created, which is a shared-ownership seam.]**
  `latest_rank` and `latest_match_score` on `candidate_posting_applications` → Mitigated by the
  single-writer rule in D4 with a test that fails on any other write path, and by
  `candidate-intake`'s own task 6.12 having named this feature as their writer in advance, so the
  seam was agreed rather than assumed.

## Migration Plan

There is no data migration; nothing here has a predecessor in production. Sequence:

1. **`TS-BL-049` first, and it is genuinely blocked.** Its dependency is `candidate-intake`'s
   `TS-BL-044`, which is not started, and it needs `DOC-008` sections to embed. Unlike
   `ai-platform-governance`'s and `candidate-intake`'s entry items, this feature's first item cannot
   start today. The pipeline's registration seam and the re-index path can be built against
   `TS-BL-043`'s declared section shape where staffing forces an early start.
2. **Provision the Vertex AI index by Terraform before `TS-BL-050`**, in Dev only. The landing zone
   already enables `aiplatform.googleapis.com` per app, which is the ground `D09` chose this backend
   on. UAT and Prod get validated, never-applied configuration, per `D15` as amended.
3. **`TS-BL-050` next, and build the filter helper before the first query path.** D3's trade-off is
   that nothing enforces filter application; the helper that takes the filter as a required argument
   is the first task, so every retrieval path after it is written through it rather than retrofitted.
4. **`TS-BL-051` before `TS-BL-052`**, which `D.10`'s edge already requires. Build the version table
   and its activation path first: writing the engine against two columns and adding versions after
   would mean rewriting every persistence path once `G-07` arrives.
5. **`TS-BL-052`, three families in one item, `candidate_ranking` first.** It is the family
   `ai-platform-governance` D3 named, so its envelope is settled; fitment and gap follow against the
   same shape. Author each as a **new template version** of the registered family, never by editing
   an existing version. All three are Dev-deployable and production-blocked until `TS-BL-032`'s
   corpus carries their cases (`ai-platform-governance` D8).
6. **`TS-BL-053`, then `TS-BL-054` and `TS-BL-055` in either order.** Both depend on work `TS-BL-053`
   or `TS-BL-052` completes and neither depends on the other. `TS-BL-055` is the better one to take
   first where a choice exists, because the board renders insufficiency and building the board's
   insufficiency display twice is the cost of the other order.
7. **Feature-flag every surface, disabled by default.** The board, the insight endpoints and the
   projection each ship behind a declared flag; a disabled capability answers 404, so there is no
   window in which a half-built ranking surface is reachable.
8. **Record the open items in `KNOWN_ISSUES.md`** as they land: that ranking runs on the stub
   provider in Dev, and that §29 item 8's weights are provisional and unapproved — the same treatment
   the unconfirmed Hubble contract already receives. The absent provider is **not** yet in that file
   (D13); `ai-platform-governance`'s task 1.18 is what puts it there, so extend its entry if it has
   landed rather than adding a second.
9. **Rollback** is redeploy-previous-artifact plus migration-down. This feature creates tables and
   provisions one external index; it transforms no existing data. The two columns it writes on
   `candidate_posting_applications` are nullable and derived, so reverting leaves them null, which is
   the documented never-ranked state rather than a corrupt one.

## Open Questions

Each is genuinely deferrable — none changes the specs, the approach, or the task breakdown.

- **Which embedding model, and at what dimensionality.** Pinned per record and changeable through
  `VEC-005`'s re-index path, so the answer arrives as a configuration change plus a re-index rather
  than a rebuild. It is a narrower question than `OD-003`'s generation provider and is not blocked on
  it — retrieval and generation are separate decisions, which is the same distinction
  `ai-platform-governance` D1 had to make in the opposite direction.
- **Ranking criteria weights, and whether weighting is applied at all.** §29 item 8 says "where
  product approved," and no product approval exists. The configuration key and its provisional
  provenance ship; the values do not. Whether §16.4's eleven signals are weighted or simply cited as
  criteria is a product question that needs a real board in front of a real Practice Manager.
- **Per-family conciseness bounds for the three ranking families** — the actual sentence and item
  counts. `S.5` states the numbers were never given and "shouldn't be invented here," and
  `ai-platform-governance` D7 settled the general form: provisional, labelled, enforced, changeable
  through the audited path. This feature supplies three families' worth of provisional values under
  that rule.
- **The aging threshold, if any, for a stale ranking.** D5 deliberately does not force a re-rank.
  Whether a board stale for N days should raise a condition is a §29 item 9 question (aging
  thresholds for postings, interviews, feedback, offer and onboarding — ranking is not currently
  among them), answerable once real boards exist.
- **Whether the two Phase-3 vector source types need any change to the registration seam.** D2
  designs for them from §12.2's enum, and `interview-pipeline` will find out. Recorded so that
  finding out is expected rather than a surprise.
- **Vector index sizing and re-index batch size.** §29 item 15 makes the batch size configuration.
  `D07`'s volumes make any reasonable value work at Dev scale, and the real value belongs to whoever
  operates a production index — which under the current billing cap is nobody, so it is deferred with
  the same reasoning `ai-platform-governance` applied to its rate limits.
