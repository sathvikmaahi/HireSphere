# AI Platform and Governance

## Why

TalentSphere's one rule that governs everything else — *AI recommends, a human decides, always* —
is a claim about **capability**, not about care. `project.md` states it is "enforced structurally
(the AI has no capability to trigger those transitions), not by convention or code review." This
feature is where that structure gets built: one governed egress to a model provider, a run record
written before the provider is ever called, and prompt contracts that bound what a model may
return — including how *long* it may be.

**Why now, and why before any AI feature.** Two of these guarantees cannot be added later, and one
is already half-built and waiting:

- **`AIRun` history cannot be backfilled** (`glossary.md`). Logging after the fact loses exactly
  the runs most worth having — the ones that crashed. An invariant that begins holding in Phase 2
  is not an invariant, so the run log must exist before the first real provider call, not
  alongside it.
- **The advisory-only guarantee is currently half-built.** `platform-core`'s `TS-BL-004` task 4.10
  already commits to the workflow-side half — *"assert no caller can trigger a transition by
  producing advisory output — the workflow half of the advisory-only guarantee that
  `ai-platform-governance`'s `TS-BL-029` completes."* `access-control-and-admin` carries a third
  half (`Approve` is unassignable to the AI Service Account). Neither is complete on its own:
  both close the door from the workflow side, and neither establishes that the gateway holds no
  transition capability at all. `TS-BL-029` is not a separately-reasoned advisory-only mechanism —
  it is the missing side of a guarantee two other features already built against.
- **Four not-yet-proposed features will consume the gateway** — `hiring-postings`' AI-drafted JD
  generation, `candidate-intake`'s LLM enrichment, `matching-and-ranking`'s ranking engine, and
  `interview-pipeline`'s AI-drafted scorecards. Every one of them either calls through this
  gateway or calls a provider directly. There is no third option, and the choice is made by
  whichever of the two exists first.

**Nothing here is built.** Sprint 0 of `talentsphere-wave-1-foundation` shipped platform-layer
code only. The AI work was Sprints 13–15 in the superseded plan and none of them ran — see
`design.md` D12 for the honest reconciliation of what Sprint 0 actually decided versus built.

## What Changes

Six backlog items, `TS-BL-027` through `TS-BL-032` from
[D.9](../talentsphere/exploration-notes.md). None is built.

- **`TS-BL-027` AI Gateway — the sole egress to a model provider.** Per-family model configuration,
  `Run AI` permission verified before invocation (§9.3), untrusted-content isolation and
  sanitization (`AI-012`, `AI-013`, `AI-014`), personal-data minimization (`AI-006`), rate limits
  and input bounds, asynchronous execution returning a job identifier, and graceful degradation so
  a provider outage never blocks non-AI work (`G-12`). **The provider itself is deliberately not
  chosen here** — see Open Questions in `design.md`; the gateway is built against a provider port
  with a deterministic stub behind it, the same shape `identity-and-access` uses for the
  unconfirmed Hubble contract. Depends on `platform-core`'s `TS-BL-003`, which is **done**.
- **`TS-BL-028` `AIRun` logging, written before invocation.** The `ai_run_logs` table and the
  write-before-invoke ordering: if the run record cannot be written, the provider is never called
  (`AI-002`). Reference-only inputs so an Auditor's access never becomes a personal-data back door,
  safety flags, the reproducibility tuple, human override records (`G-06`), output feedback
  (`AI-011`), search, and retention. Also carries the **disclosure record** that `D05`'s
  anti-anchoring mitigation needs and that nothing currently provides — see `design.md` D11.
  Depends on `TS-BL-027` and `platform-core`'s `TS-BL-002`.
- **`TS-BL-029` Advisory-only structural enforcement.** The AI-side completion of the guarantee
  described above: the gateway holds no transition capability, model output persists only as
  advisory insight or draft content and never into a workflow-governed state field, and an
  enumeration test proves no path exists from a gateway output to a transition. Depends on
  `TS-BL-027` and `platform-core`'s `TS-BL-004`.
- **`TS-BL-030` Prompt template registry — ten families, versioned.** Every family from §16.3, each
  with at least one version, an immutable version history, a configurable active version, and a
  machine-checkable output contract. **Every contract carries an explicit conciseness bound**
  (`S.5`) — maximum sentences, items, or characters per free-text field, validated by the same code
  that validates shape, so verbosity is a validation failure rather than a style note. Depends on
  `TS-BL-027`.
- **`TS-BL-031` Evidence-source labeling substrate.** The authority for whether a claim is
  resume-sourced, interview-sourced, scorecard-sourced, or a human decision (`G-02`, `UI-005`), the
  evidence-reference model behind it, and resolution of a reference to its source record under the
  reader's own permissions. `design-system`'s evidence-source label component already specifies
  that it "renders the source value it is given without interpreting or deriving it" — this item is
  what does the deriving. Depends on `TS-BL-028`.
- **`TS-BL-032` AI evaluation harness — adversarial testing.** The maintained corpus and CI gate
  adopted in full by [C-09](../talentsphere/exploration-notes.md): low-information inputs,
  adversarial and prompt-injection inputs, conflicting evidence, protected-attribute redaction,
  insufficiency outputs, evidence citation (§27.1), **and conciseness compliance** — a template that
  violates its own declared bound fails evaluation, which is what makes `S.5` structural rather
  than aspirational. Plus the security suite and the promotion gate: no template version becomes
  active in production without a recorded passing run (`AI-015`, `ENG-008`). Depends on
  `TS-BL-030`.

**Explicitly not in this change:**

- **No prompt authorship.** Ten families are *registered, contracted, and tested*; their prompt
  text is a stub sufficient to exercise the contract. Writing ten real prompts against schemas
  Phase 2 will refine guarantees rework, while the contract and the corpus are exactly what must
  exist before the first real call (inherited `D12`).
- **No AI-consuming feature.** No screen invokes a family. JD drafting is `hiring-postings`, resume
  enrichment is `candidate-intake`, ranking and fitment/gap summaries are `matching-and-ranking`,
  scorecard drafting is `interview-pipeline`.
- **No vector store, no retrieval.** Vertex AI Vector Search is settled for candidate retrieval
  (`D09`, superseding `C-01`) and belongs to `matching-and-ranking`'s `TS-BL-049`–`TS-BL-050`.
  Retrieval and generation are different decisions; **settling one did not settle the other**.
- **No permission evaluator and no audit writer.** `Run AI` and the `Approve` prohibition are
  `access-control-and-admin`'s `TS-BL-018`/`TS-BL-022`; this feature consumes them.
- **No workflow engine and no state machine.** The transition-side half of advisory-only is
  `platform-core`'s `TS-BL-004`.
- **No fairness or bias measurement.** `PRV-006` and `OD-010` forbid collecting demographic or
  protected-class data without legal approval. Protected-attribute testing verifies *redaction and
  exclusion only*; the bias-audit non-goal survives, and only the eval-harness non-goal was
  reversed (`C-09`).
- **No AI Audit Dashboard.** `TS-BL-028` provides search over run records; the reporting surface is
  `insight-and-reporting`'s `TS-BL-074`.

## Capabilities

### New Capabilities

`openspec/specs/` is empty — nothing has been archived or synced — so every capability below is
new. Four paths are **preserved exactly** from `talentsphere-wave-1-foundation` rather than
renamed, matching what `platform-core`, `identity-and-access` and `access-control-and-admin` did
with theirs.

- `ai-platform/ai-gateway`: the single governed egress — routing, per-family model resolution,
  untrusted-content isolation, personal-data minimization, rate limiting, asynchronous execution,
  degradation, and permission verification before invocation. *(`TS-BL-027`)* — **path preserved**
- `ai-platform/ai-run-logging`: the permanent record of every invocation and of every human
  decision that diverges from one — write-before-invoke, reference-only inputs, safety flags,
  reproducibility, overrides, feedback, disclosure, search, and retention. *(`TS-BL-028`)* —
  **path preserved**
- `ai-platform/advisory-only`: the absence of capability — that no AI output can reject, shortlist,
  select, offer, hire, onboard, or close, because no code path exists by which it could.
  *(`TS-BL-029`)*
- `ai-platform/prompt-registry`: the versioned registry and its enforced output contracts,
  including the conciseness bound every family must declare. *(`TS-BL-030`)* — **path preserved**
- `ai-platform/evidence-labeling`: the source vocabulary, the evidence-reference model, and
  permission-scoped resolution of a reference to the record it names. *(`TS-BL-031`)*
- `ai-platform/ai-evaluation`: the maintained corpus, the security suite, the CI gate, and the
  promotion gate that together decide whether a template version may go live. *(`TS-BL-032`)* —
  **path preserved**

### Modified Capabilities

None. No requirements exist under `openspec/specs/` to modify.

**Overlap to resolve outside this change.** `talentsphere-wave-1-foundation` remains active and
unarchived, and its four `ai-platform/` delta specs carry, in four files, requirements that D.9
splits across six independently deployable items. `design.md` D2 carries the
requirement-by-requirement redistribution so an auditor comparing the two changes can see where
each inherited requirement landed. Retiring that change spans five features and is not this
change's to perform — but `access-control-and-admin` already recorded that its own proposal
"completes the set"; this change is the last of the five to draw its inheritance across, and D2 is
where the `ai-platform` quarter of that inheritance is accounted for.

## Impact

**New application code** — `backend/app/ai/` containing the gateway (`gateway.py`), the provider
port and its stub adapter (`providers/`), the prompt registry and contract validator
(`registry/`, `contracts/`), the run-log writer, the evidence-source vocabulary and reference
resolver, and `backend/app/api/routes/ai/` for the run-status, run-search, feedback and override
endpoints. Evaluation corpus and harness under `backend/tests/ai_eval/`, wired into CI as its own
gate. No frontend screens: `design-system`'s AI-disclosure and evidence-source label components
already exist for the features that will consume them.

**New tables** — `ai_run_logs`, `prompt_templates` and `prompt_template_versions`,
`ai_output_feedback`, `ai_override_records`, `ai_disclosure_records`, and the evidence-reference
records. All by reversible migration, applied by the migration identity.

**Existing code consumed, not modified** — `platform-core`'s API Gateway ingress (`TS-BL-003`,
already live), the correlation identifier assigned at the edge, the feature-flag registry, the
audited runtime-configuration path (which is how the active template version per family changes
without a deployment), the async dispatch pattern from `TS-BL-006` (AI runs are background work
and register as a job type rather than growing their own queue), and
`backend/sql/01_least_privilege.sql`, which gains an `INSERT`/`SELECT`-only grant on
`ai_run_logs` alongside the one already written for `audit_logs`.

**Infrastructure** — a Secret Manager entry for provider credentials, reachable by the service
identity and by nothing else, and one runtime-configuration key per family for the active
template version. No new GCP service until the provider decision lands: if the answer is Vertex
AI, `aiplatform.googleapis.com` is already enabled per app by the landing zone; if it is anything
else, egress and credential handling change and nothing else does. That containment is the point
of the provider port.

**Downstream features that block on this one** — six items in D.10 name `TS-BL-027` or `TS-BL-030`
directly: `TS-BL-034` (AI-drafted JD generation → `TS-BL-030`), `TS-BL-044` (LLM enrichment →
`TS-BL-027`), `TS-BL-051` (RankingScore tuple → `TS-BL-030`), `TS-BL-052` (AI Ranking Engine →
`TS-BL-027`, the highest-volume AI surface in the product), `TS-BL-060` (AI-drafted scorecards →
`TS-BL-027`), and `TS-BL-074` (the AI Audit Dashboard → `TS-BL-028`). **That split is itself
informative** — a consumer needing a *contract* depends on the registry, a consumer needing an
*invocation* depends on the gateway, and no consumer depends on both. `design.md` D3 designs the
request contract against the first of them.

**External constraints carried, not solved here** — the **AI model provider is undecided**
(`OD-003`, owned by AI Engineering). It is not deferred out of convenience: nothing in
`exploration-notes.md` ever chose one, and `D09` settled the *vector store*, which is a retrieval
decision. The gateway is provider-agnostic by construction and the decision changes one adapter.
**Per-family conciseness numbers** are likewise open — `S.5` states plainly that a concrete number
"wasn't given and shouldn't be invented here." Provisional bounds ship, recorded as provisional,
because a wrong-but-enforced bound is fixable configuration while an absent one is an unenforceable
contract.
