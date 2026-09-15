# Matching and Ranking — Backlog

This feature's complete backlog: seven items, `TS-BL-049` through `TS-BL-055`, decomposed in
`exploration-notes.md` D.10. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.

**Nothing here is built, and nothing was inherited.** `talentsphere-wave-1-foundation` has no
`matching/` delta spec, none of its `design.md` D1–D18 was allocated here (`platform-core` D11
accounts for all eighteen elsewhere), Sprint 0 shipped platform-layer code only, and
`KNOWN_ISSUES.md` carries nothing about vectors, embeddings, ranking or matching. Checked rather
than assumed — see `design.md` D13.

**This feature's entry item is genuinely blocked, unlike the last three.** `TS-BL-049` depends on
`candidate-intake`'s `TS-BL-044`, which is not started, and it needs `DOC-008` sections to embed.
`platform-core`, `ai-platform-governance` and `candidate-intake` each had an unblocked first item;
this one does not (`design.md` Migration Plan step 1).

**Read `design.md` D1 before starting `TS-BL-052`, and read it before assuming anything about job
types.** Three statements are true at once and each is easy to invert: the envelope carries
**two references** (`posting_ref` **and** `resume_ref`, never content), **three** registered prompt
families are invoked as three runs, and this feature registers **exactly one job type** — the batch
orchestrator that subscribes to `posting.opened`. No job type is registered for an individual gateway
call, because `ai-platform-governance` D10 already dispatches those. `candidate-intake` named this
class of error as the most likely in a feature of this shape.

**Build `TS-BL-054` against `C-10`, not `D05`.** `C-10` amends `D05` **reversing it**: gaps yes,
score no. `D05`'s per-candidate / no-rank-position rule stands; its "full context upfront" decision
does not. One paragraph of `exploration-notes.md` said otherwise and was corrected during this
feature's propose conversation — see `design.md` D14 and the dated block under `D05`.

**Reuse, do not rebuild, four mechanisms that already exist.** The append-only **disclosure record**
(`ai-platform-governance`'s `ai-platform/ai-run-logging`) is the record every disclosure in this
feature writes to; the **override record** in the same capability is what `RANK-006`/`G-06` uses; the
**closed evidence-source vocabulary** (`ai-platform/evidence-labeling`) is where every source label
comes from; and `candidate-intake`'s **insufficiency vocabulary and shape** is reused verbatim by
`TS-BL-055` rather than re-termed (`design.md` D9, D10).

**Dependency edges pointing outside this feature.** `TS-BL-049` needs `candidate-intake`'s
`TS-BL-044`; `TS-BL-051` needs `ai-platform-governance`'s `TS-BL-030`; `TS-BL-052` needs
`hiring-postings`' `TS-BL-037` and `ai-platform-governance`'s `TS-BL-027`; `TS-BL-053` needs
`design-system`'s `TS-BL-009` and `access-control-and-admin`'s `TS-BL-018`. Six further interfaces are
built against without being `depends_on` edges: `platform-core`'s `TS-BL-006` (dispatch) and
`TS-BL-002` (audited runtime configuration), `access-control-and-admin`'s `TS-BL-020` (durable audit
writer), `ai-platform-governance`'s `TS-BL-028` (run log, disclosure and override records),
`TS-BL-031` (evidence vocabulary) and `TS-BL-032` (corpus), `hiring-postings`' `TS-BL-038`
(`posting.opened`), and `candidate-intake`'s `TS-BL-043` (sections) and `TS-BL-048` (resume version
pinning). Where an interface has not landed, build against its declared shape and say so.

**One production-activation obligation that is not a dependency edge.** The promotion gate refuses to
activate a template version in production with no passing corpus run recorded. `TS-BL-052` authors
three families, so all three need cases in `TS-BL-032`'s corpus before they are live **in
production** — and all three are fully buildable and deployable to Dev without them
(`ai-platform-governance` D8). `AI-015` makes those cases unusually load-bearing here: it names four
adversarial input classes for **ranking templates specifically**.

**Five item-boundary refinements this feature makes**, under D.11's permission, recorded in
`design.md` D12: `TS-BL-049` carries `VEC-005`'s re-index path, `TS-BL-051` carries ranking
staleness, `TS-BL-052` authors three template versions rather than one, `TS-BL-052` and `TS-BL-053`
split `G-01`'s propose/confirm asymmetry, and `TS-BL-053` renders `candidate-intake`'s duplicate
warning payload. **None changes a dependency edge, and none moves work between features.**

**Two questions handed to this feature by name are answered in `design.md`, and one stays open.**
D5 answers whether a resume re-point forces a re-rank (**no** — the entry is marked stale). D6
answers how an unenriched candidate is ranked (**ranked on what exists, insufficiency recorded, never
penalised**). `D23a`'s follow-on application-merge rule stays open: the ranking half is now defined,
the surviving-application half is `interview-pipeline`'s. Do not invent it.

**Three things are deliberately unresolved and must not be decided during apply.** §29 item 8's
ranking criteria weights carry **no product approval** — ship the key and provisional provenance, not
tuned values. The three families' conciseness bounds are provisional per `S.5`. `OD-003`'s generation
provider is open, so every AI path here runs on the stub in Dev.

---

## 1. TS-BL-049 — Vector embedding pipeline

**Goal:** the traceable sections `candidate-intake` already produces become vector records that trace
back to them — source-identified, model-pinned, filterable, and re-indexable on the day the model or
the parser changes rather than in a later migration. Covers `matching/resume-embedding`.

```yaml
backlog_items:
  - id: TS-BL-049
    feature: matching-and-ranking
    depends_on: [TS-BL-044]
    status: not-started
```

**Read `design.md` D2 and D3 before starting.** D2 fixes which of `VEC-001`'s four source types are
embedded here and why the other two are registrations rather than modifications; D3 fixes that
`authorization_scope_json` carries decision *inputs* and never a verdict. Both are load-bearing for
`TS-BL-050` and for `access-control-and-admin`'s already-written requirement that named this feature
as its consumer.

- [ ] 1.1 Provision the **Vertex AI Vector Search** index and endpoint by Terraform in the Dev root,
      with UAT and Prod written and validated but never applied (`D09`, `D15` as amended). The
      landing zone already enables `aiplatform.googleapis.com` per app — consume that, do not add
      service enablement
- [ ] 1.2 Migrate `vector_index_records` by reversible migration with §12.2's columns —
      `source_type`, `source_id`, `source_version`, `candidate_id`, `job_posting_id`, `practice`,
      `chunk_text_hash`, `embedding_model`, `vector_ref`, `authorization_scope_json` — with the
      `source_type` enum carrying **all four** `VEC-001` values from the first migration so Phase 3
      adds a registration and no schema change (`design.md` D2)
- [ ] 1.3 Implement the **source-type registration seam**: a source type declares its
      record-producing shape and its filter attributes, and register the two that exist —
      `resume_section` and `candidate_skill_summary`. Assert a request for an unregistered type is
      refused rather than embedded under an approximate type
- [ ] 1.4 Add the test that fails if this capability contains document segmentation logic — embedded
      units are `candidate-intake`'s `DOC-008` sections, and a second chunker would silently diverge
      from the sections evidence references point at (`design.md` D2)
- [ ] 1.5 Implement `VEC-002`'s traceback: every record names the relational record and
      object-storage reference it came from, that reference resolves, and the record is marked
      **derived data** per `VEC-006` so it never shadows its source
- [ ] 1.6 Resolve the **embedding model from configuration**, record it on every record, and verify
      records produced by two different models coexist and stay distinguishable (`C-01`'s
      `embedding_model` pinning; `VEC-005`'s model-change trigger is undetectable without this)
- [ ] 1.7 Populate `authorization_scope_json` with the **inputs** to an authorization decision —
      candidate id, posting id, practice — and add the test that fails if any resolved allow or deny
      outcome is written into it (`access-control-and-admin`'s never-store-a-verdict requirement,
      Interaction A, `design.md` D3)
- [ ] 1.8 Register **embedding generation as a job type** against `platform-core`'s dispatch pattern,
      triggered by parsed content, declaring §25's retry policy (retry on transient vector or
      embedding failure) and carrying the correlation identifier through. Build against `TS-BL-006`'s
      interface if it has not landed
- [ ] 1.9 Implement idempotency at the effect boundary: the same embedding event delivered twice
      leaves one record set for that source and version (`platform-core` D6 — delivery is
      at-least-once)
- [ ] 1.10 Add the test that fails if any code path in this feature starts embedding work outside the
      shared dispatch pattern
- [ ] 1.11 Verify exhausted retries leave the source record intact with its failure reason visible
      rather than dropping it, and that the candidate remains fully readable throughout
      (`platform-core`'s exhausted-work requirement, `G-12`)
- [ ] 1.12 Implement the **`VEC-005` re-index path** with the pipeline, not after it: identify
      affected records for a model change, a parsing-logic change and a source-content change, and
      re-index in batches of §29 item 15's configured size, reporting progress. Superseded records are
      **replaced, not duplicated**, and source records are never touched (`design.md` D12 — this ships
      here because a re-index added later is a migration, not an invariant)
- [ ] 1.13 Make re-indexing audited and re-runnable, and assert an authorization or permission change
      requires and triggers **no** re-index — the fourth `VEC-005` trigger designed out by D3
- [ ] 1.14 Assert the three refusals at the embedding boundary: no document without a clean malware
      verdict, no resume unresolved to a candidate, and **no structured contact field** in embedded
      content. Leave a comment citing `candidate-intake` D10 — "a contact field inside a section
      becomes a contact field inside a similarity index … which no downstream filter can undo"
- [ ] 1.15 Implement `RET-007`'s source linkage from the vector side: every record discoverable from
      the resume version it was derived from, so disposal of a source can reach everything derived
      from it, and the disposal is recorded
- [ ] 1.16 Seed §29 item 15 (re-index batch size) and the embedding model identifier into
      `platform-core`'s audited runtime-configuration registry, and assert no unaudited setter exists
      for either
- [ ] 1.17 Ship the pipeline behind a declared feature flag, disabled by default — a disabled
      capability answers 404

---

## 2. TS-BL-050 — Vertex AI Vector Search integration

**Goal:** retrieval stage one returns the right evidence to the right caller, evaluates authorization
at query time so a permission change is effective on the next request, and never becomes a source of
truth or a scoring signal. Covers `matching/vector-retrieval`.

```yaml
backlog_items:
  - id: TS-BL-050
    feature: matching-and-ranking
    depends_on: [TS-BL-049]
    status: not-started
```

**Read `design.md` D2 and D3, and `access-control-and-admin`'s `access-control/authorization` spec,
before starting.** That spec's "Authorization decisions are never stored as resolved verdicts" and
"Scope is applied as a query filter, not by discarding fetched rows" requirements both **name this
item as their consumer**. `access-control-and-admin` D4 records that getting this wrong here is the
expensive end of the trade.

- [ ] 2.1 Build the **filter helper first**, taking the evaluator's scope filter as a required
      argument, and route every retrieval path through it — D3's trade-off is that nothing in the type
      system enforces filter application, and this is the mitigation
      `access-control-and-admin` D4 already committed to
- [ ] 2.2 Implement filtered kNN over the index supporting `VEC-004`'s filters — candidate id,
      posting id, practice, source type, source version — applied **on the index**, not by
      post-filtering results
- [ ] 2.3 Resolve `VEC-003`'s authorization through the **central permission evaluator** and apply
      the returned filter as a query-time predicate. Assert the verdict never comes from logic local
      to this capability. Build against `TS-BL-018`'s interface if it has not landed
- [ ] 2.4 Add the test that fails if a retrieval path is exercised without applying the evaluator's
      scope filter
- [ ] 2.5 Verify a permission change is effective on the **next retrieval with no re-index**: add a
      deny override and confirm exclusion, widen a role's read scope and confirm inclusion, and assert
      the enumerated re-index triggers do not include authorization change
- [ ] 2.6 Verify the reported result count reflects only in-scope records, disclosing nothing about
      records outside scope (`access-control-and-admin`'s count-leak reasoning, which over a kNN set
      also silently shortens the evidence a ranking run sees)
- [ ] 2.7 Implement `VEC-007`'s minimum-chunk retrieval with a configured bound, and make a request
      that would exceed the bound refuse or **report its truncation** rather than returning a partial
      set as complete
- [ ] 2.8 Return `VEC-008`'s source labels from `TS-BL-031`'s closed vocabulary, and assert this
      capability derives no source value of its own. Build against the declared vocabulary if it has
      not landed
- [ ] 2.9 Assert retrieval returns **no score and no ranking**, and add the test that fails if a
      similarity distance reaches a ranking signal or a persisted score component (`C-01` — vectors
      retrieve, the LLM ranks; `design.md` D2)
- [ ] 2.10 Implement resolution of a retrieved item to its authoritative relational or object-storage
      record, and assert no consumer reads a value from the index as authoritative (§12 as cited by
      `C-01`)
- [ ] 2.11 Implement `G-12`/`NFR-004` degradation: with the backend unavailable, existing rankings,
      boards, candidate records and every non-AI action stay fully usable, and a ranking request
      **records a failed run rather than proceeding on an empty evidence set** — the specific failure
      that would otherwise produce an evidence-free ranking that looks complete
- [ ] 2.12 Require the applicable action on the invoking surface, evaluated server-side, and **deny**
      an unpermitted caller rather than returning an empty result. Verify a denial and an empty
      permitted result are distinguishable (`AUTHZ-004`, `API-002`)
- [ ] 2.13 Emit §26's retrieval metrics — call count, latency, failure rate — through OpenTelemetry
      alongside the ranking-duration metric §26 already names
- [ ] 2.14 Verify retrieval behaviour by configuration in Dev rather than by waiting for a real
      outage: backend unreachable, backend slow past its timeout, and empty index

---

## 3. TS-BL-051 — RankingScore tuple model

**Goal:** a score stays explicable for as long as the record exists — the full tuple on every entry,
prior versions preserved rather than overwritten, and an entry whose inputs have moved on that says
so instead of looking current. Covers `matching/ranking-score`.

```yaml
backlog_items:
  - id: TS-BL-051
    feature: matching-and-ranking
    depends_on: [TS-BL-050, TS-BL-030]
    status: not-started
```

**Read `design.md` D4 and D5 before starting.** D4 fixes that this is a versioned table and that
§12.2's `latest_rank`/`latest_match_score` are a single-writer pointer at the active version — the
tempting implementation adds four columns to `candidate_posting_applications` and cannot satisfy
`G-07`. D5 fixes what marks an entry stale and what re-ranks, and it is the answer this feature owes
`candidate-intake`.

- [ ] 3.1 Migrate the ranking version table and its per-application entries by reversible migration,
      each entry carrying `resume_version`, `jd_version`, `prompt_template_version`, `model_version`
      and the AI run reference
- [ ] 3.2 Reject creation of an entry with **any tuple term absent**, rather than storing a null term
      (`domain-model.md`'s provenance rule; `ai-platform-governance`'s reproducibility-tuple
      requirement, of which this is the first real consumer)
- [ ] 3.3 Verify each of the four recorded versions resolves to the content it names, including after
      later versions of the resume and the job description exist
- [ ] 3.4 Implement `G-07`/`RANK-005` versioning: a re-rank creates a **new** version and prior
      versions and their entries remain readable unchanged. Reject any write against an existing
      entry's score
- [ ] 3.5 Implement exactly one active version per posting, and make the application's `latest_rank`
      and `latest_match_score` **derived from activation** with a single writer. Add the test that
      fails if any other code path writes either column (`design.md` D4)
- [ ] 3.6 Verify a never-ranked posting's applications carry no rank and no score, and that this state
      is distinguishable from a score of zero
- [ ] 3.7 Implement **staleness**: an entry whose recorded resume version or ranking-relevant posting
      input differs from the application's current one is marked stale, its score unchanged, with the
      version it was computed against named. Assert staleness deletes nothing, recomputes nothing and
      triggers nothing (`design.md` D5)
- [ ] 3.8 Verify the two cases earlier features deferred here: an explicit resume re-point marks the
      entry stale and forces no re-rank, and a candidate merge leaving an application on a superseded
      input marks the entry stale and merges, deletes and recomputes no ranking
      (`candidate-intake`'s `TS-BL-048` task 8.9 and its `design.md` D1(b))
- [ ] 3.9 Verify a re-rank produces non-stale entries in the new version while the prior stale entries
      remain readable
- [ ] 3.10 Persist entries as **advisory insight** carrying their AI-generated marking, clear the
      marking only on a recorded human approval for that entry, and assert no persistence path from a
      ranking entry reaches a workflow-governed state field (`AI-001`, `AI-010`, `BR-007`, and
      `ai-platform-governance`'s advisory-only capability)
- [ ] 3.11 Implement retention of ranking versions against the configured policy with disposal itself
      recorded, and verify disposing a ranking version does not orphan the AI run records it
      referenced (`OD-004` supplies the interval; do not invent one)
- [ ] 3.12 Verify a ranking computed against resume version one remains attributable to that version
      after version two exists, and that version one's content is still retrievable — the property
      `candidate-intake`'s `TS-BL-048` task 8.8 asserts from the other end
- [ ] 3.13 Write every ranking activation and every human approval through the audit path in the same
      transaction, carrying references and never candidate personal data, and assert a failed audit
      write fails the operation. Build against `TS-BL-020`'s durable writer

---

## 4. TS-BL-052 — AI Ranking Engine

**Goal:** a posting and a resume become a score, a fitment summary and a gap summary — through the one
governed egress, on two references rather than content, from role-relevant signals only, and with no
path by which the model can assert anything that blocks a person. Covers `matching/ranking-engine`.

```yaml
backlog_items:
  - id: TS-BL-052
    feature: matching-and-ranking
    depends_on: [TS-BL-037, TS-BL-051, TS-BL-027]
    status: not-started
```

**Read `ai-platform-governance`'s `design.md` D3, D7 and D10 and its `ai-platform/ai-gateway` spec,
plus `design.md` D1 and D11 here, before starting.** D3 fixed this envelope and named this call in
advance — *"`candidate_ranking` sends posting and resume references"* — so the envelope is not
negotiable and the payload inside it is this item's. D10 means **no job type is registered for a
gateway call**; D1 here means **exactly one is registered** for the batch orchestrator, because
`hiring-postings` published `posting.opened` with no subscriber. Both are true at once.

**Three families, not one.** §16.1 lists Candidate Ranking, Fitment Summary and Gap Summary as three
capabilities; §16.3 registers three families with three output formats. Authoring one combined prompt
would collapse three registry entries, three model configurations, three bound sets and three corpus
obligations into one (`design.md` D1).

- [ ] 4.1 Author `candidate_ranking` as a **new template version** of the family `TS-BL-030`
      registered, never by editing an existing version. Depends on the registry; build against its
      declared interface if it has not landed
- [ ] 4.2 Build the request inside the fixed envelope with **`posting_ref` and `resume_ref` as
      identifier-and-version**, and assert the payload carries no job description text, no posting
      text, no resume text and no candidate personal data. Reject a request supplying only one of the
      two references rather than running against a partial input
- [ ] 4.3 Verify the references recorded on a completed ranking still resolve to the versions the run
      consumed after a later resume version or job description version exists
- [ ] 4.4 Author `fitment_summary` and `gap_summary` as new template versions of their registered
      families, and assert **three runs** exist per application per ranking version, each naming its
      own template and model version. Add the test that fails if any path produces output for more
      than one registered family from a single run
- [ ] 4.5 Verify per-family model configuration is honoured — two of the three configured with
      different models produce runs recording different models
- [ ] 4.6 Assert this capability contains **no outbound call to a model provider**, and that the job
      types it registers are exactly one: the batch ranking orchestrator (`design.md` D1)
- [ ] 4.7 Register the orchestrator as a **subscriber to `posting.opened`** through `platform-core`'s
      dispatch pattern, declaring §25's Candidate Ranking retry policy and carrying the correlation
      identifier. `hiring-postings`' `TS-BL-038` publishes and registers no subscriber; this is it
- [ ] 4.8 Implement idempotency at the effect boundary: the same ranking request delivered twice
      issues one run per family and produces one entry
- [ ] 4.9 Implement the three triggers and **only** the three: posting open, authorized manual
      trigger for all candidates linked to a posting (`RANK-001`), and a change to a
      ranking-relevant posting requirement field (§25's "requirement change", which resolves to the
      posting's own fields because an open posting's `jd_version` is immutable). Add the test that
      fails if a resume re-point or a candidate merge triggers a ranking (`design.md` D5)
- [ ] 4.10 Verify a posting whose `posting.opened` event was never delivered is still open and still
      rankable by manual trigger (`hiring-postings` D7's degradation property, from the consumer side)
- [ ] 4.11 Enforce §16.4's signal lists as a **contract**: every cited criterion is one of the eleven
      permitted, and output citing any of the ten prohibited attributes is rejected as a contract
      violation and recorded as a failed run (`AI-005`)
- [ ] 4.12 Verify protected-attribute redaction applies to scoring input before the prompt is
      composed and that the run records the redaction — this is the first family in the product whose
      input is destined for scoring, so the gateway clause first fires here
- [ ] 4.13 Assert the structured email, phone and name fields are absent from what the resolved
      references supply, and that **`AI-006`'s controlled exception is not invoked by any of the three
      families** (`PRV-001`, `PRV-002`, `CAN-008`)
- [ ] 4.14 Mark resume-derived content as **untrusted document-derived content** at the boundary, and
      verify a resume containing text shaped as an instruction to raise its own score alters neither
      the family, the caller's permissions, the criteria used, nor the output contract (`AI-012`,
      `SEC-014`; `D18` names ranking as the larger injection surface)
- [ ] 4.15 Declare the output contracts: every fitment claim, gap claim and contributing criterion
      carries a **source label** from `TS-BL-031`'s closed vocabulary and at least one **versioned
      evidence reference**. Reject output where a claim lacks either, and verify a mixed-source
      insight carries per-claim rather than per-insight labels (`G-02`, `AI-007`, `RANK-002`)
- [ ] 4.16 Implement `RANK-003`'s criteria explanations: the criteria contributing to a score are
      listed, each with its own evidence
- [ ] 4.17 Implement the four gap categories — missing, weak, unclear, conflicting — each with an
      evidence status, and reject a category outside the four (§16.1, §16.3's `gap_summary` contract)
- [ ] 4.18 Implement `G-01`'s asymmetry on the AI side: output may propose
      `meets | unclear | manager_review_required`, `does_not_meet` is **rejected as a contract
      violation**, and the proposed value is stored marked AI-proposed. Add the test that fails if any
      path from AI output to a stored `does_not_meet` exists — `G-01`'s named downstream reader is
      `BR-015`'s priority lane (`design.md` D11)
- [ ] 4.19 Implement `design.md` D6: an unenriched candidate is **ranked** on the evidence that
      exists, appears in the ranking version, and carries insufficiency for what enrichment would have
      supplied. Add the test that compares the version's entries against the posting's linked
      applications and fails on any silent omission
- [ ] 4.20 Declare a **bound on every free-text field** of all three contracts, provisional and
      labelled provisional in audited configuration; record a failed run with the violation preserved
      and persist no content when a bound is exceeded. Verify a gap summary is a bounded short list
      rather than prose (`S.5` names this surface as the highest-volume AI surface in the product;
      `ai-platform-governance` D7)
- [ ] 4.21 Implement `ERR-007`'s candidate-level partial-failure reporting: per-candidate outcomes,
      the successful entries still forming a ranking version that records which candidates failed, and
      a run reference retrievable for every failure
- [ ] 4.22 Implement asynchronous batch execution with retrievable progress by identifier (§32's
      Ranking row, `NFR-002`)
- [ ] 4.23 Read ranking criteria weights from `platform-core`'s audited runtime-configuration
      registry with provenance marking them **provisional and unapproved**, and verify a change is
      audited with previous and new value and a mandatory reason. §29 item 8 says "where product
      approved" and no approval exists — do not supply tuned values
- [ ] 4.24 Require the **Run AI** action on the ranking surface, distinct from view and edit, taken
      from the central evaluator, with the control absent for users who lack it and a
      directly-submitted request refused server-side (§9.3, `C-02`, `AUTHZ-004`)
- [ ] 4.25 Verify a background orchestrator run under the **AI Service Account** has its permission
      evaluated by the same evaluator on the same terms as a human caller, and that `Approve` remains
      ungrantable to it
- [ ] 4.26 Verify graceful degradation with the provider unavailable and with the vector backend
      unavailable: existing versions and boards render, non-AI actions work, and the trigger control
      reports temporary unavailability (`G-12`, `NFR-004`)
- [ ] 4.27 Add all three families' cases to `TS-BL-032`'s corpus — contract conformance, conciseness
      per bounded field, and `AI-015`'s four named classes: adverse examples, low-information resumes,
      **adversarial resumes**, and conflicting interview notes — and record that production activation
      is refused for each family until they pass (`C-09`, §36, `ai-platform-governance` D8)
- [ ] 4.28 Record in `KNOWN_ISSUES.md` that ranking runs against the stub provider in Dev pending
      `OD-003`, and that §29 item 8's weights are provisional and unapproved — the same treatment the
      unconfirmed Hubble contract already receives. **This does not duplicate
      `ai-platform-governance`'s task 1.18:** that task records `OD-003` itself, and this one records
      that this feature's three families are affected by it. If 1.18 has already landed, extend its
      entry rather than adding a second (`design.md` D13)

---

## 5. TS-BL-053 — Ranking Board UI

**Goal:** a Practice Manager sees a posting's candidates with every claim traceable to its evidence,
the recommendation labelled as advisory on its face, and their own divergence recorded beside the AI
output rather than replacing it — and the board decides nothing. Covers `matching/ranking-board`.

```yaml
backlog_items:
  - id: TS-BL-053
    feature: matching-and-ranking
    depends_on: [TS-BL-052, TS-BL-009, TS-BL-018]
    status: not-started
```

**Read `design.md` D8 and D11 before starting.** D8 fixes the payload split that keeps comparative
data off the per-candidate path — the one place `D05`'s still-standing per-candidate rule constrains
this board even though its own audience sees rank. D11 fixes that the human half of `G-01` lands
here. **`D.10`'s dependency-translation note applies:** this item's old edge to the Admin Cockpit did
not carry forward; what it actually needs is `TS-BL-009`'s dense data table and `TS-BL-018`'s
evaluator.

- [ ] 5.1 Build the board from `design-system`'s **dense data table**, page templates, AI-disclosure
      marking and evidence-source label components, adding **no new component**, no raw color and no
      one-off spacing value. The data table's Purpose already names the ranking board as one of the
      three screens it exists for
- [ ] 5.2 Render `RANK-002`'s full row — rank, match score, mandatory-criteria status, fitment, gap,
      evidence references and the **AI run version** — with the run version on the row rather than
      behind a navigation
- [ ] 5.3 Render an explicit **not-yet-ranked** state inside the table container for a posting never
      ranked, distinct from an empty grid and from a zero score, and render loading and empty states
      inside the container per the table's contract
- [ ] 5.4 Render **staleness** on the row, naming the version the entry was computed against
      (`design.md` D5)
- [ ] 5.5 Verify horizontal overflow is contained by the table and the page body never scrolls
      horizontally
- [ ] 5.6 Gate access through the central evaluator as a PM / RM / permitted-Recruiter surface (§8,
      `C-10`), resolving Recruiter access through the configured read scope and the
      `assigned-postings` predicate, applied as a **query predicate** rather than by discarding
      fetched rows. Assert an **Interviewer's direct request is denied server-side**
- [ ] 5.7 Render permission-aware affordances as **absent** rather than present and disabled, and
      verify each is refused server-side when requested directly (`UI-002`, `AUTHZ-004`, `SEC-005`)
- [ ] 5.8 Render `RANK-004`/`UI-006`'s human-review disclaimer visibly without interaction, and add
      the test that fails if any board text asserts bias-free recommendations. `PRV-006` keeps bias
      auditing out of scope, which is why the disclaimer carries weight here
- [ ] 5.9 Render the AI-generated marking on unapproved fitment, gap and score content, cleared only
      by a recorded human approval (`AI-001`, `UI-004`)
- [ ] 5.10 Implement `RANK-006`/`G-06`/`BR-016`'s override through
      `POST /api/applications/{applicationId}/ai/override` (§13.2), using
      `ai-platform-governance`'s **existing override record** rather than a second mechanism:
      mandatory reason, actor, timestamp, original output reference and human value, with the original
      output still retrievable. Reject an override with no reason
- [ ] 5.11 Verify divergence is measurable — the rate at which humans diverged from AI ranking output
      is computable from stored override records alone, which is what makes
      `insight-and-reporting`'s AI-agreement-rate dashboard possible
- [ ] 5.12 Implement the human half of `G-01`: setting `mandatory_criteria_status` to
      `does_not_meet` is a permission-gated action on this surface requiring a **mandatory reason**,
      audited with previous and new value, with the AI-proposed value still visible beside it. Reject
      it without a reason and deny an unpermitted actor (`design.md` D11)
- [ ] 5.13 Implement `GET /api/postings/{postingId}/ai/rankings` (§13.2) as the **posting-scoped**
      response carrying rank, score and comparative data, with version selection so an authorized user
      can open a prior ranking version as it stood with its own tuple values (`G-07`)
- [ ] 5.14 Implement `GET /api/applications/{applicationId}/ai/insights` (§13.2) as the
      **application-scoped** response carrying fitment, gaps, evidence and mandatory-criteria status
      and **no rank, no score and no cohort size**. Add the test that fails if any value from which
      cohort size or standing could be derived — a percentile, a quartile flag, a normalized score —
      appears in it (`design.md` D8)
- [ ] 5.15 Build the candidate-detail view from the application-scoped payload plus the board row it
      was opened from, rather than from a richer per-candidate payload later trimmed
      (`design.md` D8)
- [ ] 5.16 Render `candidate-intake`'s **duplicate warning** as a non-blocking advisory marker naming
      the counterpart, consuming the payload that feature's `TS-BL-047` task 7.10 supplies. Verify the
      board renders normally with no placeholder when no payload is supplied, and that a warning
      changes no row's rank, score or available actions
- [ ] 5.17 Add the test that fails if this capability writes an application's `status`,
      `shortlist_reason` or `final_outcome`, or triggers any workflow transition — a board showing a
      score is the most natural place for someone to add a shortlist button, and shortlisting is
      `interview-pipeline`'s `TS-BL-056`
- [ ] 5.18 Implement export as offered only against a **supplied affirmative decision**, refused
      server-side without the Export action, and audited on completion carrying references and no
      candidate personal data (`UI-008`, `API-003`; the table evaluates no permission itself)
- [ ] 5.19 Render evidence source labels through `design-system`'s label component from the closed
      vocabulary, deriving no source value on this surface
- [ ] 5.20 Ship the board and its endpoints behind a declared feature flag, disabled by default,
      answering 404 while disabled, and verify an undeclared flag name raises rather than resolving
      false

---

## 6. TS-BL-054 — AI context visibility rules for Interviewers

**Goal:** an interviewing actor gets what helps them probe — source-labelled fitment and gaps, per
candidate — and cannot reach the score, the rank, or anyone else, with every disclosure leaving a
record that answers whether their judgement was formed before or after they saw it. Covers
`matching/ai-context-visibility`.

```yaml
backlog_items:
  - id: TS-BL-054
    feature: matching-and-ranking
    depends_on: [TS-BL-053]
    status: not-started
```

**Read `design.md` D7, D8 and D9, `C-10` in full, and `D05`'s two dated correction blocks before
starting.** `C-10` **amends `D05`, reversing it** — build "gaps yes, score no," not "full context
upfront." One paragraph of `exploration-notes.md` asserted the opposite and was corrected during this
propose conversation (`design.md` D14). D7 fixes that this item is a **server-side projection, not a
screen**: the screen is `interview-pipeline`'s Interview Console, unproposed. D9 fixes that
disclosures go to `ai-platform-governance`'s existing append-only record and that **no second
disclosure mechanism is built**.

- [ ] 6.1 Implement the projected response: source-labelled fitment and gap summaries plus the job
      description and resume, with **no match score, no rank position and no cohort size**
      (`INT-005`, `C-10`)
- [ ] 6.2 Assert the projected response contains **no reference to any other candidate** on the
      posting
- [ ] 6.3 Produce the projection **server-side** so withheld values are absent from the payload on
      the wire rather than hidden at the presentation layer, and verify a client requesting the
      unprojected payload still receives the projection (`AUTHZ-003`, `UI-002`, `config.yaml`'s
      write-time-not-display-filter rule)
- [ ] 6.4 Add the test that fails if this guarantee rests on any display-time filter
- [ ] 6.5 Key the projection to the **evaluator's verdict on the ranking surface**, not to a role
      name, and add the test that fails if a role-name comparison decides it (`design.md` D7)
- [ ] 6.6 Verify all three matrix cases without a code change: an actor holding the seeded Interviewer
      permissions receives the projection; a user granted the board action receives the full response
      and the grant is audited; and a user holding the board action by role with a **direct denial**
      recorded receives the projection (`AUTHZ-005`, `ADM-005`, `INT-003`'s "unless broader permission
      is granted")
- [ ] 6.7 Apply assignment scope as a **query predicate** so an interviewing actor receives context
      only for candidates they are assigned to, **denying** an unassigned request rather than returning
      empty, and verify a denial and an empty permitted result are distinguishable (`INT-003`, `D01`,
      `C-03`)
- [ ] 6.8 Reject **comparative claims** from the projection — any fitment or gap claim stating or
      implying a comparison with other candidates, a cohort, or a ranked position — and verify an
      absolute claim about the candidate's evidence against the posting's requirements is included
      (`D05`'s still-standing per-candidate rule; `design.md` D8, which explains why withholding the
      score alone is insufficient)
- [ ] 6.9 Add the test that scans projected content for ranked-position phrasing and fails on a match
- [ ] 6.10 Write a **disclosure record** through `ai-platform-governance`'s existing append-only
      record on each disclosure — actor, output, time, context — with the AI run **unchanged**, and add
      the test that fails if a disclosure store owned by this capability exists
- [ ] 6.11 Implement `design.md` D9's disclosure granularity: **one record per retrieval of AI output
      by an actor**, not per page render and not per session. Verify a board re-render on sort, filter
      or pagination does not multiply records, and that a session which never opened a candidate
      discloses nothing about that candidate
- [ ] 6.12 Record a context value that distinguishes **pre-interview** disclosure from board
      disclosure, so `D05`'s question is answerable from the record rather than inferred
- [ ] 6.13 Verify a reviewer can answer "was this judgement formed before or after seeing AI output"
      from disclosure records alone, for both the projected and the board case
- [ ] 6.14 Add the assertion that makes `C-10`'s agreement-rate consequence checkable: **no disclosure
      record names a match score or rank position disclosed to an interviewing actor in a
      pre-interview context**, and a code path that would produce one fails the suite
- [ ] 6.15 Carry each projected claim's source label and evidence reference so a reader can see which
      claims rest on the resume and which on an earlier interview or scorecard, with reference
      resolution remaining permission-scoped — label and identifier always returned, content denied
      without permission on the referenced record (`G-02`, `INT-005`, `BR-008`)
- [ ] 6.16 Verify carried evidence is visible as carried: a claim resting on a record from a different
      Application names that record (`D17`/`D21`'s priority-lane carry-forward, and
      `ai-platform-governance`'s carried-evidence scenario)
- [ ] 6.17 Assert this item builds **no** Interview Console, structured note template, interview
      question generation or interview state — all of which are `interview-pipeline`'s `TS-BL-058`
      onward. Leave a comment citing `design.md` D7 so the absence reads as a boundary rather than as
      unfinished work
- [ ] 6.18 Publish the projection as a documented payload contract an interview surface consumes,
      mirroring how `candidate-intake` kept its duplicate warning to a payload this feature's board
      renders
- [ ] 6.19 Ship the projection endpoint behind a declared feature flag, disabled by default,
      answering 404 while disabled

---

## 7. TS-BL-055 — Insufficiency outputs

**Goal:** when the evidence does not support a conclusion the product says so, and — the consequence
that only exists once there is a score — the absence is never quietly turned into a low number.
Covers `matching/ranking-insufficiency`.

```yaml
backlog_items:
  - id: TS-BL-055
    feature: matching-and-ranking
    depends_on: [TS-BL-052]
    status: not-started
```

**Read `design.md` D10 and `candidate-intake`'s `candidate/resume-enrichment` spec before starting.**
That spec already established `G-03`'s shape concretely — a requirement titled *"Missing evidence
produces an insufficiency marker, not a value"* with the scenario that an insufficient field is
distinguishable from one never requested. **Reuse that vocabulary and shape verbatim**; do not invent
parallel terms for the same idea across two AI stages a Practice Manager reads side by side. What
ranking adds is a **third** state — insufficient distinguishable from *evaluated as weak* — and it is
where `G-03`'s trust argument actually lands.

**Build this before `TS-BL-053`'s insufficiency display where the order is free** (`design.md`
Migration Plan step 6): the board renders insufficiency, and taking the other order means building
that display twice.

- [ ] 7.1 Implement insufficiency markers for a ranking criterion, a fitment claim or a gap
      classification the evidence does not support, each naming what it applies to and **what evidence
      was absent**, rather than an inferred value (`G-03`, `AI-008`, `DOC-009`)
- [ ] 7.2 Reject as a contract violation any output assigning a criterion a value with no evidence
      reference, recorded as a failed run
- [ ] 7.3 Make **three** states distinguishable on a ranking entry — insufficient, not evaluated, and
      evaluated as weak — and verify a criterion outside this posting's evaluation reads as not
      evaluated rather than as insufficient
- [ ] 7.4 Implement the rule this item exists for: an insufficiency marker makes **no negative
      contribution to the score**. Add the test that inspects the scoring path and fails if one
      exists (`design.md` D10)
- [ ] 7.5 Implement a **withheld or explicitly qualified** score for a candidate whose evidence
      supports few criteria, listing the insufficient criteria, and verify a weak-evidence candidate
      and an absent-evidence candidate produce distinguishable entries with the latter not scored
      lower for the absence
- [ ] 7.6 Implement `G-01`'s specific case: unevidenced mandatory criteria resolve to
      `unclear` or `manager_review_required`, never `does_not_meet`, and output proposing a block on
      the basis of absent evidence is rejected as a contract violation. Absent and disqualifying
      evidence look alike to a model, which is why this case is asserted separately from
      `TS-BL-052`'s general prohibition
- [ ] 7.7 Verify the human resolution path on the ranking board is available for an `unclear` or
      `manager_review_required` status and that its use is audited
- [ ] 7.8 Persist insufficiency as its own insight type using §12.2's existing
      `ai_insights.insight_type` `insufficiency` value, linked to its run and its ranking entry, and
      retrievable with the entry
- [ ] 7.9 Render insufficiency **alongside the affected claim or score** on the board rather than only
      in a detail view — an insufficiency stored and not shown is a guess from the reader's point of
      view (`RANK-002`, `UI-003`)
- [ ] 7.10 Include insufficiency affecting projected fitment or gap claims in `TS-BL-054`'s
      projection, so an interviewing actor sees "we could not tell" rather than a silently thinner
      summary
- [ ] 7.11 Declare a bound on every free-text field of insufficiency output, and reject a reason that
      **asserts a fact about the candidate** rather than describing the absent evidence — a
      speculative reason is itself an unsupported claim under `AI-007` (`S.5`,
      `ai-platform-governance` D7)
- [ ] 7.12 Add a **low-information case** to `TS-BL-032`'s corpus for each of the three ranking
      families, asserting insufficiency rather than an inferred value, and verify production
      activation is refused for a family with no passing insufficiency case (`AI-015`'s named
      "low-information resumes" class, `C-09`, `ai-platform-governance` D8). Without such a case the
      insufficiency path is code that never ran
