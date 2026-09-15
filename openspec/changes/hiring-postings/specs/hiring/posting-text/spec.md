## Purpose

The external posting text as a distinct artifact from the internal job description — separate in
storage and in lifecycle, not a rendering of the JD — together with the second clearly-scoped AI
generation target that separation exists to enable. Owned by `TS-BL-039`.

## ADDED Requirements

### Requirement: Posting text is stored separately from internal job description content

External posting text SHALL be stored as its own artifact, distinct from the job description version's
content. Neither SHALL be derived from the other at read time.

*Source: `JOB-008`; `G-08` as adopted — "External posting text is a distinct artifact from the internal
job description; this session conflated them." The separation stands independently of whether AI ever
writes the text: posting copy can be typed by hand and this requirement still holds.*

#### Scenario: Posting text created

- **WHEN** posting text is written for a posting
- **THEN** it is stored against the posting as its own artifact

#### Scenario: Job description content unchanged

- **WHEN** posting text is written or edited
- **THEN** the referenced job description version's content is unchanged

#### Scenario: Internal content is not published as posting text

- **WHEN** a posting's external text is read
- **THEN** it returns the posting text artifact, never the internal job description content

### Requirement: Posting text has its own lifecycle and is not propagated to

Posting text SHALL be editable independently of the job description's approval state. Approving a new
job description version SHALL NOT alter, invalidate or regenerate a posting's text.

*Source: `JOB-008`, and `design.md`'s Non-Goals — building propagation would recreate the coupling
`JOB-008` exists to break. Regenerating posting text after a job description changes is a human action
with a visible prompt, not a rule.*

#### Scenario: New job description version approved

- **WHEN** a later job description version is approved
- **THEN** existing postings' text is unchanged

#### Scenario: Text edited on an open posting

- **WHEN** posting text is edited on an open posting
- **THEN** the edit is accepted and audited, without affecting the posting's pinned job description
  version

### Requirement: Posting text may be generated through the AI gateway

Posting text generation SHALL be invoked through the AI gateway using the `job_posting_generation`
family, taking the approved job description version reference and posting metadata reference as input.
It SHALL require the Run AI action, and its output SHALL be marked AI-generated until a human approves
it.

*Source: §16.1's Job Posting Text Generation row — trigger "posting preparation," inputs "approved JD
version, posting metadata," output "external posting draft," human gate "Human approval required";
§16.3's registered `job_posting_generation` family. `G-08` names this family as the reason the
separation is worth making. `design.md` D11 records this as a boundary refinement — D.10's item title
names only the separation, and no other item among the eighty covers this family's content.*

#### Scenario: Generation from an approved version

- **WHEN** an authorized user with Run AI generates posting text for a posting
- **THEN** the request is issued through the gateway carrying references, not content
- **AND** a run reference is returned

#### Scenario: Generation without Run AI

- **WHEN** a user who may edit posting text but does not hold Run AI triggers generation
- **THEN** the request is denied and no provider invocation occurs

#### Scenario: Generated text requires human approval

- **WHEN** posting text is generated
- **THEN** it is marked AI-generated and does not become the posting's approved text until a human
  records approval

#### Scenario: Posting opens with hand-written text

- **WHEN** posting text is written entirely by hand
- **THEN** the posting can open with no AI run having occurred

### Requirement: Generated posting text is bounded as a free-text field

The `job_posting_generation` output contract SHALL declare a length bound on its markdown body, and
output exceeding it SHALL be recorded as a failed run with no content written.

*Source: `S.5`'s standing bar; `ai-platform-governance`'s registry requirement that no family may
declare a free-text field without a bound. §16.3 requires "Markdown plus compliance text flags" here,
and unlike a job description this output genuinely is prose whose shaping is the point — so the model
authors the markdown and it is bounded directly, rather than being composed from bounded parts as
`design.md` D4 does for the job description. `design.md` D11 records why that answer does not
transfer.*

#### Scenario: Output exceeds the bound

- **WHEN** generated posting text exceeds its declared bound
- **THEN** the run is recorded as failed with the contract violation preserved
- **AND** no text is written to the posting

#### Scenario: Bound is configuration

- **WHEN** the bound is changed through the audited configuration path
- **THEN** subsequent runs validate against the new value with no deployment

### Requirement: Compliance flags are proposed, never set, by generation

Compliance text flags returned by generation SHALL be surfaced as a proposal for a human to confirm.
They SHALL NOT set a posting's compliance text status.

*Source: §16.3's "Markdown plus compliance text flags"; `G-01`'s adopted pattern — AI proposes,
a human confirms anything that blocks; `AI-010` and `project.md`'s central rule. The compliance status
field and its gate are `TS-BL-040`.*

#### Scenario: Flags returned

- **WHEN** generation returns compliance text flags
- **THEN** they are shown to the user as a proposal

#### Scenario: Status unchanged by generation

- **WHEN** generation completes
- **THEN** the posting's compliance text status is unchanged until a human records a decision

### Requirement: The template version is promoted, never edited in place

The authored `job_posting_generation` prompt content SHALL be introduced as a new version of the
already-registered family, activated through the audited configuration path.

*Source: `ai-platform-governance`'s prompt-registry requirements — a content change creates a new
version rather than modifying one, and production activation is refused without a recorded passing
corpus run. `TS-BL-030` registered this family with stub text; this is one of the two families Phase 2
authors.*

#### Scenario: New version created

- **WHEN** the authored prompt content is introduced
- **THEN** a new template version is created and the prior version remains retrievable

#### Scenario: Activation without a passing corpus run

- **WHEN** activation of the new version in production is attempted with no passing corpus run
  recorded for it
- **THEN** the registry refuses activation
