# AI Platform and Governance — Design

## Context

See `proposal.md` — Why, for motivation, and the six delta specs under `specs/` for the behavior
contracts. This document covers only the technical decisions this feature must settle, the
redistribution of what `talentsphere-wave-1-foundation` left behind, and an honest account of what
Sprint 0 built versus decided about AI governance.

Constraints that shape the approach:

- **The model provider is undecided, and nothing in the project has ever decided it.** `OD-003`
  lists it as one of ten open decisions owned by AI Engineering. `D09` settled the stack and, on
  2026-08-17, amended one of its rows to Vertex AI Vector Search — **a retrieval decision**. No
  section of `exploration-notes.md` chooses a generation provider. See D1.
- **Three of this feature's guarantees are enforced elsewhere and incomplete without this one.**
  `platform-core`'s workflow engine and `access-control-and-admin`'s permission evaluator each
  carry a side of the advisory-only guarantee, and each names `TS-BL-029` as the side it does not
  carry.
- **This feature's entry dependency is already satisfied.** `TS-BL-027` depends on
  `platform-core`'s `TS-BL-003` (API Gateway ingress), which is **done and live in GCP Dev**. This
  is the only Phase-1 feature whose first item is unblocked today.
- **No consumer exists.** Six items across four unproposed features will call this substrate. This
  is the same position `platform-core`'s `TS-BL-006` was in, and it is mitigated the same way —
  see D3.
- **Settled stack** (`D09`): FastAPI on Python, PostgreSQL on Cloud SQL, Terraform on GCP with
  GitLab CI/CD, OpenTelemetry, and background work on the landing zone's Pub/Sub → Eventarc →
  Workflows → Cloud Run Job chain.
- **Only Local and Dev are provisionable.** The billing account caps linked projects; UAT and Prod
  are validated Terraform that is never applied. This has a specific consequence for the promotion
  gate — see D8.

## Goals / Non-Goals

**Goals:**

- One egress, established before any consumer exists, so the choice between "call through the
  gateway" and "call a provider directly" is never actually available to a Phase 2 feature.
- Guarantees that are checkable by absence rather than by inspection — no unlogged run, no second
  egress, no transition capability, no unbounded output field.
- A provider decision that costs one adapter when it lands, and costs nothing while it is open.
- Conciseness moved from a written standard to a failing test.
- An honest split of `talentsphere-wave-1-foundation`'s four AI delta specs across six
  independently deployable items, with every inherited requirement accounted for.

**Non-Goals (design-level boundaries beyond the proposal's scope):**

- **No prompt engineering.** Ten stubs that satisfy their contracts, not ten working prompts.
- **No model selection tuning, cost modelling, or latency budget.** Per-family model configuration
  exists; choosing what to put in it needs a provider and real traffic, and has neither.
- **No caching layer.** `reference/spec.md` §36 names caching among the cost mitigations. Caching
  AI output requires knowing what makes two requests equivalent, which is a per-family question
  none of the consumers exist to answer. Rate limits and input bounds ship; caching does not.
- **No streaming.** Every run is asynchronous with a retrievable status, which is what §25 requires
  and what the dispatch substrate supports. A streaming path would be a second egress shape.
- **No second evaluator, no second audit writer, no second queue.** This feature consumes all
  three.

## Decisions

### D1 — The provider is not chosen here, and the gateway is built so that it does not have to be

**The AI model provider is recorded as an open decision needing an owner, not resolved.**
`reference/spec.md` §35 `OD-003` — *"Confirm initial AI model provider, deployment model, and
gateway contract"* — is owned by AI Engineering and has never been answered. `D.9`'s `TS-BL-027`
title says "sole egress to model provider" without naming one, because there is none to name.

*Why this is not a gap to be filled by inference:* the closest available fact is `D09`'s vector
store row, and it does not reach. Vertex AI Vector Search was chosen on 2026-08-17 for
**candidate and resume retrieval**, on the specific grounds that the landing zone already enables
`aiplatform.googleapis.com` per app. Retrieval is not generation. Citing that row for a generation
provider would manufacture a decision from an adjacent one — precisely the failure `AGENTS.md`
warns about when it says an uncitable decision "hasn't actually been made yet."

**What ships instead:** a provider port with a single adapter behind it and a deterministic stub
adapter used in Local, in Dev until a provider is configured, and by the evaluation harness. This
is the shape `identity-and-access` already uses for the unconfirmed Hubble contract (`OD-001`), and
`config.yaml` already states the rule for both: *"unconfirmed external contracts sit behind
adapters: Hubble login (`OD-001`) and the AI provider (`OD-003`)."* The stub is a first-class
artifact, not a test fixture, because everything downstream keeps depending on it until the
decision lands.

*What the decision actually costs when it arrives:* one adapter implementation, one Secret Manager
entry, per-family model configuration values, and a corpus re-run (the evaluation spec requires
re-running when the provider changes, because a passing result attributed to one provider is not
evidence about another). It costs no change to any caller, any contract, or any run record shape —
which is the whole point of paying for the port now.

*Alternative considered:* pick Vertex AI provisionally and revise later. Rejected for a reason
specific to this project rather than a general preference for deferral — **a provisional choice
here does not stay provisional.** Per-family model configuration, credential handling, safety-flag
semantics, and token accounting all take their shape from the provider, and four features will
build against whatever shape exists when they arrive. `platform-core` recorded "which
administrative identity automates schema ownership" as blocked-on-a-decision rather than guessing
at it, so the gap was visible at scheduling time instead of surfacing later; this is the same
treatment for a larger decision.

*What is genuinely settled and worth stating so it is not reopened:* whichever provider is chosen,
it is reached only from the gateway, its credentials live only in Secret Manager, and its identity
is recorded on every run. Those are provider-independent and are specified now.

### D2 — Where every inherited requirement lands, stated explicitly

`talentsphere-wave-1-foundation` remains active and unarchived. Its four `ai-platform/` delta specs
carry, in four files, requirements that `D.9` splits across six independently deployable items.
`platform-core`'s D11 assigns that change's D9–D12 to this feature. This table is the accounting,
so an auditor comparing the two changes can see where each inherited requirement went and confirm
nothing was dropped.

| Inherited requirement | Inherited spec | Lands in | Capability |
|---|---|---|---|
| All AI calls route through the gateway | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| Per-family model configuration | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| Untrusted content isolation | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| Personal data minimization | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| Rate limiting and consumption bounds | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| Graceful degradation | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| Failure preservation and safe retry | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| Asynchronous execution | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| Run AI permission enforced at the gateway | `ai-gateway` | `TS-BL-027` | `ai-platform/ai-gateway` |
| **Advisory-only invocation** | `ai-gateway` | **`TS-BL-029`** | `ai-platform/advisory-only` |
| Every AI run is logged | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| Reference-only inputs | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| Safety flags | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| Reproducibility tuple | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| AI content labelled until approved | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| Human override records | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| Output feedback | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| Run log access and search | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| Retention | `ai-run-logging` | `TS-BL-028` | `ai-platform/ai-run-logging` |
| Ten registered prompt families | `prompt-registry` | `TS-BL-030` | `ai-platform/prompt-registry` |
| Template versioning | `prompt-registry` | `TS-BL-030` | `ai-platform/prompt-registry` |
| Enforced output contracts | `prompt-registry` | `TS-BL-030` | `ai-platform/prompt-registry` |
| Conciseness constraints | `prompt-registry` | `TS-BL-030` | `ai-platform/prompt-registry` |
| Promotion gate | `prompt-registry` | `TS-BL-030` (registry side) + `TS-BL-032` (corpus side) | both |
| No family invoked by a user-facing feature | `prompt-registry` | `TS-BL-030` | `ai-platform/prompt-registry` |
| **Evidence and insufficiency in contracts** | `prompt-registry` | **`TS-BL-030` (clause) + `TS-BL-031` (model)** | both |
| **Source labelling in contracts** | `prompt-registry` | **`TS-BL-030` (clause) + `TS-BL-031` (vocabulary)** | both |
| All six evaluation and security requirements | `ai-evaluation` | `TS-BL-032` | `ai-platform/ai-evaluation` |

**Four requirements are new here rather than inherited**, each with a stated reason:

- *Provider independence* (`TS-BL-027`) — D1. The inherited specs assumed a provider would be
  chosen by the time they were built.
- *Uniform request and response envelope* (`TS-BL-027`) — D3. The inherited spec fixed central
  output validation and said nothing about the request side.
- *Run records are append-only* (`TS-BL-028`) — D5. `config.yaml` states the rule product-wide and
  `access-control-and-admin` specifies it for `audit_logs`; no AI spec ever stated it for
  `ai_run_logs`.
- *Disclosure records* (`TS-BL-028`) — D11. `D05` committed to a mitigation that no spec anywhere
  carries, and that cannot take the form `D05` describes.

Plus two strengthenings: *provisional bounds are recorded as provisional* (`TS-BL-030`) and *every
registered family is covered* (`TS-BL-032`), both of which close a way the inherited requirements
could pass while being vacuous.

### D3 — The request envelope is designed against a concrete future call, without building it

`TS-BL-027` will be consumed by six items in four features that do not exist. The registration
surface for `platform-core`'s workflow engine faced the same problem and was mitigated by designing
against two real machines specified in `reference/spec.md` without registering either. The same
pattern applies here: **design the envelope against a real call shape, build no consumer.**

The chosen shape is `hiring-postings`' `TS-BL-034`, AI-drafted JD generation — the first AI
consumer in dependency order, and the only one whose inputs are fully specified today (§16.1 row 1,
§16.3's `job_description_generation` contract, `D12`/`C-12`'s PM-drafts/RM-approves flow).
Concretely, that call is:

```
request:
  family:        job_description_generation
  caller:        { actor, page: JD Workspace, action: Run AI, correlation_id }
  input:         { jd_draft_ref: <id@version>, notes: <free text, bounded> }
response:
  run_ref:       <ai_run id>
  status:        succeeded | failed
  output:        contract-satisfying draft, or
  failure:       { kind: validation | provider | configuration, detail }
```

Three things this exercise actually changed, which is the justification for doing it:

1. **The envelope carries references, not content.** A JD draft is already a record by the time the
   PM asks for AI help, so the input is `jd_draft_ref` rather than a copy of the draft. That
   generalizes: `resume_extraction` sends a resume version reference, `candidate_ranking` sends
   posting and resume references. It also makes the run log's reference-only requirement natural
   rather than a redaction step — the gateway never held the content in the first place.
2. **Contract clauses are per family; the envelope is not.** JD generation produces no claim about
   a candidate, so evidence references and source labels do not apply to it. Writing the envelope
   against a family that needs neither prevented baking candidate-insight assumptions into the
   shared shape — which is exactly what would have happened had `candidate_ranking` been the
   worked example.
3. **The conciseness bound has to be per field, not per output.** §16.3 requires
   `job_description_generation` to return *"Markdown plus structured JSON summary"*, and a job
   description is a document. A single bound over the whole output is either vacuous (large enough
   to permit a JD) or impossible (small enough to satisfy `S.5`). **The bound is therefore declared
   per free-text field** — each JD section capped individually — which the spec now states
   explicitly. This is the concrete tension `S.5` did not anticipate when it said "no long-form AI
   text anywhere in the UI," and it is resolvable, but only once.

*What is deliberately not done:* no `job_description_generation` prompt is written beyond a stub,
no JD schema is fixed, and `hiring-postings` remains free to refine its own input shape. The
envelope is the contract; the payload inside it is the consuming feature's.

### D4 — Write-before-invoke, and the one real cost of it

The gateway resolves configuration, writes the run record, invokes the provider, validates output
against the family contract, then persists output as advisory insight. **If the run record cannot
be written, the provider is never called.**

*Why the ordering rather than a transaction:* the provider call is not transactional and cannot be.
Logging after the call loses precisely the runs most worth having — the ones that crashed before
returning. `glossary.md` states that `AIRun` history "cannot be backfilled," which makes ordering
the only available enforcement.

*The cost, stated rather than discovered:* a database write is now on the critical path of every AI
invocation, and a database outage disables AI entirely rather than degrading it. That is the
correct failure direction — an AI capability that keeps working when its governance record cannot
be written is the exact failure mode this design exists to prevent — but it is a real coupling, and
it means the graceful-degradation requirement covers provider outages, not database outages. A
database outage is a product-wide outage anyway, which is why this is acceptable rather than
merely accepted.

*Alternative considered:* write the record asynchronously and reconcile. Rejected — reconciliation
implies a window in which an unlogged run exists, and an invariant with a window is a convention.

### D5 — The run log is append-only by grant, reusing the pattern already written for `audit_logs`

`ai_run_logs` gets an `INSERT`/`SELECT`-only grant for the application role, added to
`backend/sql/01_least_privilege.sql` alongside the one already written for `audit_logs`.

*Why this is nearly free:* `platform-core`'s D3 already split the database into two identities so
the runtime holds DML only and the migrator owns DDL — the split exists specifically so an
append-only grant is not decorative. `access-control-and-admin`'s D7 established the pattern and
`TS-BL-020` is where it is first proven against a real Cloud SQL table using the harness
`platform-core`'s task 2.9 provides. This feature is the second table through the same door, and
should be materially cheaper because the harness and the argument both already exist.

*The consequence that matters for D11:* append-only means a fact learned after a run completed
cannot be written onto that run. That is not a limitation to work around; it is what forces the
disclosure record to be its own record.

### D6 — `TS-BL-029` is the third side of one guarantee, not a fourth mechanism

The advisory-only guarantee has three enforcement points, in three features:

| Side | Enforced by | Feature | Status |
|---|---|---|---|
| Workflow | Transitions accepted only from an authenticated actor holding the declared permission; no caller transitions by producing advisory output | `platform-core` `TS-BL-004` task 4.10 | specified, not built |
| Authorization | `Approve` denied to the AI Service Account and ungrantable through the matrix | `access-control-and-admin` `TS-BL-018`/`TS-BL-022` | specified, not built |
| Gateway | No transition capability exists on the gateway; output persists only as advisory insight | **`TS-BL-029`, here** | this change |

`platform-core`'s task 4.10 names `TS-BL-029` explicitly as what completes it, and
`access-control-and-admin`'s service-account requirement names the other two. **So the design
question is not "how should advisory-only be enforced on the AI side" — that was settled by
inherited `D10` and by two features that already built against it. The question is what the third
side must add that the first two do not already cover.**

Two things:

1. **Output has no destination.** The workflow side establishes that nothing transitions without an
   authenticated permitted actor. It does not establish that AI output has nowhere to be written
   other than an advisory record. `TS-BL-029` enumerates persistence paths from run output and
   asserts none reaches a workflow-governed state field.
2. **The three sides are asserted together.** Three partial assertions in three features, each
   passing its own suite, is the defect shape `platform-core` D4 names: independently-correct
   implementations of one rule that no test compares, where each passes and the rule still breaks.
   `TS-BL-029` carries the verification that runs all three in one pass and reports which failed.

*Alternative considered:* have each feature assert only its own half and rely on the three
descriptions matching. Rejected on that same D4 reasoning — the control is a shared assertion, not
more coverage on either side.

*A sequencing consequence:* `TS-BL-029` depends on `platform-core`'s `TS-BL-004`, which is
not-started. The joint verification cannot pass until the workflow engine exists. The gateway-side
requirements can be built and asserted before it; the joint assertion is the last task in the item
and is what makes it done.

### D7 — Conciseness is enforced in three places, and none of them is documentation

`AGENTS.md`'s standing bar requires "an explicit, machine-checkable length bound (sentence count,
word cap), not just a JSON shape." That is satisfied by three separate mechanisms, deliberately:

1. **The registry refuses to register** a family whose contract declares a free-text field with no
   bound. This is what stops the standard being followed for nine families and forgotten for the
   tenth.
2. **The gateway rejects output** exceeding a declared bound, as a contract violation recorded as a
   failed run. Verbosity is a validation failure at runtime, not a review comment.
3. **The harness fails a family** that produces over-long output on any evaluated case — and fails
   a family that declares a bound no case exercises. A bound nothing tests is not enforced.

*Why all three rather than the cheapest one:* each covers a different failure. Without (1) a bound
can be absent; without (2) a bound can be declared and unenforced at runtime; without (3) a bound
can be enforced and wrong — set so loosely that it never fires, which reads identically to
compliance. That third case is `platform-core` D5's silent-success class, and it is why the "bound
present but never exercised" scenario exists.

*What is not decided:* the numbers. `S.5` says plainly that concrete per-family bounds were never
given and "shouldn't be invented here." Provisional values ship carrying machine-readable
provenance marking them provisional, changeable through the audited configuration path. A
wrong-but-labelled-and-enforced bound is fixable configuration; an absent one is an unenforceable
contract.

### D8 — The registry's promotion gate bites before the corpus exists, and that is the correct order

`TS-BL-030` refuses to activate a template version in production with no passing corpus run
recorded. `TS-BL-032`, which produces those runs, depends on `TS-BL-030` and therefore lands after
it. Between the two, **no template version can be promoted to production at all.**

*Why this is right rather than a sequencing bug:* production is not provisionable under the current
billing cap, no user-facing feature invokes a family in this scope, and the gate failing closed is
the behavior the requirement asks for. A gate that permits activation until its checker exists is
`platform-core` D5's conditional-gate-that-skips, which is the failure mode that produced three
days of deploys that deployed nothing.

*What this means practically:* Dev and Local activate versions freely; the gate's production
branch is exercised by the harness once `TS-BL-032` lands, and is asserted to refuse before that.
The alternative — build the corpus first — is not available, because a corpus needs contracts to
evaluate against.

### D9 — Evidence labeling has no hard dependency on the registry, and the seam is stated rather than assumed

`D.9` gives `TS-BL-031` one dependency: `TS-BL-028`. Not `TS-BL-030`. That looks wrong at first
glance, since source labels appear in prompt contracts — and it is correct on inspection, so it is
recorded here rather than silently amended.

`TS-BL-031` owns the **substrate**: the closed source vocabulary, the evidence-reference record,
permission-scoped resolution, and the no-personal-data constraint. None of that needs a prompt
template. Its real dependency is `TS-BL-028`, because the reason evidence labeling exists at all is
the one `glossary.md` gives — keeping an Administrator's run-log access from becoming a personal
data back door — which is a property of the run log.

`TS-BL-030` owns the **contract clause**: that a family producing candidate insight must label each
claim using that vocabulary. If `TS-BL-030` lands first, the clause is written against the declared
vocabulary as an interface, which is the same stub-and-integrate shape `platform-core` D8 uses for
the audit port and `identity-and-access` uses for the permission resolver.

*So there is no dependency edge, but there is a preferred order.* Building `TS-BL-031` before
`TS-BL-030` means the clause is written against a real vocabulary rather than a declared one. That
is a sequencing recommendation in the Migration Plan, not a dependency — inventing an edge D.9 does
not carry would misrepresent what actually blocks what.

*Downstream consequence worth naming:* `design-system`'s evidence-source label component already
specifies that it "renders the source value it is given without interpreting or deriving it." That
clause is only satisfiable if exactly one component derives source values. `TS-BL-031` is that
component, and the vocabulary is closed for the same reason.

### D10 — AI runs register as a job type; they do not get a queue

Every AI run executes asynchronously through `platform-core`'s single dispatch pattern, registering
as a job type with its own retry declaration rather than implementing dispatch here.

*Why:* `platform-core`'s async-orchestration spec has a scenario that fails the suite on background
work started outside the shared pattern, and `TS-BL-005` already accepted the same constraint for
notification delivery rather than becoming its first exception. `reference/spec.md` §25 lists AI
ranking among the background job types alongside resume parsing.

*The one thing this feature must implement rather than inherit:* idempotency at the effect
boundary. `platform-core` D6 records this as the honest cost of the Pub/Sub chain — delivery is
at-least-once, so a duplicated run request must produce exactly one provider invocation and exactly
one output record. For AI runs this is more than a correctness concern: a duplicate is a duplicate
charge and a second run record for one logical request, which would corrupt the divergence
measurement `G-06`'s override records are supposed to support.

### D11 — `D05`'s mitigation cannot be a field on the run, and nothing currently carries it

**A genuine gap found in `exploration-notes.md` during this propose conversation, corrected there
rather than worked around here**, per `AGENTS.md`'s convention and the precedent set by `D09`'s
background-jobs supersession and `S.6`.

`D05` accepted a real trade-off — Interviewers see full AI context before the interview, against
the recommendation, with anchoring named as the known cost. Its agreed mitigation reads: *"Every
`AIRun` records that the score was **visible pre-interview**, so a later review can account for
it."*

That mitigation cannot take the form it describes, for two reasons that are themselves decided
elsewhere:

- **The run record is written before invocation** (inherited `D9`, `AI-002`). At write time, whether
  the output will later be shown to an interviewer is unknowable — the ranking run happens at
  posting or submission time, the interview happens days later.
- **The run record is append-only** (`config.yaml`; D5 above). So the field cannot be set later
  either.

A ranking run is also disclosed to many actors on many occasions, so a boolean on the run could not
represent it even if the ordering allowed.

**Resolution: a separate, append-only disclosure record, linked to the run**, capturing actor,
output, time, and context. It answers exactly the question `D05` wanted answered — whether a human
judgement was formed before or after seeing AI output — and it does so from stored data rather than
from a flag nothing could have set.

*Why this feature owns it:* `matching-and-ranking`'s `TS-BL-054` owns the AI-context **visibility
rules** for Interviewers (`D05`/`C-10`). The rules are that feature's; the record they must leave
is substrate, and substrate that a later feature would have to retrofit is exactly what this
feature exists to prevent. It is also unbackfillable for the same reason `AIRun` is.

*Why it was missed:* `D05` was decided in Round 2, before `AIRun`'s write-before-invoke ordering
existed as a decision at all. Nothing subsequently reconciled the two. The dated correction block
added to `D05` records this.

### D12 — What Sprint 0 actually built and decided about AI governance: nothing, and four decisions

Same reconciliation discipline the other four Phase-1 features applied. The honest answer is short.

**Sprint 0 built no AI code.** Sprint 0 was 13/13 tasks against a 196-task Wave 1 — 7%. Its own
handover states that its ~2,000 lines are "all of it platform-layer; **no hiring feature exists
yet**, by design." AI work was Sprints 13, 14 and 15 in the superseded plan and **none of the three
ran**. There is no `ai_run_logs` table, no gateway, no registry, no corpus, and no AI module: the
backend's directories are `core/`, `api/`, `db/`, `audit/`, `features/`, `runtime_config/`,
`middleware/` and `migrations/`.

**`KNOWN_ISSUES.md` carries no AI entry.** That is correct rather than reassuring — nothing AI
exists to be broken. Recorded because an empty section can otherwise read as a clean bill of
health.

**What Sprint 0 did decide** — four decisions in `talentsphere-wave-1-foundation`'s `design.md`,
plus four `ai-platform/` delta specs, all of which this feature inherits:

- **D9** — the gateway is the only egress, and logging precedes invocation. Carried forward whole;
  see D4.
- **D10** — advisory-only is enforced structurally, not by review. Carried forward and completed;
  see D6.
- **D11** — conciseness is a contract clause, not prompt wording. Carried forward and strengthened;
  see D7.
- **D12** — prompt families are registered and tested but not authored in this wave. Carried
  forward whole, and it is the reason `TS-BL-030` ships stubs.

**One thing Sprint 0 did build that belongs to this feature's story**, small but real: the global
error handler's redaction tests assert that **raw prompt content cannot reach an API response**,
alongside DSNs, tokens and stack traces, and the OpenTelemetry redaction processor applies
centrally rather than at call sites. That is the only shipped line of code in the repository that
exists because of AI governance, and it was written before any AI existed to govern. It is also
already-satisfied scope for this feature's sensitive-data-exposure security probe.

**Three Sprint 0 artifacts this feature consumes rather than rebuilds:** the API Gateway ingress
(`TS-BL-003`, done — which is why `TS-BL-027`'s dependency is already satisfied), the
edge-assigned correlation identifier that survives into background tasks (which is what lets a
background AI run share an identifier with the audit record of the request that triggered it), and
the feature-flag registry plus audited runtime-configuration path (which is how a family's active
template version and its conciseness bounds change without a deployment).

### D13 — Two item-boundary refinements, recorded rather than assumed

Under `D.11`'s permission for a feature's propose conversation to refine its internal item
boundaries:

- **`TS-BL-028` carries the disclosure record** (D11), which `D.9`'s one-line title does not
  mention. It belongs with the run log because it is linked to it, append-only for the same reason,
  and unbackfillable for the same reason.
- **`TS-BL-032` carries the security suite as well as the evaluation corpus.** `D.9`'s title says
  "adversarial testing, C-09" — and `C-09`'s resolution adopts §27.1's *Security Tests* row in the
  same breath as its *AI Evaluation Tests* row. Splitting them would put prompt-injection cases in
  one item and prompt-injection probes in another.

Neither changes a dependency edge. `D.9`'s six items and their `depends_on` values are otherwise
carried through unchanged.

## Risks / Trade-offs

- **The provider decision may land late and be unlike the stub** — a provider with different safety
  semantics, token accounting, or failure taxonomy than the stub models → Mitigated by keeping the
  port narrow: the adapter returns contracted output or a typed failure, and everything richer
  (safety flags, token usage) is optional on the run record because `AI-002` already says "where
  available." Residual risk accepted: the first real provider will refine the adapter, and that is
  expected rather than a defect.
- **The substrate is built before any real consumer exists**, risking a contract that fits no
  actual caller — the same position `platform-core`'s `TS-BL-006` was in → Mitigated by D3's
  design against a concrete future call, and by exercising every family end-to-end through the
  harness and a diagnostic path rather than leaving them compiled but unrun. Residual risk
  accepted: Phase 2's first real consumer will refine the payloads.
- **Ten stub prompts can satisfy their contracts while being useless**, giving a false signal that
  the families work → Mitigated by scoping honestly: inherited `D12` says the families are
  registered and tested, not authored, and the specs say so. The thing being tested is the
  contract, not the prompt. The risk is not that the stubs are weak but that someone later reads
  green as "the AI works," which is why `TS-BL-030` carries an explicit no-user-facing-invocation
  requirement.
- **Provisional conciseness bounds may be set so loosely that they never fire**, making the whole
  mechanism decorative → Mitigated by the harness failing a family whose bound no case exercises,
  and by provenance marking each bound provisional so it reads as unfinished rather than decided.
  This is the specific failure D7 exists to prevent and the one most likely to be under-built.
- **Three-sided advisory-only verification depends on `platform-core`'s `TS-BL-004`**, so
  `TS-BL-029` cannot fully pass until an unrelated feature's item lands → Recorded as a real
  dependency rather than a sequencing note; `D.9` already carries the edge. The gateway-side
  assertions land independently.
- **A database write now sits on the critical path of every AI call** (D4) → Accepted deliberately;
  the alternative permits an unlogged run. Named here because it makes AI availability strictly
  worse than provider availability, which the graceful-degradation requirement does not cover.
- **At-least-once dispatch means a duplicated run is a duplicated provider charge** and a second
  run record → Idempotency at the effect boundary is a spec requirement with its own scenario, per
  D10. `platform-core` already identified this as the one place its rejected alternative was
  genuinely cheaper, which makes it the one most likely to be treated as an implementation detail.
- **Two active changes describe the same four `ai-platform` capability paths** → Accepted as
  interim, exactly as the other four Phase-1 features accepted it. D2 is the accounting;
  `talentsphere-wave-1-foundation` is the source being migrated *from*. **With this change, all
  five Phase-1 features have drawn their inheritance across** and that change has no unaccounted
  content left.
- **`TS-BL-031`'s permission-scoped reference resolution depends on an evaluator that does not
  exist yet** (`access-control-and-admin`'s `TS-BL-018`) → Built against its interface, the same
  stub-and-integrate shape three other features already use. `D.9` does not carry this edge
  because the substrate is buildable without it; only the denial behavior needs the real evaluator.

## Migration Plan

There is no data migration; this is substrate, and none of it has a predecessor in production.
Sequence:

1. **`TS-BL-027` first, and it is unblocked today.** Its only dependency, `platform-core`'s
   `TS-BL-003`, is done and live. The provider port and stub adapter are the first task, so
   everything after it is built against a real interface rather than an imagined provider.
2. **`TS-BL-028` immediately after, in the same sprint if possible.** Inherited `D9` is explicit
   that the log substrate must land *"in the same sprint as the gateway and ahead of it, not in the
   sprint after — an invariant that holds from the second sprint onward is not an invariant."*
   Practically this means the `ai_run_logs` migration and the write-before-invoke ordering are
   built with the gateway even though the item is tracked separately, and the gateway is not
   deployed to Dev with a provider configured until they are.
3. **`TS-BL-031` before `TS-BL-030`** where staffing allows. There is no dependency edge (D9), but
   building the vocabulary first means the registry's source-label clause is written against a real
   enumeration rather than a declared one. If `TS-BL-030` goes first, it is written against the
   declared vocabulary as an interface — which works, and costs one integration pass.
4. **`TS-BL-030`, then `TS-BL-032`.** The corpus needs contracts to evaluate against. Between them,
   production promotion is refused for every family, which is correct (D8).
5. **`TS-BL-029` last, gated on `platform-core`'s `TS-BL-004`.** The gateway-side assertions can be
   written at any point after `TS-BL-027`; the three-sided verification is the closing task and is
   what makes the item done.
6. **Feature-flag the gateway.** It ships disabled and is enabled in Dev by a flag change, matching
   how `platform-core` ships the async substrate. A disabled capability answers 404, so there is no
   window in which a partly-built gateway is reachable.
7. **Rollback** is redeploy-previous-artifact plus migration-down. This feature creates tables and
   adds one grant; it transforms no existing data, so rollback stays low-risk throughout. The one
   irreversible act is the append-only grant, which is a narrowing and safe to leave in place.

## Open Questions

Each is genuinely deferrable — none changes the specs, the approach, or the task breakdown.

- **Which AI model provider, in which deployment model** (`OD-003`, owned by AI Engineering). **The
  headline open question of this feature**, and the reason D1 exists. It is not deferred for
  convenience: nothing in `exploration-notes.md` ever chose one, and `D09`'s Vertex AI Vector
  Search row is a retrieval decision that does not reach generation. The gateway is
  provider-agnostic by construction and the answer costs one adapter, one secret, per-family
  configuration values, and a corpus re-run. Until it lands, Dev runs on the stub — the same
  position `identity-and-access` is in with the unconfirmed Hubble contract, and it should be
  recorded in `KNOWN_ISSUES.md` the same way.
- **Per-family conciseness bounds** — the actual sentence, item and character counts. `S.5` states
  that concrete numbers were never given and "shouldn't be invented here." Provisional values ship
  labelled as provisional and change through the audited configuration path, so the answer arrives
  as configuration rather than a release.
- **AI run log retention period.** Company policy input, the same shape as
  `access-control-and-admin`'s audit retention question. The mechanism and its disposal
  attribution are specified; the interval is not. `D22` caps configurable windows by retention, so
  the two must be set consistently once both are known.
- **Which roles hold `Run AI` on which pages, and whether the AI Service Account holds it at all.**
  The gateway enforces whatever the matrix says; the seeded values are
  `access-control-and-admin`'s `TS-BL-022` and are already flagged there as needing business
  sign-off.
- **Whether the controlled exception permitting candidate contact information in a prompt ever
  gets used** (`AI-006`). The mechanism is required and specified. No family in §16.1 currently
  needs it, so it may ship and never fire, which is an acceptable outcome for an escape hatch.
- **Rate limit and consumption cap values.** Required, unnumbered. They belong to whoever operates
  Dev once a real provider is configured and real cost is observable — the same treatment
  `platform-core` gave its queue-depth alert threshold.
