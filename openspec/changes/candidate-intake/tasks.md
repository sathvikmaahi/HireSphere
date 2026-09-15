# Candidate Intake — Resumes, Identity and Deduplication — Backlog

This feature's complete backlog: eight items, `TS-BL-041` through `TS-BL-048`, decomposed in
`exploration-notes.md` D.10. Each is independently deployable to dev, uat and prod.

**Grouped by backlog item, not by sprint or wave.** Per D.11, all twelve features are proposed
*before* the sprint/wave schedule is redone, in its own conversation against `delivery/`.

**Nothing here is built, and nothing was inherited.** Sprint 0 of `talentsphere-wave-1-foundation`
shipped platform-layer code only — its handover states "no hiring feature exists yet, by design."
That change has no `candidate/` delta spec, no `D1`–`D18` decision from its `design.md` was
allocated here, and `KNOWN_ISSUES.md` carries nothing about resumes, parsing, candidates or object
storage. See `design.md` D12 for the check rather than the assumption.

**One decision this feature rests on is explicitly unconfirmed, and it must not be treated as
settled during apply.** `D23a`'s status line reads *"recommendation made, not yet confirmed"* — the
only such citation in this backlog. `design.md` D1 makes the call in two parts: its **two-check
recommendation is treated as confirmed** on four stated grounds and `TS-BL-047` builds it behind its
own flag; its **follow-on application-merge rule is treated as genuinely open** and is excluded from
scope. Do not invent a rule for which application survives a merge, which ranking is retained, or
what happens at differing stages — two of those three inputs belong to unproposed features. Read
`design.md` D1 before starting `TS-BL-047`.

**Async dispatch is not built twice, and it is not skipped either.** Two non-AI job types register
against `platform-core`'s `TS-BL-006`: the **malware scan** (`TS-BL-041`) and **deterministic
extraction** (`TS-BL-043`) — these are the "resume parsing" consumers that `platform-core`'s
async-orchestration spec and §25 both name. **`TS-BL-044` registers nothing**: every gateway call
already dispatches asynchronously inside the gateway (`ai-platform-governance` `design.md` D10).
Five of `CAN-002`'s six checks stay synchronous in the upload request. `design.md` D2 has the split
and the reasoning; getting it wrong in either direction is the most likely error in this feature.

**Dependency edges pointing outside this feature.** `TS-BL-044` needs `ai-platform-governance`'s
`TS-BL-027`; `TS-BL-047` needs `hiring-postings`' `TS-BL-037`. Six further interfaces are built
against without being `depends_on` edges, because the work is buildable without them:
`access-control-and-admin`'s `TS-BL-018` (the evaluator `CAN-001` resolves through) and `TS-BL-020`
(the durable audit writer), `platform-core`'s `TS-BL-006` (the dispatch pattern) and `TS-BL-005`
(internal notification for both review queues), and `ai-platform-governance`'s `TS-BL-030` (the
registry `resume_extraction`'s template version is authored into) and `TS-BL-031` (the
evidence-source vocabulary). Where an interface has not landed, build against its declared shape and
say so — the pattern Phase 1 used throughout. `design.md` D11 records why the missing `TS-BL-030`
edge is inconsequential rather than an error.

**`TS-BL-041`'s dependencies are already satisfied.** `platform-core`'s `TS-BL-001` is done, and so
are `TS-BL-002` and `TS-BL-003`, which an upload endpoint also implicitly needs — all three shipped
by Sprint 0 and live in Dev. This feature's entry item is unblocked today (`design.md` D11).

**One production-activation obligation that is not a dependency edge.** The promotion gate refuses
to activate a template version in production with no passing corpus run recorded. `TS-BL-044`
authors `resume_extraction`, so it needs cases in `TS-BL-032`'s corpus before it is live **in
production** — and it is fully buildable and deployable to Dev without them, which is why D.10
records no edge and none is added here. `ai-platform-governance` `design.md` D8 argues the gate
biting before the corpus exists is the correct order.

**Three item-boundary refinements this feature makes**, under D.11's permission to refine internal
item boundaries, recorded in `design.md` D13: `TS-BL-041` creates the resume record with a sequential
version number from the first upload (because `TS-BL-044`'s already-fixed envelope takes
`resume_version_ref: <id@version>` and sits four items earlier), `TS-BL-046` creates the
`candidate_posting_applications` record (it cannot exist before the candidate and must exist the
moment it does), and `TS-BL-043` carries `DOC-008`'s chunking as well as contact-field extraction
(§25 lists text chunks among Resume Parsing's outputs, and `TS-BL-044`'s evidence references need a
target). **None changes a dependency edge, and none moves work between features.**

**Two decisions are deliberately open and must not be resolved during apply.** `OD-004` (candidate
retention period) and `OD-009` (candidate deletion, access, correction, withdrawal) are owned
outside engineering. Build `RET-007`'s source linkage and read §29 item 12's configuration key; do
not invent a retention interval and do not build a deletion workflow.

---

## 1. TS-BL-041 — Resume upload endpoint and malware scan

**Goal:** a recruiter can put a resume into the system for a posting they are authorized for, anything
that is not a readable unencrypted PDF within limits is refused in the response, and nothing
downstream can touch the document until a scanner says it is clean. Covers `candidate/resume-upload`.

```yaml
backlog_items:
  - id: TS-BL-041
    feature: candidate-intake
    depends_on: [TS-BL-001]
    status: not-started
```

**Read `design.md` D2 and D5 before starting.** D2 fixes which of `CAN-002`'s checks run in the
request and which become jobs; D5 fixes the record ordering, including the cited divergence from
§12.2 that makes `candidate_resumes.candidate_id` nullable. Both are load-bearing for every item
after this one.

- [ ] 1.1 Provision the private resume bucket by Terraform with a **quarantine prefix** and a
      **resume prefix**, no public access, and signed or proxied reads only (`SEC-004`, `SEC-006`)
- [ ] 1.2 Migrate `candidate_resumes` by reversible migration with the §12.2 columns —
      `uploaded_by`, `source_posting_id`, `file_name`, `file_hash`, `object_storage_uri`,
      `mime_type`, `size_bytes`, `malware_scan_status`, `parsing_status`, `ocr_used`,
      `extraction_confidence`, `extracted_text_location`, `parsed_json`, `is_active_version` — plus a
      sequential version number and the identity state of `design.md` D5, and **`candidate_id`
      nullable**, leaving a comment citing D5 so the divergence from §12.2 reads as a decision
- [ ] 1.3 Implement the synchronous validation set: extension, allowed MIME type, configured size
      limit (`DOC-001`, `DOC-002`, `DOC-003`, `CAN-002`), each returning a field-level error naming
      what failed
- [ ] 1.4 Implement the **bounded structural check** — header, version, trailer, encryption
      dictionary and page count only, under a declared time and memory bound, rejecting on exceed —
      and assert **no document text is extracted** in the request path (`design.md` D2). Without this
      line `CAN-002`'s "PDF readability" forces synchronous parsing, which §25 and `NFR-002` prohibit
- [ ] 1.5 Write the uploaded bytes to the **quarantine prefix** before any scan, and assert the
      resume record's `object_storage_uri` is populated only on a clean verdict — how `DOC-004`'s
      "stored only after scanning" is satisfied while the scan is asynchronous
- [ ] 1.6 Register the **malware scan as its own job type** against `platform-core`'s dispatch
      pattern, declaring §25's retry policy (retry on scanner outage) and carrying the upload's
      correlation identifier into the job. Build against `TS-BL-006`'s interface if it has not landed
- [ ] 1.7 Add the test that fails if any code path in this feature starts the scan outside the shared
      dispatch pattern (`platform-core`'s async spec carries this scenario; be its first real
      consumer, not its first exception)
- [ ] 1.8 Implement promotion on a clean verdict — quarantine to resume prefix — and assert an
      unsafe or unverdicted document reaches **no** parsing, chunking, OCR, embedding, prompt or
      rendering path (`DOC-005`, and the refusal recorded rather than silent)
- [ ] 1.9 Return a **job identifier** from the upload and expose status by identifier, with the
      intake screen reporting scan and parsing progress asynchronously (`API-007`, `NFR-001`,
      `NFR-002`)
- [ ] 1.10 Implement `ERR-005`'s five distinguishable failure kinds — validation, scan, parsing, OCR,
      storage — with retry offered where the failure is safely retryable and a stated reason where it
      is not (`ERR-003`, `DOC-011`, `NFR-006`)
- [ ] 1.11 Create the resume record with a **sequential version number** on first upload and make it
      addressable as identifier-and-version — required by `TS-BL-044`'s already-fixed
      `resume_version_ref` envelope field four items before versioning appears in D.10's titles
      (`design.md` D3, D13)
- [ ] 1.12 Implement `POST /api/postings/{postingId}/candidates/upload-resume` (§13.2) declaring its
      permission requirement and taking its verdict from the central evaluator, never from logic
      local to this module. `CAN-001`'s "authorized postings" resolves to
      `access-control-and-admin`'s `assigned-postings` scope predicate; depends on `TS-BL-018`, build
      against its interface if it has not landed
- [ ] 1.13 Assert a Recruiter's upload against an **unassigned** posting is denied server-side and not
      only by a hidden control (`C-03` as resolved, `UI-002`, `SEC-005`)
- [ ] 1.14 Seed §29 items 2, 3, 4 and 5 — size limit, allowed MIME types, OCR enablement (off), and
      the scanner endpoint and timeout — into `platform-core`'s audited runtime-configuration
      registry, and assert no unaudited setter exists for any of them
- [ ] 1.15 Enforce a rate limit on the upload endpoint (`SEC-013`)
- [ ] 1.16 Write every accepted upload through the audit path in the same transaction, carrying
      references and never candidate personal data, and assert a failed audit write fails the upload.
      Build against `TS-BL-020`'s durable writer; `app/audit/port.py` already exists and needs no
      call-site change when it substitutes
- [ ] 1.17 Build the **Candidate Resume Intake** screen (§14.2) from `design-system`'s page templates,
      form controls and dense data table — upload, per-resume processing state, and the failure kind
      where one occurred — adding no new component, and rendering permission-aware affordances as
      **absent** rather than disabled
- [ ] 1.18 Verify scanner-outage behavior by configuration in Dev rather than by waiting for a real
      outage: uploads queue, the queue depth alerts, and every existing candidate stays readable
      (`G-12`, `NFR-004`)
- [ ] 1.19 Ship the intake surface behind a feature flag, disabled by default — a disabled capability
      answers 404

---

## 2. TS-BL-042 — Resume file hash

**Goal:** the identical-PDF-uploaded-twice case that `D23`'s three fuzzy signals miss entirely becomes
a cheap indexed lookup — and this item decides nothing with it. Covers `candidate/resume-hash`.

```yaml
backlog_items:
  - id: TS-BL-042
    feature: candidate-intake
    depends_on: [TS-BL-041]
    status: not-started
```

**Read `design.md` D6 before starting.** `G-04`'s single sentence names a computation, a decision and
a consequence, and D.10 puts them in three items. This item is the first third only. Building the
decision here would duplicate `TS-BL-046`, and building the consequence would duplicate `TS-BL-048`.

- [ ] 2.1 Compute the hash over the **bytes as uploaded**, before promotion out of quarantine and
      independently of the scan verdict, and store it on the resume record (§12.2's `file_hash`)
- [ ] 2.2 Assert a quarantined unsafe document still has its hash recorded — the signal must survive a
      document that never becomes clean
- [ ] 2.3 Index the hash and implement lookup returning matching resume records **with the candidate
      each is attached to**, distinguishing same-candidate, cross-candidate and not-yet-attached
      matches — the three cases `TS-BL-046` needs to tell apart
- [ ] 2.4 Assert this item performs **no** merge, no version suppression and raises no review item,
      with a comment citing `design.md` D6 so the absence reads as a boundary rather than as
      unfinished work
- [ ] 2.5 Refuse an upload whose hash matches a document already recorded unsafe, returning the
      recorded scan-failure reason and dispatching no scan job — and record in the code that this is a
      round-trip saving, **not** a security control: a hash is trivially changed, and anything not
      already known-unsafe is still scanned (`design.md` D6)
- [ ] 2.6 Verify the same file uploaded twice produces the same hash, and that a file with different
      bytes and similar content does not

---

## 3. TS-BL-043 — Deterministic contact-field extraction

**Goal:** the fields candidate identity depends on come out of the document by rule, not by model —
and the traceable sections that later evidence references and embeddings point at exist. Covers
`candidate/resume-extraction`.

```yaml
backlog_items:
  - id: TS-BL-043
    feature: candidate-intake
    depends_on: [TS-BL-041]
    status: not-started
```

**No AI call occurs in this item, and that is the point.** `D18` chose hybrid parsing on one ground:
*"dedup keys come from deterministic logic rather than model output. Candidate identity should not
depend on a model's mood."* **This is also the "resume parsing" consumer that `platform-core`'s
async-orchestration spec and §25 both name** — its trigger is the scan job's clean verdict, so there
is no interactive request to be synchronous with even in principle (`design.md` D2).

- [ ] 3.1 Add the PDF text-extraction library `D18` anticipated and left to the stack, pinned, and
      record the choice with its version
- [ ] 3.2 Register **deterministic extraction as its own job type** against `platform-core`'s
      dispatch pattern, triggered by a clean scan verdict, declaring §25's retry policy (retry on
      transient parser failure) and carrying the correlation identifier through
- [ ] 3.3 Implement idempotency at the effect boundary: the same extraction event delivered twice
      leaves one extraction result, since the dispatch substrate is at-least-once
- [ ] 3.4 Add the test that fails if document text extraction runs inside an interactive request
- [ ] 3.5 Implement deterministic extraction of email, phone, name, dates and links by regex and
      heuristics (`D18`, `DOC-006`), and add the test that fails if any of these fields is populated
      from model output
- [ ] 3.6 Implement `DOC-008`'s chunking — content stored as sections, each with a source reference
      locating it in the document — carried by this item rather than by the embedding item that
      consumes it (`design.md` D13; §25 lists text chunks among Resume Parsing's outputs)
- [ ] 3.7 Exclude the structured contact fields from the extracted-text artifact and the section
      records, enforced by **where the data is written** rather than by a read filter (`CAN-008`,
      `PRV-002`, `design.md` D10). A contact field inside a section becomes a contact field inside a
      similarity index once `TS-BL-049` embeds it, which no downstream filter can undo
- [ ] 3.8 Detect a document with no usable text layer, record the OCR-required condition and
      `DOC-007`'s `ocr_used` flag, **register no OCR job type**, and assert the registered job types
      contain none (`project.md` and `D18`: readiness only; §29 item 4's switch exists and stays off)
- [ ] 3.9 Implement the manual-entry gate: extraction yielding none of the contact fields marks the
      resume manual-entry-required and **blocks candidate creation** until a human supplies them
      (`D18`, `design.md` D5). Record the manual entry with its actor
- [ ] 3.10 Verify partial extraction does not block: contact fields present with dates or links
      missing leaves the resume usable for identity resolution, with the absent fields recorded as
      absent — `D18`'s decoupling, under which a candidate can exist on deterministic fields alone
- [ ] 3.11 Verify exhausted retries retain the resume with its failure reason and surface it to the
      uploading recruiter and authorized administrators rather than dropping it (`platform-core`'s
      exhausted-work requirement, `DOC-011`)
- [ ] 3.12 Assert no unscanned or unsafe document is ever parsed, from this side as well as
      `TS-BL-041`'s — the ordering `DOC-005` requires deserves an assertion at both ends

---

## 4. TS-BL-044 — LLM enrichment layer

**Goal:** skills, seniority and experience come from the one governed egress, from a *reference* to a
resume version rather than from resume content, and every claim says where it came from and admits
when the resume did not support it. Covers `candidate/resume-enrichment`.

```yaml
backlog_items:
  - id: TS-BL-044
    feature: candidate-intake
    depends_on: [TS-BL-043, TS-BL-027]
    status: not-started
```

**Read `ai-platform-governance`'s `design.md` D3, D4 and D10 and its `ai-platform/ai-gateway` spec
before starting.** D3 fixed this envelope and named this call in advance — *"`resume_extraction` sends
a resume version reference"* — so the envelope is not negotiable and the payload inside it is this
item's. **D10 means this item registers no job type of its own**: the gateway dispatches
asynchronously on its behalf. Building a second dispatch path is the specific error `design.md` D2
exists to prevent.

**Structurally unlike `hiring-postings`' `TS-BL-034`, in one way that changes the contract.** That
family produced no claim about a candidate, so `ai-platform-governance` D3 deliberately left evidence
references and source labels unexercised. **This family makes claims about a person**, so `G-02` and
`G-03` apply in full (`design.md` D4).

**Production activation obligation:** the promotion gate refuses this family's new template version in
production until `TS-BL-032`'s corpus carries passing cases for it. Deployable to Dev without them.

- [ ] 4.1 Author the `resume_extraction` prompt content as a **new template version** of the family
      `TS-BL-030` registered, never by editing an existing version. Depends on the registry; build
      against its declared interface if it has not landed (`design.md` D11)
- [ ] 4.2 Build the request inside the fixed envelope — `resume_version_ref` as
      identifier-and-version — and assert the request payload carries **no resume text, no extracted
      contact field and no candidate personal data**
- [ ] 4.3 Narrow what the resolved reference supplies to the prompt: professional content with the
      structured email, phone and **name** excluded, and assert **`AI-006`'s controlled exception is
      not invoked by this family** (`design.md` D3 point 2 and 3; `PRV-001`, `PRV-002`). The cheap
      implementation sends the extracted text, and the extracted text contains the contact block
- [ ] 4.4 Mark the input as **untrusted document-derived content** at the boundary so the gateway's
      isolation applies (`AI-012`, `SEC-014`, `D18`). The isolation mechanism is the gateway's; what
      this item owes is the marking
- [ ] 4.5 Declare the output contract: skills, seniority, role summaries and domain experience, each
      claim carrying a **source label** from `TS-BL-031`'s closed vocabulary and an **evidence
      reference** to the `DOC-008` section it came from. Assert output is rejected as a contract
      violation when a claim lacks either, or carries a label outside the vocabulary, or carries a
      label other than the resume source value for this family. Build against `TS-BL-031`'s declared
      vocabulary if it has not landed
- [ ] 4.6 Implement `G-03`'s insufficiency markers — a field the resume does not support is marked
      insufficient rather than inferred — and make "marked insufficient" distinguishable from "never
      requested" (`AI-008`, `DOC-009`)
- [ ] 4.7 Declare a **bound on every free-text output field**, provisional and labelled provisional,
      held in audited configuration; record a failed run with the violation preserved and persist no
      content when a bound is exceeded (`S.5`, `AGENTS.md`'s standing bar,
      `ai-platform-governance` D7)
- [ ] 4.8 Assert this item registers **no job type** and contains **no outbound call to a model
      provider** — every invocation through the gateway, dispatch inherited
      (`ai-platform-governance` D10)
- [ ] 4.9 Handle the three typed failure kinds — validation, provider, configuration — reporting the
      kind, and assert a run reference is returned on **every** outcome including failure
- [ ] 4.10 Implement `D18`'s asymmetry as a requirement rather than an accident: enrichment failure
      leaves the candidate and resume valid and retryable, and an unenriched candidate is a valid
      readable state distinguishable from an empty enrichment (`ERR-004`)
- [ ] 4.11 Persist output only as advisory insight carrying its AI-generated marking, clear the
      marking only on a recorded human approval for that version, and assert no persistence path
      reaches a workflow-governed state field (`AI-001`, `AI-010`, `UI-004`, and
      `ai-platform-governance`'s advisory-only capability)
- [ ] 4.12 Require the **Run AI** action on the intake surface, distinct from view and edit, with the
      control absent for users who lack it and a directly-submitted request refused server-side
      (§9.3, `C-02`)
- [ ] 4.13 Refuse enrichment for a resume that has not resolved to a candidate (`design.md` D5)
- [ ] 4.14 Verify graceful degradation: with the provider unavailable, upload, validation,
      deterministic extraction, manual entry, identity resolution and merge all remain fully usable
      and the enrichment control reports temporary unavailability (`G-12`, `NFR-004`)
- [ ] 4.15 Add the family's cases to `TS-BL-032`'s corpus — contract conformance, conciseness per
      bounded field, insufficiency on a thin resume, and **adversarial prompt-injection cases**, since
      this is the first family whose input is a document the company did not write — and record that
      production activation is refused until they pass (`C-09`, §36, `ai-platform-governance` D8)

---

## 5. TS-BL-045 — Extraction confidence scoring

**Goal:** a parse that ran but should not be trusted is a visible, workable third state rather than
either a number that looks final or a failure. Covers `candidate/extraction-confidence`.

```yaml
backlog_items:
  - id: TS-BL-045
    feature: candidate-intake
    depends_on: [TS-BL-044]
    status: not-started
```

**Read `design.md` D7 before starting.** The tempting implementation adds a fifth `parsing_status`
value, and it is wrong: every consumer selecting `parsing_status == parsed` would then silently
exclude low-confidence resumes, which is the opposite of what `G-05` wants — they are usable *and*
flagged. The dependency on `TS-BL-044` is also load-bearing rather than incidental: a confidence
value published before enrichment existed would describe half a hybrid parse while appearing to
describe the whole.

- [ ] 5.1 Record confidence **per section** together with `DOC-009`'s insufficient-information
      markers, and derive the record-level roll-up into §12.2's `extraction_confidence`
- [ ] 5.2 Make the roll-up reflect the weakest material section rather than an average that hides it,
      and verify a mixed-quality parse reports differing per-section values
- [ ] 5.3 Report confidence as **not yet determined** while enrichment has neither completed nor
      failed, and publish a value accounting for absent enrichment once it fails permanently
      (`design.md` D7)
- [ ] 5.4 Implement low confidence as a **review state on the record** alongside `parsing_status ==
      parsed`, and assert the parsing status enumeration still carries only pending, parsed, failed
      and unsupported
- [ ] 5.5 Verify a consumer selecting parsed resumes **includes** low-confidence ones
- [ ] 5.6 Build the low-confidence review queue on `design-system`'s dense data table, naming the
      sections needing attention, showing state, owner and next action (`UI-003`), and denying an
      unpermitted request rather than returning an empty list
- [ ] 5.7 Implement human correction of a low-confidence parsed value: recorded with actor and
      timestamp, prior value preserved, resume leaving the review state (§36's low-quality-parsing
      mitigation names manual correction)
- [ ] 5.8 Hold the low-confidence threshold in audited configuration with provisional provenance, and
      verify changing it does not return already-reviewed resumes to the review state by itself
- [ ] 5.9 Deliver review-queue notification through `platform-core`'s internal notification engine,
      and assert no candidate-facing delivery path exists (`D04`). Build against `TS-BL-005`'s
      interface if it has not landed

---

## 6. TS-BL-046 — Candidate identity model and dedup logic

**Goal:** the same person uploaded twice becomes one candidate, two different people who share a
filing accident do not, and a machine never decides which of those it is looking at unless the
evidence is exact. Covers `candidate/candidate-identity`.

```yaml
backlog_items:
  - id: TS-BL-046
    feature: candidate-intake
    depends_on: [TS-BL-041]
    status: not-started
```

**`D.10` gives this item no edge to `TS-BL-043`, and that is correct on inspection — but there is a
preferred order.** `D23` says match inputs come from the deterministic half of parsing, which looks
like a missing dependency. It is not: `D18` makes manual entry a **first-class path** (extraction
failing *requires* it), so the identity model, signal evaluation, review queue, merge and suppression
list are all fully buildable and testable against manually-entered fields with no parser present.
Build `TS-BL-043` first where staffing allows so evaluation is written against real output;
otherwise build against its declared output shape (`design.md` D9). **The manual-entry path is not
optional scope here — it is the path this item is verifiable through.**

- [ ] 6.1 Migrate `candidates` by reversible migration with the §12.2 columns — `normalized_name`,
      `display_name`, encrypted primary email and phone, `current_location`, `profile_summary`,
      `global_status` — plus the dedup review, suppression and merge records `D23` requires
- [ ] 6.2 Implement name normalization for matching, and record the normalization actually applied so
      the open question about transliteration and script folding is answerable later against real
      data rather than re-derived (`CAN-004`, `design.md` Open Questions)
- [ ] 6.3 Implement `D23`'s five signals as **independent** evaluations, and add the test that fails
      if any signal can only fire when another has already matched — the exact defect `D23`'s revision
      corrected, whose real-world case is one candidate applying with a personal email once and a work
      email another time
- [ ] 6.4 Implement auto-merge on **exact email match only**, audited, with no other signal reaching
      it
- [ ] 6.5 Implement the cross-candidate hash match as a **review item, never a merge**, citing `G-04`:
      the hash is document-level, and the failure case is a recruiter uploading candidate A's PDF
      under candidate B's name. Merging on it would merge two real people on a filing error
- [ ] 6.6 Implement the same-candidate hash match as a link to the existing resume, raising no review
      item (the version consequence is `TS-BL-048`'s, per `design.md` D6)
- [ ] 6.7 Build the dedup review queue on `design-system`'s dense data table, naming the signal that
      raised each item and both candidates, and denying an unpermitted request rather than returning
      an empty list
- [ ] 6.8 Implement the merge as **reversible and audited in all cases** — both source records, the
      causing signal, the actor where human, a timestamp — with automatic merges recorded as
      completely as human ones, and a failed audit write failing the merge
- [ ] 6.9 Implement the un-merge action restoring the separated records, itself audited
- [ ] 6.10 Implement the suppression list: a not-a-duplicate verdict recorded with actor, timestamp
      and reason, **per pair rather than per candidate**, so the pair never re-queues while a new pair
      involving either candidate still can (`D23`)
- [ ] 6.11 Implement the invariant `design.md` D1(b) commits to: **a merge never silently merges,
      withdraws, re-stages or re-ranks an application.** Where a merge leaves one candidate holding
      two applications against one posting, record and surface the condition and take no further
      action. Add the test that fails if merge code writes an application's stage field
- [ ] 6.12 Create the `candidate_posting_applications` record on resolution, carrying candidate,
      posting, submitting recruiter and `CAN-007`/`G-10`'s source type, and creating no second link
      where one already exists for that candidate and posting (`CAN-003`, `CAN-005`,
      `design.md` D13). **Create only the identity, source and resume-pin columns** — `status`,
      `latest_rank`, `latest_match_score`, `mandatory_criteria_status`, `shortlist_reason` and
      `final_outcome` are written by `matching-and-ranking`, `interview-pipeline` and
      `decision-and-offers`
- [ ] 6.13 Refuse candidate creation where no contact field is present, holding the resume in the
      manual-entry state, and run resolution against manually-entered fields exactly as against
      extracted ones (`D18`, `design.md` D5)
- [ ] 6.14 Assert an unresolved resume is **not** addressable as a `resume_version_ref`, not
      attachable to an application, and not enrichable — the two constraints that make `candidate_id`
      nullable safe (`design.md` D5)
- [ ] 6.15 Add the test that fails if enriched skills, seniority or summaries are used as a match
      signal — `D18`'s central guarantee, and the one most likely to erode quietly once enrichment
      output exists and looks useful
- [ ] 6.16 Store contact fields under approved sensitive-data controls and return them only to a
      permitted caller, **omitting the fields** rather than failing the response, and verify an
      Administrator with no active break-glass grant does not receive them (`SEC-003`, `API-006`,
      `D16`)
- [ ] 6.17 Implement the candidate endpoints §13.2 enumerates — search, profile, timeline and resume
      listing — each declaring its permission requirement, with the scope filter applied as a **query
      predicate** rather than by discarding fetched rows
- [ ] 6.18 Verify a candidate not selected for their posting keeps their record, resume versions and
      application history (`CAN-006`, `BR-020`, `RET-001`)
- [ ] 6.19 Implement `RET-007`'s source linkage — every derived record (sections, enrichment, later
      embeddings) linked back to the resume version it came from — now rather than when `OD-004` and
      `OD-009` land, because retrofitting it means finding every derived record without a link
- [ ] 6.20 Build the candidate profile and timeline screens from `design-system`'s page templates and
      dense data table, adding no new component, and rendering the evidence-source label component for
      enriched claims rather than deriving source values locally

---

## 7. TS-BL-047 — Duplicate backstop at posting level

**Goal:** when the submission-time check misses entirely, the same person does not quietly occupy two
of five priority slots — a recruiter sees a warning and decides. Covers `candidate/duplicate-backstop`.

```yaml
backlog_items:
  - id: TS-BL-047
    feature: candidate-intake
    depends_on: [TS-BL-046, TS-BL-037]
    status: not-started
```

**Read `design.md` D1 in full before starting. This is the one item in this backlog whose underlying
decision was never confirmed.** `D23a`'s status line reads *"recommendation made, not yet
confirmed"*; every other citation in this feature is settled. D1 splits the call rather than building
through it:

- **Its two-check recommendation is treated as confirmed** — the problem is confirmed (`D20`'s slot
  arithmetic and `BR-012`), the intervention is the weakest the architecture permits, `D.10` already
  created this item, and `D07`'s volumes make it cheap. Build it, behind its own flag.
- **Its follow-on application-merge rule is treated as genuinely open** — which application survives,
  which ranking is retained or whether a re-rank is forced, what happens at differing stages. Two of
  those three inputs belong to `matching-and-ranking` and `interview-pipeline`, both unproposed.
  **Do not invent it.**

- [ ] 7.1 Implement posting-scoped pairwise comparison over the candidates holding applications
      against one posting, using the **parsed** signals — employer history, education, dates, skill
      fingerprint — and not repeating the submission-time contact-field check (`D23a`)
- [ ] 7.2 Verify the scope: near-duplicates on **different** postings raise nothing here
- [ ] 7.3 Make detection run against a posting's current candidate set and be re-runnable, not
      triggered per submission — `D23a`'s two checks sit at different depths on purpose
- [ ] 7.4 Verify a re-run reports each pair once and does not re-raise a dismissed pair
- [ ] 7.5 Surface the result as a **non-blocking warning** identifying the counterpart on the
      posting's candidate list, and assert ranking, shortlisting, scheduling and selection all remain
      available on both entries
- [ ] 7.6 Implement dismissal through the **same remembered-verdict mechanism** `TS-BL-046`'s review
      queue uses, rather than a second suppression store
- [ ] 7.7 Implement confirmation as routing into `TS-BL-046`'s candidate-level review and merge path,
      and assert **no merge, application write, ranking write or state-field write exists in this
      item** regardless of how strong a similarity is
- [ ] 7.8 Implement the open-condition record: where a confirmed merge leaves two applications against
      one posting, record and surface it to the posting's owner and recruiters, leave both
      applications exactly as they were, and keep it visible as a blocker until a human resolves it
      (`UI-003`)
- [ ] 7.9 Add the test that fails if any code path selects a surviving application, retains a ranking
      or triggers a re-rank in response to that condition — the assertion that keeps `design.md`
      D1(b)'s open question actually open
- [ ] 7.10 Render the warning through `matching-and-ranking`'s ranked-list surface (`TS-BL-053`) where
      it exists, and on the posting's candidate list otherwise, using existing `design-system`
      components. This is the one place this feature reaches into an unproposed feature's surface;
      keep the contract to a warning payload the list renders, so `TS-BL-053` consumes rather than
      accommodates it
- [ ] 7.11 Gate the whole capability behind its own declared feature flag, **disabled by default**,
      answering 404 while disabled — the mechanism `design.md` D1's risk mitigation depends on, so
      rejection of `D23a` costs a flag flip rather than a revert
- [ ] 7.12 Record in `KNOWN_ISSUES.md` that this capability implements an unconfirmed recommendation
      and is disabled by default pending an owner decision, the same way the unconfirmed Hubble
      contract and the absent AI provider are recorded

---

## 8. TS-BL-048 — Resume versioning

**Goal:** a ranking score computed against a resume stays explicable after a newer resume arrives —
the same guarantee `hiring-postings` gave `jd_version`, for the other term of the same tuple. Covers
`candidate/resume-versioning`.

```yaml
backlog_items:
  - id: TS-BL-048
    feature: candidate-intake
    depends_on: [TS-BL-046]
    status: not-started
```

**The version *rows* already exist** — `TS-BL-041` task 1.11 creates them, because `TS-BL-044`'s fixed
envelope needed them four items earlier. This item adds immutability, the active version, application
pinning, and the retrieval guarantee (`design.md` D13).

- [ ] 8.1 Make a resume version's content, hash and storage reference **immutable** once created,
      rejecting any write against them (`DOC-010`)
- [ ] 8.2 Implement sequential version creation on each new upload for a candidate, leaving prior
      versions unchanged
- [ ] 8.3 Implement the identical-file rule: an upload whose hash matches an existing version **of the
      same candidate** links to it and creates no new version, while the upload attempt is still
      recorded. Leave a comment citing `design.md` D8 — this is a deliberate divergence from
      `DOC-010`'s literal "every upload preserves a new version," reconciled on the grounds that
      `DOC-010`'s purpose is no *overwrite* and linking overwrites nothing
- [ ] 8.4 Verify a file with different bytes and similar content **does** create a new version — the
      signal is document-level, not content-level
- [ ] 8.5 Implement exactly one active version per candidate, with a new version becoming active and
      the prior one ceasing to be, and assert the one-active invariant holds after every path
- [ ] 8.6 Implement making an earlier version active again as a permission-gated action recorded with
      actor, reason and timestamp
- [ ] 8.7 Implement application pinning: the application records the version it was submitted with,
      and a newer upload for the candidate **does not** re-point it. Add the test that fails if any
      code path re-points an application without an explicit human action (`design.md` D8)
- [ ] 8.8 Verify a ranking computed against version one remains attributable to that version after
      version two exists, and that version one's content is still retrievable — the property whose
      absence `hiring-postings` D9 describes as leaving scores "not wrong-looking, just quietly
      meaningless"
- [ ] 8.9 Implement re-pointing as an explicit permission-gated action with a **mandatory reason**,
      recording previous and new version, emitting no state transition and recomputing no ranking —
      whether a re-point forces a re-rank is `matching-and-ranking`'s to decide under `G-07`
      (`WF-005`)
- [ ] 8.10 Verify the prior ranking is preserved rather than deleted on a re-point, and that its
      computation against a superseded version is visible
- [ ] 8.11 Verify a version reference resolves to that version's content after later versions exist,
      after the referencing application closes, and after the source posting closes (`RET-002`,
      `RET-003`, `WF-007`) — the property `ai-platform-governance`'s evidence-labeling requirement
      depends on when it requires a claim on a versioned record to name the version
- [ ] 8.12 Implement `GET /api/candidates/{candidateId}/resumes` (§13.2) listing version number,
      source posting, uploader, timestamp, active indicator, scan state, parsing state and confidence,
      declaring its permission requirement and denying an unpermitted request rather than returning an
      empty list
