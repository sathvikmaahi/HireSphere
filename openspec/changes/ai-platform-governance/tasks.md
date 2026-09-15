# AI Platform and Governance — Backlog

This feature's complete backlog: six items, `TS-BL-027` through `TS-BL-032`, decomposed in
`exploration-notes.md` D.9. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.

**Nothing here is built.** Sprint 0 of `talentsphere-wave-1-foundation` shipped platform-layer code
only — its own handover states "no hiring feature exists yet, by design." The AI work was Sprints
13–15 in the superseded plan and none of the three ran. See `design.md` D12 for what Sprint 0
decided as opposed to built. Three Sprint 0 artifacts are consumed rather than rebuilt: the API
Gateway ingress, the edge-assigned correlation identifier that survives into background tasks, and
the feature-flag plus audited runtime-configuration path.

**This feature's entry dependency is already satisfied.** `TS-BL-027` depends on `platform-core`'s
`TS-BL-003`, which is **done and live in GCP Dev** — the only Phase-1 feature whose first item is
unblocked today.

**Dependency edges pointing outside this feature.** `TS-BL-027` needs `platform-core`'s
`TS-BL-003` (done); `TS-BL-028` needs `platform-core`'s `TS-BL-002`; `TS-BL-029` needs
`platform-core`'s `TS-BL-004`. Three items additionally build against interfaces owned elsewhere
that are not `depends_on` edges because the work is buildable without them:
`access-control-and-admin`'s `TS-BL-018` (the permission evaluator, for `Run AI` and for
permission-scoped reference resolution), `TS-BL-020` (the durable audit writer, for the shared
correlation identifier), and `platform-core`'s `TS-BL-006` (the dispatch pattern AI runs register
against). In the other direction, six items in D.10 name `TS-BL-027`, `TS-BL-028` or `TS-BL-030` as
their enabler: `TS-BL-034`, `TS-BL-044`, `TS-BL-051`, `TS-BL-052`, `TS-BL-060` and `TS-BL-074`.

**Two boundary refinements this feature makes**, recorded in `design.md` D13 under D.11's
permission to refine internal item boundaries: `TS-BL-028` carries the disclosure record that
`D05`'s corrected mitigation requires, and `TS-BL-032` carries §27.1's security suite alongside the
evaluation corpus, since `C-09` adopted both rows together. Neither changes a dependency edge.

**One decision is deliberately open and must not be resolved during apply.** The AI model provider
is `OD-003`, owned by AI Engineering, and nothing in `exploration-notes.md` has ever chosen one —
`D09`'s Vertex AI Vector Search row settled *retrieval*, not generation. Every item below is built
against the provider port, never against a named provider. See `design.md` D1.

---

## 1. TS-BL-027 — AI Gateway: the sole egress to a model provider

**Goal:** exactly one place in the product can reach a model provider, and it verifies permission,
isolates untrusted content, minimizes personal data, bounds consumption, and degrades without
taking the rest of the product down. Covers `ai-platform/ai-gateway`.

```yaml
backlog_items:
  - id: TS-BL-027
    feature: ai-platform-governance
    depends_on: [TS-BL-003]
    status: not-started
```

- [ ] 1.1 Define the provider port — contracted output or a typed failure, nothing richer — and
      implement the deterministic stub adapter behind it as a first-class artifact, not a test
      fixture, so Local and Dev run without a provider decision (design D1)
- [ ] 1.2 Implement the uniform request/response envelope — family, caller context, typed input
      payload in; run reference, status, and contracted output or failure detail out — designed
      against the AI-drafted JD call shape in design D3 **without building that consumer**, and
      assert two different families use the same envelope
- [ ] 1.3 Implement the gateway as the sole egress, with provider credentials reachable only from
      it, and add the enumeration test that fails if a second outbound call site to a model
      provider appears anywhere in the codebase
- [ ] 1.4 Implement per-family model configuration resolution — provider, model name, model
      version — failing a run with a configuration error where a family has none rather than
      falling back to a default model
- [ ] 1.5 Route model configuration changes through the audited runtime-configuration path, so a
      model change is audited with previous and new value and needs no deployment
- [ ] 1.6 Implement `Run AI` permission verification before any provider invocation, taking the
      verdict from the central permission evaluator rather than from logic local to the gateway.
      Depends on `access-control-and-admin`'s `TS-BL-018`; build against its interface if it has
      not landed
- [ ] 1.7 Assert a background run under the AI Service Account is evaluated by the same evaluator
      on the same terms as a human caller
- [ ] 1.8 Implement untrusted-content isolation: system instructions separated from
      document-derived content, content sanitized and passed as delimited data, and tool or action
      capability restricted during document processing (`AI-012`, `AI-013`, `AI-014`)
- [ ] 1.9 Implement personal-data minimization — contact fields excluded from prompt payloads by
      default and the exclusion recorded on the run, approved exceptions recorded so they can be
      audited, and protected-attribute redaction applied to scoring inputs (`AI-005`, `AI-006`)
- [ ] 1.10 Implement rate limiting per caller and per family, and per-family input size bounds
      rejecting oversized input before provider invocation
- [ ] 1.11 Implement asynchronous execution returning a job identifier with a retrievable status
      endpoint, dispatched as a registered job type against `platform-core`'s single dispatch
      pattern rather than a queue owned here (design D10)
- [ ] 1.12 Implement idempotency at the effect boundary so a duplicated dispatch produces exactly
      one provider invocation and exactly one output record — a duplicate is a duplicate charge and
      would corrupt the divergence measurement `G-06` depends on (design D10)
- [ ] 1.13 Implement graceful degradation: a provider outage leaves every non-AI page, record and
      workflow action fully usable, with AI-triggering controls reporting temporary unavailability
      (`G-12`)
- [ ] 1.14 Implement per-item success and failure reporting for partial batch failures, and safe
      retry preserving failure detail without touching workflow state
- [ ] 1.15 Ship the gateway behind a feature flag, disabled by default, so no window exists in
      which a partly-built gateway is reachable — a disabled capability answers 404 (design,
      Migration Plan step 6)
- [ ] 1.16 Provision the Secret Manager entry for provider credentials, readable by the service
      identity and by nothing else, and assert no credential-shaped value reaches source or
      configuration through the existing CI gate
- [ ] 1.17 Write the integration suite against the stub provider covering success, contract
      violation, timeout, outage degradation, rate-limit rejection, and duplicate dispatch
- [ ] 1.18 Record the unresolved provider decision (`OD-003`) in `KNOWN_ISSUES.md`, in the same
      form the unconfirmed Hubble contract (`OD-001`) already takes

---

## 2. TS-BL-028 — AIRun logging, written before invocation

**Goal:** no AI run exists that was not logged first, the log is a governance record rather than a
personal-data store, and the question "what did a human see, and when" is answerable from stored
data. Covers `ai-platform/ai-run-logging`.

```yaml
backlog_items:
  - id: TS-BL-028
    feature: ai-platform-governance
    depends_on: [TS-BL-027, TS-BL-002]
    status: not-started
```

**Sequencing note carried from inherited `D9`:** this item lands with `TS-BL-027` and ahead of it,
not in the sprint after. An invariant that begins holding once the second item ships is not an
invariant. The gateway is not deployed to Dev with a real provider configured until 2.1–2.3 are
done.

- [ ] 2.1 Migrate `ai_run_logs` by reversible migration with provider, model name, model version,
      template identifier and version, input and output references, token usage where available,
      safety flags, status, start and completion timestamps, and failure detail
- [ ] 2.2 Add the `INSERT`/`SELECT`-only grant for `ai_run_logs` to
      `backend/sql/01_least_privilege.sql` alongside the one already written for `audit_logs`, and
      prove the narrowing against a real Cloud SQL table using `platform-core`'s task 2.9 harness —
      not against local PostgreSQL, where the developer is a superuser (design D5)
- [ ] 2.3 Implement write-before-invoke ordering, and test that a run whose record cannot be
      written is never issued to the provider
- [ ] 2.4 Add the enumeration test asserting every invocation path writes a run record before
      reaching the provider, so a path added later that does not fails the suite
- [ ] 2.5 Implement reference-only input capture, and test that no resume text, contact detail or
      other personal data can reach a run record — the reason evidence labeling exists at all
      (`glossary.md`)
- [ ] 2.6 Implement safety-flag capture with retrieval by flag category
- [ ] 2.7 Verify the reproducibility tuple: input references, template version and model version
      together identify what produced any past output, and a referenced record versioned forward
      since the run still resolves to the version the run consumed
- [ ] 2.8 Implement AI-generated and AI-assisted marking that persists until a human approval is
      recorded with actor and timestamp
- [ ] 2.9 Migrate and implement the **disclosure record**: append-only, linked to the run, carrying
      actor, output, time and context; assert it leaves the run record unchanged, that multiple
      disclosures of one output are recorded separately, and that a reviewer can answer whether a
      judgement was formed before or after seeing AI output from stored data alone (design D11, and
      the dated correction to `D05` in `exploration-notes.md`)
- [ ] 2.10 Implement the human override record: mandatory reason, original AI output preserved and
      retrievable, rejecting an override submitted with no reason (`G-06`)
- [ ] 2.11 Verify divergence between AI output and human decisions is computable from stored
      override records alone
- [ ] 2.12 Implement AI output feedback with categories for inaccurate, incomplete, biased, unsafe
      and not useful, stored against the run and leaving the output unchanged (`AI-011`)
- [ ] 2.13 Carry the correlation identifier onto every run record so an investigation spans audit
      and AI activity together, including across the asynchronous dispatch boundary — the
      identifier `access-control-and-admin`'s audit-review capability fixes and this item consumes
- [ ] 2.14 Implement run log search filterable by family, template version, model version, status,
      safety flag and time range, denying callers without permission
- [ ] 2.15 Verify an Auditor can read run records with governance metadata and no candidate
      personal data in the payload
- [ ] 2.16 Implement retention against configured policy with disposal itself recorded, and check
      the configured value against `D22`'s rule that staleness windows are capped by retention

---

## 3. TS-BL-029 — Advisory-only structural enforcement

**Goal:** the product's central promise — AI recommends, a human decides — is true because no code
path exists by which it could be false, and that is asserted from all three sides at once rather
than three times separately. Covers `ai-platform/advisory-only`.

```yaml
backlog_items:
  - id: TS-BL-029
    feature: ai-platform-governance
    depends_on: [TS-BL-027, TS-BL-004]
    status: not-started
```

**This item completes a guarantee two other features already half-built.** `platform-core`'s
`TS-BL-004` task 4.10 names it explicitly — *"the workflow half of the advisory-only guarantee that
`ai-platform-governance`'s `TS-BL-029` completes"* — and `access-control-and-admin`'s
service-account requirement carries a third half. Read `platform-core`'s `design.md` D7 and D10 and
its workflow-engine spec before starting; this is not a separately-reasoned mechanism (design D6).

- [ ] 3.1 Read `platform-core`'s `platform/workflow-engine` spec and task 4.10, and
      `access-control-and-admin`'s service-account authorization requirement, and record which of
      their scenarios this item must satisfy jointly rather than restate
- [ ] 3.2 Assert the gateway's actor identity is denied the Approve action on every page and cannot
      be granted it through the matrix
- [ ] 3.3 Assert the gateway never assumes a human identity: output is attributed to the AI Service
      Account with the requesting user recorded as requester, never as the actor of a decision
- [ ] 3.4 Enumerate every persistence path from run output and assert none writes a field governed
      by a registered state machine — output lands only in an insight or draft record carrying its
      AI marking
- [ ] 3.5 Implement the human decision record so an action taken *after* AI output names the human
      as decision-maker with their reason, referencing the AI output as input rather than
      authority — including the agreement case, which would otherwise leave no trace of a human
      having decided anything
- [ ] 3.6 Assert a decision taken with no AI involvement produces a record of the same shape with
      no AI reference
- [ ] 3.7 Build the joint advisory-only verification: one pass asserting the workflow-side,
      authorization-side and gateway-side guarantees together, reporting which failed — the
      comparison that three independently-passing suites structurally cannot make (design D6, and
      `platform-core` D4's split-brain lesson)
- [ ] 3.8 Assert the joint verification fails when any single half is weakened: a transition
      accepted without an authenticated permitted actor, Approve becoming grantable to the AI
      Service Account, or a transition capability appearing on the gateway
- [ ] 3.9 Wire the joint verification into CI as a gate that fails when it cannot run, rather than
      skipping (`platform-core` D5's selection rule)

---

## 4. TS-BL-030 — Prompt template registry: ten families, versioned

**Goal:** every AI call in the product resolves to a registered, versioned template whose output
contract fixes both shape and length — and no family can exist without a bound. Covers
`ai-platform/prompt-registry`.

```yaml
backlog_items:
  - id: TS-BL-030
    feature: ai-platform-governance
    depends_on: [TS-BL-027]
    status: not-started
```

**Scope boundary carried from inherited `D12`:** the families are *registered, contracted and
tested* here, not authored. Prompt text is a stub sufficient to exercise the contract, because
writing ten real prompts against schemas Phase 2 will refine guarantees rework — while the contract
and the corpus are exactly what must exist before the first real call.

- [ ] 4.1 Migrate the registry — templates and immutable template versions — with an active version
      per family configurable through the audited runtime-configuration path
- [ ] 4.2 Implement versioning so a content change creates a new version rather than modifying one,
      prior versions stay retrievable, and a past run still names the exact version it used after
      the active version has moved on
- [ ] 4.3 Register all ten families from §16.3 — `job_description_generation`,
      `job_posting_generation`, `resume_extraction`, `candidate_ranking`, `fitment_summary`,
      `gap_summary`, `interview_questions`, `interview_note_summary`, `scorecard_generation`,
      `resurfacing` — each with stub prompt text and a declared output contract
- [ ] 4.4 Reject a run requested for a family not present in the registry
- [ ] 4.5 Implement contract validation in the gateway, recording a violation as a failed run with
      the validation detail preserved and persisting no insight or draft from invalid output
- [ ] 4.6 Implement conciseness bounds **per free-text field**, not per output — a maximum
      sentence, item or character count — validated by the same code that validates shape. The
      per-field decision is load-bearing: §16.3 requires markdown for
      `job_description_generation`, and a single whole-output bound would be either vacuous or
      impossible (design D3, point 3)
- [ ] 4.7 Refuse registration of a family whose contract declares a free-text field carrying no
      bound, so the standard cannot be followed for nine families and forgotten for the tenth
- [ ] 4.8 Attach machine-readable provenance to every bound recording whether its value is a
      confirmed product decision or a provisional default, and make changing a bound an audited
      configuration change rather than a code change (`S.5` is explicit that the numbers were never
      given and should not be invented)
- [ ] 4.9 Add evidence-reference and insufficiency clauses to the contracts of every family
      producing candidate insight (`AI-007`, `AI-008`, `G-02`, `G-03`)
- [ ] 4.10 Add the source-label clause referencing `TS-BL-031`'s vocabulary rather than restating
      it, rejecting output whose label falls outside it. If `TS-BL-031` has not landed, write the
      clause against the declared vocabulary as an interface (design D9)
- [ ] 4.11 Implement the registry side of the promotion gate: refuse to activate a template version
      in production with no passing corpus run recorded for it, and assert it refuses — the gate
      is expected to refuse everything until `TS-BL-032` lands, which is correct (design D8)
- [ ] 4.12 Require a recorded prompt and model version review for any change altering ranking
      behavior (`ENG-008`)
- [ ] 4.13 Verify no end-user screen invokes any family, and that each family is exercisable
      through the evaluation harness and an authorized diagnostic path
- [ ] 4.14 Assert each family's stub template, executed against the stub provider, produces output
      satisfying its own contract including its conciseness bound

---

## 5. TS-BL-031 — Evidence-source labeling substrate

**Goal:** one authority decides whether a claim is resume-sourced, interview-sourced,
scorecard-sourced or a human decision, and a reader can follow a claim to its source only if they
may read that source. Covers `ai-platform/evidence-labeling`.

```yaml
backlog_items:
  - id: TS-BL-031
    feature: ai-platform-governance
    depends_on: [TS-BL-028]
    status: not-started
```

**Why this depends on the run log and not the registry** — the reason evidence labeling exists is
the one `glossary.md` gives: keeping an Administrator's run-log access from becoming a personal-data
back door. That is a property of the run log. The registry's source-label *clause* is `TS-BL-030`'s;
the vocabulary and the reference model are this item's, and there is no hard edge between them
(design D9). Building this item first is preferred but not required.

- [ ] 5.1 Define the closed source vocabulary — resume, interview note, scorecard, human decision —
      as the single authority, and assert no other component defines, extends or derives a source
      value. `design-system`'s label component specifies that it renders the value it is given
      "without interpreting or deriving it", which is only satisfiable if exactly one component
      derives
- [ ] 5.2 Reject a claim carrying a source value outside the vocabulary rather than storing or
      rendering it
- [ ] 5.3 Migrate the evidence-reference model — source value, record identifier, and version where
      the record is versioned — and require every AI-surfaced claim about a candidate to carry a
      label and at least one reference
- [ ] 5.4 Assert a claim resting on a versioned record names the version it was made against, since
      `D12` forks job descriptions on edit and `D18` versions resumes — an unversioned reference
      would resolve to content the claim was never made about
- [ ] 5.5 Implement permission-scoped resolution: the label and reference identifier are returned to
      any authorized reader, and the referenced content is evaluated on that record's own terms and
      denied without permission on it. Build against `access-control-and-admin`'s `TS-BL-018`
      interface if it has not landed
- [ ] 5.6 Assert a denial on resolution discloses nothing about the referenced record's contents
- [ ] 5.7 Assert labels and references carry no personal data — identifier and version only, never
      an excerpt of what they point at — and that a governance surface reading claims returns none
- [ ] 5.8 Implement human decision as a first-class source, so a claim resting on a recorded human
      judgement is distinguishable from one resting on a document, and evidence carried from another
      Application names the record it came from (`D17`/`D21`'s carry-forward)
- [ ] 5.9 Implement per-claim labelling for mixed-source insights, so each claim carries its own
      label and references rather than the insight carrying one for all — what makes `G-02`'s
      stated purpose real, that "an interviewer can see which claims rest on the resume versus an
      earlier interview"
- [ ] 5.10 Verify the vocabulary, reference model and resolution are exercisable on deployment while
      the screens and families consuming them arrive in later features
- [ ] 5.11 Implement insufficiency as the alternative to an unreferenced claim: where no evidence
      supports a conclusion, record what was missing rather than a claim with no reference (`G-03`)

---

## 6. TS-BL-032 — AI evaluation harness: adversarial testing

**Goal:** nothing reaches production untested, and every contract this feature declares is proven
by a case rather than asserted by a document. Covers `ai-platform/ai-evaluation`.

```yaml
backlog_items:
  - id: TS-BL-032
    feature: ai-platform-governance
    depends_on: [TS-BL-030]
    status: not-started
```

**Scope note:** this item carries §27.1's **security suite** as well as its AI Evaluation Tests.
`D.9`'s title names only adversarial testing, but `C-09` adopted both rows of §27.1 in the same
resolution, and splitting them would put prompt-injection cases in one item and prompt-injection
probes in another (design D13).

- [ ] 6.1 Build the versioned corpus structure with results recorded against the specific template
      version and model version that produced them, so two versions are separately retrievable and
      comparable
- [ ] 6.2 Author low-information cases for every candidate-consuming family, asserting an
      insufficiency result rather than fabricated detail
- [ ] 6.3 Author adversarial and prompt-injection cases, asserting the injected instruction does not
      take effect and output stays within contract
- [ ] 6.4 Author conflicting-evidence cases, asserting the conflict is reported rather than silently
      resolved
- [ ] 6.5 Author protected-attribute cases, asserting absence from both output and any scoring
      rationale
- [ ] 6.6 Author evidence-citation cases, asserting every candidate claim carries a resolvable
      reference
- [ ] 6.7 Author conciseness cases per family, asserting declared bounds hold — and fail a family
      that declares a bound no case exercises, since a bound nothing tests is not enforced (design
      D7)
- [ ] 6.8 Implement per-family, per-category coverage enumeration, failing the suite when a family
      is registered with no cases for an applicable category — coverage that is sampled rather than
      enumerated reports green for the families nobody remembered
- [ ] 6.9 Build the security suite: broken access control, file upload attacks, sensitive data
      exposure in responses, logs and run records, and export permission checks
- [ ] 6.10 Wire both suites into CI so failure blocks merge, and make the gate fail rather than skip
      when it cannot execute — no provider, no corpus, or a misconfiguration
- [ ] 6.11 Complete the promotion gate against `TS-BL-030`'s registry side: refuse promotion of a
      template version with no passing corpus run recorded for it
- [ ] 6.12 Implement corpus re-run and result recording when the configured model changes under an
      unchanged template
- [ ] 6.13 Implement corpus re-run when the configured **provider adapter** changes — a passing
      result attributed to one provider is not evidence about another, which is the check that
      makes `OD-003`'s eventual resolution safe to adopt (design D1)
- [ ] 6.14 Document the process for adding a corpus case from a real failure found through output
      feedback or governance review, so the corpus does not decay into the failure modes
      anticipated before the product had users
- [ ] 6.15 Verify no evaluation case collects or processes demographic or protected-class data, and
      record fairness measurement as pending legal and compliance approval — `C-09` reversed only
      the eval-harness non-goal, and `PRV-006`/`OD-010` keep the bias-audit non-goal alive
