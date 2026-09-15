# Candidate Intake — Resumes, Identity and Deduplication — Design

## Context

See `proposal.md` — Why, for motivation, and the eight delta specs under `specs/` for the behavior
contracts. This document covers only what this feature must actually settle: the one unconfirmed
decision it is built on top of, the synchronous/asynchronous split that `platform-core`'s async
spec and §25 both point at without resolving, the record ordering that `D18`'s
identity-before-enrichment rule forces, and the boundaries between items that a single sentence in
`G-04` or `G-05` spans.

Constraints that shape the approach:

- **One decision this feature depends on is explicitly unconfirmed.** `D23a`'s status line reads
  *"recommendation made, **not yet confirmed**"* — the only such decision in this backlog. Every
  other citation here (`D18`, `D23`, `G-04`, `G-05`, `D07`, `D20`, `D22`) is settled. D1 makes the
  call rather than building through it.
- **The request envelope for `resume_extraction` is already fixed, and this feature did not fix
  it.** `ai-platform-governance` `design.md` D3 designed the gateway envelope against
  `hiring-postings`' `TS-BL-034` and generalized it to this exact case in one sentence:
  *"`resume_extraction` sends a resume version reference, `candidate_ranking` sends posting and
  resume references."* The envelope is a contract this feature consumes. See D3.
- **AI runs already dispatch asynchronously; non-AI work does not.** `ai-platform-governance` D10
  settled that every gateway call registers as a job type *inside the gateway*, so `TS-BL-044`
  inherits async execution and must not build it again. `platform-core`'s async-orchestration spec
  and §25 name **"resume parsing"** — not "AI calls" — as an intended consumer, and §25 lists
  **Malware Scan** as its own job type with its own trigger and retry policy. Those two are this
  feature's, and they are not AI calls. See D2.
- **This is the first untrusted input the product ingests, and the first candidate personal data it
  stores.** `SEC-006`, `SEC-014`, `AI-012`, `PRV-001`, `PRV-002`/`CAN-008` and `API-006` all first
  apply to a real record here. Three substrate guarantees written against employee-authored data
  become load-bearing: gateway content isolation, reference-only run logs, and no-personal-data
  audit records.
- **Volume is small and was decided so.** `D07`: fewer than 5,000 candidates and fewer than 30
  postings. This is what makes `D23a`'s posting-scoped pairwise comparison trivially affordable and
  what keeps a full-database deterministic signal lookup a plain indexed query.
- **Settled stack** (`D09`): FastAPI on Python, PostgreSQL on Cloud SQL, React and TypeScript,
  Terraform on GCP with GitLab CI/CD, background work on the landing zone's Pub/Sub → Eventarc →
  Workflows → Cloud Run Job chain. A **PDF text-extraction library** is the one addition `D18`
  anticipated and left to the stack.
- **Only Local and Dev are provisionable.** UAT and Prod are validated Terraform that is never
  applied. One consequence specific to this feature: the malware scanner is a real external
  dependency, so its outage behavior must be exercised in Dev by configuration rather than by
  waiting for a real outage.

## Goals / Non-Goals

**Goals:**

- Candidate identity that depends on no model output, in the strong sense `D18` intended — not
  "prefers deterministic fields" but "cannot be established from enrichment at all."
- An untrusted document that is never parsed, embedded, prompted or displayed before it is known
  clean, and whose unsafe form exists in exactly one place.
- A resume version reference that means the same thing to a ranking score in three months as it
  does at upload — the same guarantee `hiring-postings` D9 gave `jd_version`, for the other term of
  the same tuple.
- The unconfirmed decision handled visibly: what is being treated as confirmed, on what grounds,
  and what is deliberately left unbuilt because it is not.
- Boundaries stated where one requirement sentence spans several items, so no item half-builds a
  rule its neighbour also half-builds.

**Non-Goals (design-level boundaries beyond the proposal's scope):**

- **No parser accuracy target.** `D18` chose hybrid parsing for identity independence, not for
  extraction quality. `G-05` handles a bad parse by surfacing it for review; a measured accuracy
  bar would need a labelled corpus of real resumes, which does not exist and could not be committed
  to this repository if it did.
- **No candidate search ranking or relevance.** `CAN-006` requires candidates stay searchable.
  Filtering and paging come from `design-system`'s dense data table; semantic search over resume
  content is `matching-and-ranking`'s vector retrieval, and building a second search path here
  would be the shape of duplication `AGENTS.md` warns about.
- **No merge automation beyond `D23`'s two auto cases.** Exact email match auto-merges; identical
  hash under one candidate links. Everything else raises a review item for a human, and no
  confidence threshold is introduced that would silently promote a review case to an auto-merge.
- **No PII detection or classification engine.** Contact fields are separated because
  `CAN-008`/`PRV-002` name them, by schema placement, not by scanning free text for anything that
  looks personal.
- **No second audit writer, no second queue, no second egress, no second evaluator.** All four are
  consumed.

## Decisions

### D1 — `D23a` is unconfirmed: its detection half is treated as confirmed, its merge-resolution half is not

**Flagged rather than resolved silently, because this is the only decision in this backlog whose
status line says it was never confirmed.** Every other citation this feature rests on — `D18`,
`D23`, `G-04`, `G-05` — is a settled decision. `D23a`'s reads: *"**Status:** recommendation made,
**not yet confirmed**."* Building against it as though it were settled, the way the rest of these
citations are, would misrepresent the record.

**The call, stated in two parts:**

**(a) `D23a`'s two-check recommendation is treated as confirmed for this feature's purposes.**
`TS-BL-047` builds the posting-scoped content check and its non-blocking advisory warning, as
recommended. Four grounds, in descending order of weight:

1. **The problem it addresses is confirmed, even though the fix was not ratified.** `D23a`'s stated
   harm is that two duplicate records both ranked against one posting can occupy two of five
   priority slots. `D20` (five ranked slots per vacancy, displacement with reason) is settled, and
   `BR-012` in the reference spec independently confirms the arithmetic. A confirmed rule is being
   corrupted by a confirmed miss case — `D23` itself enumerates the misses, including the one that
   motivated its own revision (a candidate applying with a personal email once and a work email
   another time, with no other shared identifier).
2. **The recommendation introduces no governance surface, so the cost of being wrong is a
   deletable UI affordance.** It is advisory, non-blocking, never auto-merging, and it triggers no
   state transition — which is to say it is the weakest intervention the architecture permits, and
   is already what `project.md`'s central rule requires of anything AI- or heuristic-derived. There
   is no decision an owner could later make that this implementation would have foreclosed.
3. **The backlog already treats it as work to be done.** `D.10` created `TS-BL-047` with dependency
   edges to `TS-BL-046` and `TS-BL-037`, added it to the running total, and `D.11` declared the
   decomposition settled and closed explore. Building nothing would leave a numbered item with no
   content in a backlog that claims to be complete — a worse inconsistency than the one being
   resolved.
4. **`D07` makes it cheap and `D23a` says so.** Tens of candidates per posting, so pairwise
   comparison within a posting is trivial. There is no scaling argument that would make an owner
   want to refuse it.

**One weaker observation, recorded as supporting context rather than as a fifth ground:** in this
document, "recommendation made, not yet formally decided" has repeatedly not meant "not adopted" —
the separate *Recommendations made, not yet formally decided* table carries the `Application`
entity, the `RankingScore` reproducibility tuple and `MatchSuggestion`, all three now **canonical in
`domain-model.md`** and built against by three proposed features. **It is deliberately not one of
the four, because the parallel is loose in the direction that matters:** those three are foundational
data-model conventions that nothing ever seriously contested, whereas `D23a` is a product judgement
about whether a recruiter finds another advisory signal useful — something an owner could plausibly
reject on noise grounds, as this document's own Risks section concedes. The precedent establishes
that the label marks an absent ratification step rather than a live objection *in general*; it does
not establish that for this decision. (`D23a` is not itself on that table; it carries its status
inline.)

**What makes deferring it worse than building it:** the warning renders on
`matching-and-ranking`'s ranked list (`TS-BL-053`). Deferring means either retrofitting a signal
into another feature's surface later, or `matching-and-ranking` designing that surface around a
warning that may or may not arrive. Building it now, behind the feature flag every surface ships
behind anyway (`config.yaml`, `ENG-009`), makes "turn it off" the cost of being wrong — which is
strictly cheaper than either alternative.

**(b) `D23a`'s follow-on merge rule is treated as genuinely open, and `TS-BL-047` is scoped to
exclude it.** `D23a`'s own words: *"when a post-submission duplicate is confirmed and merged, the
**two applications against the same posting must also merge**. Which application survives, which
ranking score is retained (or is a re-rank forced), and what happens if the two sat at different
stages, all need explicit rules."* Nothing anywhere wrote them, and `D23`'s consequences list
carries the same gap in a wider form (*"merging two candidates who are live in different
pipelines"*).

This half is not treated as confirmed, for a reason independent of its status: **two of the three
facts it would operate on do not exist and are not this feature's.** A ranking score belongs to
`matching-and-ranking` (`TS-BL-051`/`TS-BL-052`, with `G-07`'s ranking versions), and an
application's stage belongs to `interview-pipeline` — both unproposed. A rule invented here about
which score survives a merge would be invented against a score model this feature has never seen.

**The invariant this feature commits to instead**, which is decidable now and pre-empts nothing:

> A candidate merge SHALL NOT silently merge, withdraw, re-rank or re-stage any application. Where
> a merge leaves one candidate holding two applications against one posting, that condition is
> recorded and surfaced for human resolution, and the merge is refused no further.

That is not a compromise position; it is what the rest of the architecture already requires.
`config.yaml`: *"every state transition goes through the workflow service. No surface writes a
state field directly."* An application's stage is a governed state field, so a dedup merge
collapsing two applications *would already be prohibited* — the open question is only who resolves
it and how, not whether the merge may do it quietly. Recording the condition rather than resolving
it also leaves the eventual rule cheap to add: whoever answers it writes a resolution action
against an existing, visible, already-audited record.

**What this means for whoever owns the decision:** the open question in this document is narrowed
to the application-merge rule alone, with the two features that own its inputs named. The
detection, the warning and the confirmed-duplicate record are built and need no further sign-off.

*Alternative considered for (a):* scope `TS-BL-047` to detection with no user-visible surface —
compute and record possible-duplicate pairs, render nothing, pending confirmation. Rejected as the
worst of both: it spends the whole implementation cost, produces records nobody reads, and gives
an owner nothing to actually evaluate. A decision about whether recruiters find a warning useful
cannot be made from a feature nobody can see.

### D2 — Two non-AI job types register against `TS-BL-006`; five of `CAN-002`'s six checks stay synchronous; `TS-BL-044` registers nothing

`platform-core`'s async-orchestration spec names **"resume parsing"** in its own Purpose as a
consumer that must inherit the shared pattern, and §25 lists **Malware Scan** and **Resume
Parsing** as two separate job types with distinct triggers and retry policies. Neither is an AI
Gateway call. Meanwhile `ai-platform-governance` D10 already established that every gateway call —
including `TS-BL-044`'s — dispatches through `TS-BL-006` *inside the gateway*. Leaving the split
implicit would produce exactly one of two errors: `TS-BL-044` building a second dispatch path the
gateway already provides, or `TS-BL-041`/`TS-BL-043` doing unbounded work inside an interactive
request because "AI calls are the async ones."

**The split:**

| Work | Where it runs | Why |
|---|---|---|
| Extension and MIME-type check (`DOC-001`, `DOC-002`) | Synchronous, in the request | No external service, no document parse. A non-PDF must be refused in the response, not by polling a job |
| Size check against §29 item 2's configured limit (`DOC-003`) | Synchronous | Known before a byte is stored |
| PDF structural readability and encryption detection (`DOC-003`) | Synchronous, **bounded** | Reads header, version, trailer, encryption dictionary and page count only. **Never extracts text** — see the bound below |
| **Malware scan** (`CAN-002`, `DOC-003`) | **Registered job type** | §25 gives it its own trigger ("Resume upload") and retry policy ("retry on scanner outage, do not parse until clean"). An external scanner's availability must not sit on an interactive request (`G-12`, `NFR-004`) |
| **Deterministic extraction and chunking** (`TS-BL-043`) | **Registered job type** | §25's "Resume Parsing", trigger "Clean PDF"; `NFR-002` names resume parsing asynchronous by ID |
| OCR | **Not registered** | §25 lists it as its own job type; `project.md` and `D18` scope this product to readiness only. `TS-BL-043` detects and records the condition |
| LLM enrichment (`TS-BL-044`) | **Inherits async from the gateway** | `ai-platform-governance` D10. Registers no job type of its own |

**The strongest reason deterministic extraction is asynchronous is not latency — it is that there
is no request to be synchronous with.** Its trigger is the scan job's clean verdict (§25: "Clean
PDF"), which arrives after the upload response has already been returned. Even if extraction were
instantaneous, it could not run inline. The latency argument is real too and worth recording
because it is the one an implementer will be tempted to override: PDF text extraction is CPU- and
memory-bound work over an attacker-influenced file at the configured maximum size, and
pathological-PDF resource exhaustion is a well-known parser failure class. Doing it in an API
worker couples interactive capacity to untrusted input.

**Where the synchronous line falls inside "PDF readability," stated because it is the one genuinely
ambiguous case.** `CAN-002` requires checking "PDF readability" at upload, and reading a document's
structure is not the same as extracting its text. The synchronous check opens the document
structure only — enough to answer *is this a PDF, is it encrypted, how many pages* — under its own
declared time and memory bound, and refuses rather than proceeding when the bound is exceeded. Full
text extraction is the job's work. Without that line, `CAN-002` read literally forces synchronous
parsing, which is precisely what §25 and `NFR-002` prohibit.

**The upload response therefore returns a job identifier** (`API-007`), and the intake screen
reports progress asynchronously — the scenario `platform-core`'s async spec already carries
("a screen triggering background work still returns within the interactive threshold").
`ERR-005`'s five distinguishable failure kinds — validation, scan, parsing, OCR, storage — map
onto this split directly, which is a useful confirmation that the reference spec expected the
boundary to fall roughly here.

*Alternative considered:* one "resume intake" job type covering scan and extraction together.
Rejected on §25's own shape — it declares two rows with different retry semantics (retry on
*scanner* outage; retry on *transient parser* failure), and `DOC-005` requires an unsafe file never
reach parsing at all. One job type would either retry a parse after a scanner outage or re-scan
after a parse failure, and the "do not parse until clean" ordering would become an internal
convention rather than a boundary between two registered types.

*Alternative considered:* scan synchronously and keep the upload request open. Rejected — it puts a
third-party service's timeout on a user-facing request, and `DOC-004`'s "stored only after
scanning" is satisfiable without it (D5).

### D3 — The enrichment request carries a resume version reference, and the content the model sees is narrower than the resume

`ai-platform-governance` D3 fixed the envelope and named this call in advance. This feature supplies
the payload:

```
request:
  family:        resume_extraction
  caller:        { actor, page: Candidate Resume Intake, action: Run AI, correlation_id }
  input:         { resume_version_ref: <id@version> }
response:
  run_ref:       <ai_run id>
  status:        succeeded | failed
  output:        contract-satisfying enrichment, or
  failure:       { kind: validation | provider | configuration, detail }
```

`resume_version_ref` is `id@version` for the same reason `jd_draft_ref` is: `RankingScore` is a
function of `resume_version`, and an enrichment attributed to "this candidate's resume" rather than
to a specific version becomes unattributable the moment a second version exists. `TS-BL-048` is
what makes the reference resolvable, and it lands after `TS-BL-044` in `D.10`'s order — the same
apparent inversion `hiring-postings` D3 hit with `jd_draft_ref`, and it resolves the same way:
`TS-BL-041` creates a resume row with a sequential version number from the first upload, so a
resume is always *version n* of something. What `TS-BL-048` adds is immutability, the active-version
concept and application pinning, not the existence of versions. Recorded as a boundary refinement
in D13.

**Three things follow that are specific to this family, and are the reason the envelope's
"references, not content" shape matters more here than it did for JD generation.**

1. **The gateway resolves the reference; the resume text never appears in the request or the run
   record.** For a JD draft this was almost incidental — D3 observed that the draft "is already a
   record by the time the PM asks for AI help." For a resume it is the whole privacy control:
   `PRV-001` (minimize personal data sent to AI), the run log's reference-only requirement, and the
   audit trail's no-candidate-data rule all hold structurally rather than by a redaction step,
   because no candidate-intake code path ever hands resume text to the gateway.
2. **What the reference resolves *to* is the extracted text with the deterministically-extracted
   contact fields removed — not the resume.** `resume_extraction`'s output is skills, seniority,
   role summaries and domain experience. None of it requires an email address, a phone number or a
   postal address, and `TS-BL-043` has already extracted all three by the time this runs. So the
   prompt input is narrowed, and **`AI-006`'s controlled exception permitting candidate contact
   information in a prompt is not invoked by this family.** That answers, for this family, the open
   question `ai-platform-governance` left as *"whether the controlled exception ever gets used"* —
   no. It is worth stating positively rather than leaving as an absence, because the cheap
   implementation is "send the extracted text," and the extracted text contains the contact block.
   `PRV-002`/`CAN-008` require that separation anyway (D10); this is where it first pays for itself.
3. **The name is untrusted-but-needed, and it is not sent.** A candidate's name is a contact field
   under `CAN-008` and is extracted deterministically. Enrichment neither needs it nor should
   receive it — a model that can see the name can produce a claim keyed to it, which is precisely
   the inference `PRV-006`/`OD-010` rule out collecting. The prompt receives professional content.

**Prompt injection is handled by the gateway, and this feature must not build a second answer to
it.** `AI-012`/`AI-013`/`AI-014` and `SEC-014` are the gateway's untrusted-content-isolation
requirement, already specified and asserted. `D18` names resume text as untrusted input that "must
be delimited and treated as data, never as instructions," and adds the observation that the larger
surface is ranking, not parsing. What this feature owes is (a) marking the input as untrusted at the
boundary so the gateway's isolation applies to it, and (b) adversarial cases in `TS-BL-032`'s
corpus, since this is the first family whose input is a document the company did not write.

*Alternative considered:* send resume text in the request and let the gateway redact. Rejected —
it inverts D3's stated benefit ("the gateway never held the content in the first place"), makes the
reference-only run log a redaction step that can fail, and would make every future consumer's
privacy posture depend on redaction correctness rather than on structure.

### D4 — This is the first family that makes claims about a candidate, so evidence and source labeling apply where they did not for `TS-BL-034`

`ai-platform-governance` D3 point 2 is explicit that it chose JD generation as its worked example
partly *because* it needed neither: *"JD generation produces no claim about a candidate, so
evidence references and source labels do not apply to it. Writing the envelope against a family
that needs neither prevented baking candidate-insight assumptions into the shared shape."*

`resume_extraction` is the other case. "Eight years of Python" is a claim about a person, and `G-02`
(evidence references and source labelling) plus `G-03` (insufficiency outputs rather than guessing)
were adopted precisely for it. Concretely, `TS-BL-044`'s output contract requires each enriched
claim to carry:

- a **source label** from `TS-BL-031`'s closed vocabulary — always `resume` for this family, which
  is a constraint worth asserting rather than a triviality, since a family that can emit any label
  can emit a wrong one;
- an **evidence reference** to the `DOC-008` chunk the claim came from, which is why chunking is
  `TS-BL-043`'s work and not `matching-and-ranking`'s;
- an **insufficiency marker** instead of a value where the resume does not support one (`G-03`,
  `DOC-009`) — the failure mode `G-03` was adopted to attack, and the one most likely to erode trust
  in ranking downstream;
- a **declared bound per free-text field** (`S.5`, `ai-platform-governance` D7), provisional and
  labelled provisional, held in audited configuration. `D4`'s compose-from-bounded-parts answer in
  `hiring-postings` does not transfer and does not need to: enrichment output is structured, not a
  document.

**A consequence for `design-system`, already provided for.** Its foundations spec carries an
evidence-source-label component that "renders the source value it is given without interpreting or
deriving it," and `ai-platform-governance` D9 notes that clause is only satisfiable if exactly one
component derives source values — `TS-BL-031`. This feature is the first real producer of those
values, so it is where that chain is exercised end to end for the first time.

**`TS-BL-031` is therefore an interface this feature builds against, and `D.10` records no edge to
it.** That is consistent with how three Phase 1 features handled the same situation and how
`hiring-postings` handled four of them: the work is buildable against a declared vocabulary, so the
obligation is stated in `tasks.md` rather than added as a dependency. `TS-BL-031` is a Phase 1 item
and lands well before this one regardless.

*Alternative considered:* treat enrichment as neutral structured extraction needing no labels,
since it reports what the resume says rather than judging it. Rejected — the distinction does not
survive contact with the output. "Senior" is an interpretation, `D18` lists seniority among the
interpretive fields for exactly that reason, and `G-02` was adopted so that an Administrator's
run-log access cannot become a personal-data back door. An unlabelled claim about a person is the
thing `G-02` exists to prevent.

### D5 — The candidate record is created after identity resolves, so `candidate_resumes.candidate_id` is nullable until it does

This is the record-ordering consequence of `D18`, and nothing upstream resolves it. `D18` is
unambiguous: deterministic extraction failing means *"contact fields empty → **manual entry
required before the candidate record can be created**, because dedup depends on those fields."* So
a candidate cannot exist at upload time. But `DOC-004`, `DOC-005`, `SEC-006` and `G-04` all require
the file to be stored, scanned and hashed before anything about a candidate is known — and §12.2's
`candidate_resumes.candidate_id` is a non-nullable parent reference. The reference spec is simply
silent on where an upload lives in between.

**Resolution: one table, with `candidate_id` nullable until identity resolves.** A resume row is
created by `TS-BL-041` at upload with its uploader, source posting, file metadata, storage
reference and scan state, and no candidate. `TS-BL-046` sets `candidate_id` when a signal
evaluation resolves — to an existing candidate, to a new one, or not at all — and creates the
`candidate_posting_applications` row at the same moment, since an Application is
*candidate × posting* and cannot exist before the candidate does.

The record also carries an explicit **identity state** (`unresolved` | `resolved` |
`manual_entry_required` | `review_queued`) as a stored field rather than a value derived at read
time. Two constraints hold on it: a resume with no candidate is **not addressable as a
`resume_version_ref`** and cannot be attached to an Application or enriched; and only a resolved
resume can be an active version.

*Why a stored field rather than a derived one:* the intake screen and both review queues are
list views over this state, and `hiring-postings` D8 already settled the general form of this
question in the other direction's favour for a governed field. Here the specific reason is
narrower — `manual_entry_required` and `review_queued` are not derivable from the resume row at
all. One is a parser outcome plus the absence of contact fields; the other is the existence of a
review item raised by a signal evaluation that may have compared against a candidate since merged.
Deriving them means re-running dedup on every page load.

*A deliberate, cited divergence from §12.2:* its `candidate_resumes.candidate_id` is not marked
nullable. This is the same class of divergence `hiring-postings` D2 recorded against `JOB-002`'s
field list — the spec's table and a decision made after it disagree, `AGENTS.md` gives the
exploration notes precedence, and the divergence is recorded here so a reader comparing the two
finds a decision rather than an omission.

*Alternative considered:* a separate `resume_submissions` staging table promoted into
`candidate_resumes` on resolution. Rejected — it duplicates the hash, scan status, storage
reference and parse status columns across two tables, which means `G-04`'s cross-candidate hash
lookup has to search both, and a resume that never resolves ends up in neither the intake view nor
the candidate's history. The nullable column keeps one row with one identity for its whole life,
which is also what `RET-002`'s "resume versions retained with source posting, uploader, timestamp"
implies.

*Alternative considered:* create a provisional candidate at upload and merge it later. Rejected on
`D18`'s own reasoning — it makes every upload a candidate record, so the dedup review queue becomes
the normal path rather than the exception, and `D23`'s reversible merge becomes load-bearing for
ordinary intake rather than for genuine ambiguity.

### D6 — `G-04`'s one sentence spans three items; each carries exactly one third

`G-04` reads, in full: *"Same hash under the same candidate means the same resume: link to the
existing record, do not create a new version. Same hash pointing at a different candidate raises a
review-queue item rather than auto-merging."* That sentence names a computation, a decision and a
consequence, and `D.10` puts them in three different items (`TS-BL-042`, `TS-BL-046`, `TS-BL-048`).
Left unstated, each would half-build it.

| Third | Item | Content |
|---|---|---|
| Compute and expose the signal | `TS-BL-042` | Hash over the bytes **as uploaded**, stored, indexed, with lookup by hash. Decides nothing |
| Evaluate it as a signal | `TS-BL-046` | Two rows of `D23`'s table: same-candidate → link; cross-candidate → review queue, never auto-merge |
| Act on the same-candidate case | `TS-BL-048` | Creating **no new version** for an identical file, which is a versioning rule |

**Hashed over the bytes as uploaded, before promotion out of quarantine**, so the hash identifies
what the recruiter sent rather than what storage holds — and so it is available to `TS-BL-046` even
for a file that never becomes clean.

**Why cross-candidate never auto-merges, restated because it is the counter-intuitive half:** the
hash is a **document-level** signal. `D23`'s note says it plainly — an identical file proves two
documents are the same, not that two candidate records are the same person, and the failure case is
a recruiter uploading candidate A's PDF under candidate B's name. Auto-merging on it would merge
two real people on the strength of a filing error.

**One small addition, recorded as a convenience rather than a control:** the hash of a quarantined
unsafe file is retained, so the identical file re-uploaded is refused immediately with the
scan-failure reason instead of being re-scanned. This is not a security mechanism — anything not
already known-unsafe is still scanned, and a hash is trivially changed — it removes a pointless
round trip to the scanner. Stated because an implementer would otherwise reasonably read it as
either a control (it is not) or an oversight (it is not).

### D7 — Low extraction confidence is a review state on the record, not a fifth `parsing_status` value

`G-05` describes confidence as *"a third state between 'parsed' and 'failed'"* in `D18`'s failure
model. §12.2 gives `candidate_resumes.parsing_status` four values — `pending`, `parsed`, `failed`,
`unsupported` — and `extraction_confidence` as a separate decimal. Read together, that invites
adding a fifth enum value.

**Resolution: `parsing_status` keeps its four values and low confidence is a distinct review state
on the record, carrying its reason.** `parsing_status` answers *did the parse run*;
`extraction_confidence` and its insufficiency markers answer *should its output be trusted*. Those
are independent, and conflating them makes `parsed` ambiguous in a way that breaks the decoupling
`D18` bought: *"a candidate can exist on deterministic fields alone."* A candidate whose parse
succeeded with low confidence is a real, usable candidate flagged for review — not a candidate in a
failure state.

**Confidence is recorded per section as well as rolled up.** `DOC-009` says confidence *values*,
plural, plus insufficient-information markers, and `DOC-008` already requires traceable sections.
A single record-level number cannot say *which* field to check, which is the entire operational
purpose `G-05` gives it ("drives recruiter review of low-confidence parses"). `extraction_confidence`
holds the roll-up because §12.2 declares it and `matching-and-ranking` will read one number; the
per-section values are what the review queue renders.

**`TS-BL-045` lands after `TS-BL-044`, and that ordering is right rather than incidental.**
Confidence must span both halves of a hybrid parse. A value published while enrichment is still
outstanding would describe the deterministic half alone while occupying the field, and the name, that
every consumer reads as the parse's confidence — so nothing distinguishes a provisional number from a
final one, and a resume that will fall below the review threshold once enrichment lands reads as
trustworthy until it does. Hence the requirement that confidence report as not-yet-determined until
enrichment has completed or failed, rather than publishing a partial value shaped exactly like a
complete one.

*Alternative considered:* add `parsed_low_confidence` to `parsing_status`. Rejected — every consumer
checking `parsing_status == parsed` would then silently exclude low-confidence resumes, which is the
opposite of what `G-05` wants (they are usable *and* flagged). Every such consumer would need
updating, and the ones that were not would look correct.

### D8 — An Application is pinned to the resume version it was submitted with

The mirror of `hiring-postings` D9, for the other term of the same tuple. `domain-model.md`:
`RankingScore = f(resume_version, jd_version, prompt_template_version, model_version)`, and without
it "a score can't be explained or reproduced once the JD or the model changes underneath it."
`hiring-postings` D9 pinned `jd_version` on exactly that reasoning. `resume_version` needs the same
treatment or the tuple is half-guaranteed.

- A resume version is **immutable** once created. A new upload for the same candidate creates
  version *n+1* (`DOC-010`); it never overwrites.
- `is_active_version` marks one current version per candidate, for display and for new submissions.
- **An existing Application keeps the version it was submitted with.** Uploading a newer resume does
  not re-point it. Re-pointing is an explicit, permission-gated, audited human action.
- A superseded version remains retrievable forever, including after the posting closed (`RET-002`,
  `RET-003`).

**What re-pointing costs, and where the rule for it lives:** re-pointing invalidates any ranking
computed against the prior version. Whether that forces an automatic re-rank is
`matching-and-ranking`'s (`G-07` gives it ranking versions that preserve prior boards). This feature
records that the pin changed, with actor, reason and both versions, and emits nothing that
transitions anything.

**One reconciliation, because two adopted requirements collide on one case.** `DOC-010` says *every*
resume upload preserves a new version; `G-04` says an identical file under the same candidate creates
**no** new version. `G-04` is the later, TalentSphere-specific adoption and `AGENTS.md` gives the
exploration notes precedence — but the two do not actually conflict on intent: `DOC-010`'s purpose is
that an upload never *overwrites* a prior version, and linking to the existing identical version
overwrites nothing. Recorded so the divergence from `DOC-010`'s literal wording reads as a decision.

### D9 — `TS-BL-046` has no dependency on `TS-BL-043`, and that is correct on inspection

`D.10` gives `TS-BL-046` (identity and dedup) a single dependency: `TS-BL-041`. Not `TS-BL-043`.
That looks wrong, because `D23` says plainly that *"match inputs come from the **deterministic** half
of parsing (`D18`) — identity does not depend on model output."* Recorded here rather than silently
amended, the same treatment `ai-platform-governance` D9 gave the equivalent case.

**It is right, for a reason that is a decision rather than a technicality: `D18` makes manual entry
a first-class path, not a fallback hack.** Its words are that failed extraction means *"manual entry
required before the candidate record can be created."* So the identity model, the five-signal
evaluation, the review queue, the merge and the suppression list are all fully buildable and
testable against manually-entered contact fields, with no parser in the picture. `TS-BL-043` supplies
those fields automatically for the common case; it is not what makes them exist.

**So there is no dependency edge, but there is a preferred order:** build `TS-BL-043` before
`TS-BL-046` where staffing allows, so signal evaluation is written against real extracted output
rather than against a declared shape. If `TS-BL-046` goes first, it is written against
`TS-BL-043`'s declared output shape as an interface — the same stub-and-integrate pattern used
across Phase 1 — and costs one integration pass. This is a sequencing recommendation in the
Migration Plan, not a dependency; inventing an edge `D.10` does not carry would misrepresent what
blocks what.

*A consequence worth naming:* the manual-entry path is therefore not optional scope in
`TS-BL-046`, and not a later nicety. It is the path the item is verifiable through.

### D10 — Contact data separation is a storage split, not a display filter

`CAN-008` (*"candidate contact information shall be stored separately from scoring input data"*) and
`PRV-002` (*"contact information shall be separated from AI scoring input"*) are two statements of
one requirement, and `API-006` adds that contact fields are returned only to a permitted caller.
§12.2 gives `candidates.primary_email_encrypted` and `primary_phone_encrypted`, and `SEC-003`
requires the encryption.

**These are enforced by where the data lives, not by filtering on the way out** — the same principle
`config.yaml` already states for audit records: *"redaction is applied at write time, not as a
display filter."* Concretely: contact fields live on the candidate record in encrypted form and are
excluded from the extracted-text artifact and the chunk records that enrichment and embedding read.
A resolved reference handed to the gateway therefore *cannot* carry them (D3 point 2), because they
are not in the artifact being referenced.

*Why this matters beyond tidiness:* `matching-and-ranking` will embed the chunks
(`VEC-001`) into a vector store that `VEC-004` filters by authorization scope. Contact data inside
an embedded chunk is contact data inside a similarity index, retrievable by resemblance rather than
by permission — which no filter downstream can undo. Getting the split right here is the only place
it is cheap.

*The residual case, stated:* a resume's *body text* can contain an email address the extractor
missed. The split guarantees the structured contact fields are not in the scoring artifact; it does
not guarantee no personal data appears anywhere in resume prose, and nothing could, short of the
PII-detection engine the Non-Goals rule out. `PRV-001` is minimization, not elimination. This is
recorded so the guarantee is not read as stronger than it is.

### D11 — Two dependency asymmetries in `D.10`, neither of which is an error to correct

Both noticed while checking edges, and recorded rather than "fixed," because neither changes what
blocks what.

1. **`TS-BL-044` has no edge to `TS-BL-030` (prompt registry) while `hiring-postings`'
   `TS-BL-034` does**, for structurally identical work — both author a registered family's template
   content as a new version. The asymmetry is real and it is inconsequential: `TS-BL-030` is a
   Phase 1 item and `TS-BL-044` is Phase 2, so the registry lands first under every possible
   schedule whether or not the edge is drawn. `D.10` drew the edge in one place and not the other,
   and nothing reads it. Stated as an obligation in `tasks.md` — author against the registry, never
   by editing a version in place — which is how `hiring-postings` and
   `ai-platform-governance` both handled unlisted-but-real interfaces.
2. **`TS-BL-041` depends only on `TS-BL-001`, which is already done.** So this feature's entry item
   is unblocked today, like `ai-platform-governance`'s was. Worth checking rather than assuming,
   because an upload endpoint also needs the API ingress (`TS-BL-003`) and a database
   (`TS-BL-002`) — and **both are also done**, shipped by Sprint 0 and live in Dev. `D.10`'s single
   edge understates what is needed and overstates nothing; every implied prerequisite is satisfied.
   The one genuinely unsatisfied interface `TS-BL-041` needs is
   `access-control-and-admin`'s `TS-BL-018`, because `CAN-001` ("recruiters shall upload resumes only
   for authorized postings") resolves to the `assigned-postings` scope predicate. Built against its
   interface, per the Phase 1 pattern.

### D12 — What Sprint 0 and `talentsphere-wave-1-foundation` left for this feature: nothing

Checked rather than assumed, the same discipline `hiring-postings` D10 applied, and with the same
result.

- **`talentsphere-wave-1-foundation` has no `candidate/` delta spec.** Its `specs/` tree covers
  `access-control/`, `ai-platform/`, `design-system/`, `identity/` and `platform/`. Nothing needs
  redistributing into this feature, which is why `proposal.md` carries no "overlap to resolve"
  block and the five Phase 1 features all did.
- **No `D1`–`D18` decision from that change's `design.md` was allocated here.** `platform-core`
  `design.md` D11 distributed all eighteen across the five Phase 1 features.
- **Sprint 0 built no candidate or resume code.** Its handover states its ~2,000 lines are "all of
  it platform-layer; **no hiring feature exists yet**, by design," and lists the backend's
  directories as `core/`, `api/`, `db/`, `audit/`, `features/`, `runtime_config/`, `middleware/` and
  `migrations/`.
- **`KNOWN_ISSUES.md` carries nothing about resumes, parsing, candidates or object storage.** Its
  eight open items are environment provisioning, the unconfirmed Hubble contract, the not-yet-durable
  audit trail, the unexercised `audit_logs` grant, hand-applied schema ownership, `/build` metadata,
  a blocked npm registry, and Sprint 0's four endpoints declaring no permission. Two touch this
  feature the way they touch every feature: candidate endpoints need `TS-BL-018`'s permission
  mechanism, and material writes here need `TS-BL-020`'s durable audit writer — both already Phase 1
  items.

**Three Sprint 0 artifacts this feature consumes rather than rebuilds**, all of which happen to
matter more here than they did for `hiring-postings`: the correlation identifier that survives into
background tasks (this feature's work is almost entirely background tasks, and `ERR-006`/`NFR-007`
want the chain traceable from one identifier), the global error handler's redaction tests (which
already assert raw prompt content cannot reach a response — written before any prompt existed, and
directly relevant to a feature whose prompts contain resume text), and the audited
runtime-configuration registry (which is where §29 items 2, 3, 4, 5 and 12 live).

**One mislabel, recorded because `hiring-postings` D10 recorded the same one for §19.2.** The
candidate intake requirements are `reference/spec.md` **§19.3**, and document processing is **§15**;
**§16 is AI Implementation Requirements**. §16 is load-bearing here — §16.1's Resume Extraction row
and §16.3's `resume_extraction` contract are `TS-BL-044`'s — which is presumably how the two get
conflated. No shared document contains the error, so there is nothing to correct under `AGENTS.md`'s
convention; it is noted so the next reader looking for `CAN-001` in §16 does not conclude it is
missing.

### D13 — Three item-boundary refinements, recorded rather than assumed

All three under `D.11`'s permission for a feature's propose conversation to refine its internal item
boundaries. **None changes a dependency edge, and none moves work between features.**

1. **`TS-BL-041` creates the resume record with a sequential version number from the first
   upload.** `D.10`'s titles put versioning at `TS-BL-048`, but `TS-BL-044`'s already-fixed envelope
   takes `resume_version_ref: <id@version>` and sits four items earlier. `TS-BL-041` owns the version
   *row*; `TS-BL-048` owns immutability, the active version, and application pinning. Exactly the
   refinement `hiring-postings` D11 made for `jd_draft_ref`, for the same reason. See D3.
2. **`TS-BL-046` creates the `candidate_posting_applications` record.** `D.10`'s title names the
   identity model and dedup logic; the Application row is neither, but it cannot exist before the
   candidate does (D5) and it must exist the moment the candidate does, because `CAN-003` describes
   one pipeline that creates-or-updates a profile *and* links it. `CAN-005`, `CAN-007` and `G-10`
   (source metadata and application source type) come with it. No other item among the eighty
   creates this table, and `interview-pipeline` needs it to exist before it can put a stage on it.
3. **`TS-BL-043` carries `DOC-008`'s chunking as well as contact-field extraction.** `D.10`'s title
   names only the contact fields, but §25 lists "text chunks" among Resume Parsing's outputs
   alongside parsed JSON and confidence, and `TS-BL-044`'s evidence references have to point at
   something (D4). The alternative — chunking in `matching-and-ranking`'s embedding item — would put
   the evidence-reference target in a feature that lands after the claims that reference it.

## Risks / Trade-offs

**[`D23a` is treated as confirmed for the detection half, and an owner could still reject it.]** →
Mitigated by D1's scoping: the surface ships behind a feature flag, it triggers no transition, and
it introduces no schema that other items read. Rejection costs a flag flip and a deleted panel. The
residual risk is accepted deliberately and is the reason D1 exists rather than a silent build. What
is *not* mitigated, and correctly so, is the follow-on merge rule — it is excluded from scope and
named as an open question, because a rule invented here would be invented against a ranking model
this feature has never seen.

**[The dedup review queue may be noisier than anyone has estimated.]** `D23`'s three fuzzy signals
each raise a review item independently, which was the whole point of its revision — but nobody has
sized the false-positive rate, and a queue that is always full is a queue nobody works. → Partly
mitigated by the suppression list `D23` requires (a "not a duplicate" verdict is remembered, so a
pair never re-queues) and by `D07`'s volumes making the absolute count small. Not mitigated:
per-signal thresholds are not tunable in this scope, because tuning needs real data. Recorded as
the most likely thing to need revising after first contact with real resumes.

**[Untrusted input meets a language model for the first time, and the corpus that would prove the
isolation works lands in another feature.]** → Mitigated by the isolation being the gateway's,
already specified and asserted, rather than something this feature invents; by the prompt input
being narrowed to professional content (D3); and by the production promotion gate refusing
`resume_extraction`'s new template version until `TS-BL-032`'s corpus carries passing adversarial
cases for it. Not mitigated: Dev runs the family before those cases exist, which is the same
position `hiring-postings`' two families are in and which `ai-platform-governance` D8 argues is the
correct order.

**[A candidate can exist with no enrichment, and downstream features may not expect that.]** `D18`
made this decoupling deliberately — deterministic failure blocks the candidate, LLM failure does
not — so a candidate with contact fields, no skills and no seniority is a valid, expected state. →
Mitigated by stating it as a requirement rather than an accident, and by `G-03`'s insufficiency
markers making "not enriched" distinguishable from "enriched as empty." The residual risk is
downstream: `matching-and-ranking` ranking an unenriched candidate needs a defined answer, and that
is its feature's to give. Flagged here because the state originates here.

**[The malware scanner is the first external dependency other than Hubble and the model
provider, and it sits in front of everything.]** No clean verdict means no parse, no candidate, no
application. → Mitigated structurally: the scan is its own job type with §25's declared retry
policy, uploads queue rather than fail during an outage, and `G-12`/`NFR-004` keep every existing
candidate readable throughout. Not mitigated: sustained scanner unavailability halts *new* intake
entirely, which is the correct direction — `DOC-005` prohibits an unscanned file reaching parsing —
but it is a real single point of failure for one workflow, and it is worth an alert rather than a
growing queue nobody watches.

**[`candidate_id` nullable diverges from §12.2 and weakens a foreign key.]** A resume with no
candidate is now representable, so every consumer must handle it. → Mitigated by the two constraints
in D5 (an unresolved resume is not addressable as a `resume_version_ref` and cannot be an active
version), which are assertable, and by the state being explicit rather than inferred from a null.
The alternative — a staging table — was rejected in D5 for reasons that cost more.

**[Provisional conciseness bounds on enrichment fields may be wrong in both directions.]** Too tight
and a real role summary will not fit; too loose and the bound never fires, which reads identically
to compliance. → Mitigated on both sides the way `hiring-postings` mitigated it: bounds are audited
configuration rather than code, and `TS-BL-032`'s harness fails a family declaring a bound no case
exercises. This is the third feature to inherit `ai-platform-governance` D7's argument and the
mechanism is unchanged.

**[Two open decisions govern data this feature creates.]** `OD-004` (retention period) and `OD-009`
(deletion, access, correction, withdrawal) are owned outside engineering, and this is the feature
that creates candidate personal data. → Mitigated by building `RET-007`'s source linkage now —
derived data linked to its source for deletion and re-indexing — regardless of when the policy
lands, because retrofitting it means finding every derived record without a link. The interval
itself is configuration (§29 item 12). Not mitigated: no deletion or correction *workflow* is built,
so a request arriving before those items exist is handled manually. Recorded, not solved.

## Migration Plan

Greenfield feature on a live Dev environment with no candidate data. Order follows the backlog's own
dependency chain; each item is independently deployable, and every step is a reversible migration
plus a feature-flagged surface, per `config.yaml`'s standing rules.

1. **`TS-BL-041` first, and its dependencies are already satisfied.** `platform-core`'s
   `TS-BL-001`, `TS-BL-002` and `TS-BL-003` are all done and live in Dev (D11). Provision the private
   bucket with quarantine and resume prefixes, migrate `candidate_resumes` with `candidate_id`
   nullable, register the malware-scan job type, and ship the upload endpoint and intake screen
   behind a flag, disabled by default. §29 items 2, 3, 4 and 5 are seeded into the runtime-config
   registry in this step, so the scanner endpoint and the size limit are configuration from the
   first deploy rather than after the first surprise.
2. **`TS-BL-042` immediately after.** Hash on upload. Cheap, and it must precede `TS-BL-046` so the
   signal exists before the evaluator that reads it.
3. **`TS-BL-043` before `TS-BL-046` where staffing allows.** No dependency edge exists (D9), but
   building extraction first means signal evaluation is written against real output. If `TS-BL-046`
   goes first, it is written against `TS-BL-043`'s declared output shape and costs one integration
   pass. This is the one place in this feature's sequence where the recommended order and the
   dependency graph differ, so it is stated rather than left to staffing.
4. **`TS-BL-046`, then `TS-BL-048`.** Identity resolution creates the candidate and the Application;
   versioning then adds immutability, the active version and pinning. Between the two, a candidate
   can hold several resume rows with no active-version concept — correct and short-lived, and
   `TS-BL-048`'s migration marking the latest resolved resume active is a no-op on an empty table.
5. **`TS-BL-044`, then `TS-BL-045`.** Author `resume_extraction` as a **new template version**,
   leaving `TS-BL-030`'s stub active until a passing corpus run exists for the new one. Activation is
   a configuration change, so rollback is re-activating the prior version rather than a deployment.
   Confidence lands after, because a value published before enrichment existed would describe half a
   parse while appearing to describe the whole (D7).
6. **`TS-BL-047` last.** It needs `hiring-postings`' `TS-BL-037` as well as `TS-BL-046`, and it is
   the item whose underlying recommendation is unconfirmed — so it ships behind its own flag,
   disabled by default, and is enabled in Dev for evaluation. That flag is the mechanism D1's
   risk mitigation depends on, not a general precaution.
7. **Rollback** is redeploy-previous-artifact plus migration-down, with two asymmetries worth
   naming. Uploaded objects are not removed by a down-migration — they are quarantined or stored
   files, and deleting them on rollback would destroy the only copy of something a recruiter sent.
   And a completed merge is reversible by its own audited un-merge path (`D23`), not by rolling back
   the deployment that performed it.

## Open Questions

Each is genuinely deferrable — none changes the specs, the approach, or the task breakdown.

- **`D23a`'s follow-on application-merge rule** (D1(b)). *The headline open question of this
  feature.* When a confirmed duplicate is merged and the same candidate then holds two applications
  against one posting: which application survives, which ranking score is retained or whether a
  re-rank is forced, and what happens if the two sat at different stages. `D23` carries the same gap
  in wider form for candidates live in different pipelines. Two of the three inputs belong to
  unproposed features — ranking scores to `matching-and-ranking` (`TS-BL-051`/`TS-BL-052`, `G-07`)
  and stages to `interview-pipeline` — so the answer is theirs to shape. This feature records the
  condition and refuses to resolve it silently, which is what keeps the question answerable later
  at the cost of one resolution action against an existing record.
- **Whether `D23a`'s recommendation is ratified at all**, by whoever owns `D23`. D1 explains why
  building the detection half now is the cheaper bet in both directions. A rejection costs a flag
  flip.
- **Candidate data retention period** (`OD-004`, owned by Product, HR and Legal). The mechanism,
  §29 item 12's configuration key and `RET-007`'s source linkage all ship; the interval does not.
  `D22` caps configurable staleness windows by retention, so the two must be set consistently once
  this answers.
- **Candidate deletion, access, correction and withdrawal handling** (`OD-009`, owned by Legal, HR
  and Product). `PRV-003`–`PRV-005` require workflows; none is built here, and `RET-006` is not in
  any of the eighty items. Deferrable because `RET-007`'s linkage is what makes the eventual
  workflow implementable, and that is built now.
- **Per-field conciseness bounds on enrichment output.** `S.5` states the numbers were never given
  and should not be invented. Provisional values ship labelled provisional and change through the
  audited configuration path.
- **The malware scanner product itself**, its timeout, and whether it runs as a service the
  landing zone already provides. §29 item 5 makes the endpoint and timeout configuration, so this is
  an operational choice rather than a design one — the same treatment
  `ai-platform-governance` gave its rate-limit values.
- **Whether normalized-name matching needs transliteration or script folding beyond case and
  whitespace normalization.** `CAN-004` says "normalized name" without defining it, and `D23` uses
  it in two of five signals. A narrow normalization raises fewer false review items and misses more
  real duplicates; the right setting needs real candidate names. Deferrable because both
  behaviors are the same code path with a different normalizer, and `D23`'s signals are independent
  so a missed name match does not disable the others.
