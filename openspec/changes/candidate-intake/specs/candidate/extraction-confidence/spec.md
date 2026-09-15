## Purpose

The third state between a parse that worked and a parse that failed: per-section confidence values
and insufficiency markers spanning both halves of a hybrid parse, a record-level roll-up, and the
recruiter review a low-confidence parse drives. Owned by `TS-BL-045`.

## ADDED Requirements

### Requirement: Confidence is recorded per section and rolled up on the record

Parsed resume data SHALL carry confidence values per extracted section together with
insufficient-information markers, and the resume record SHALL carry a single roll-up confidence value
derived from them.

*Source: `G-05`, `DOC-009` — confidence *values*, plural, plus insufficient-information markers;
§12.2's `candidate_resumes.extraction_confidence` for the roll-up; `design.md` D7. A single
record-level number cannot say which field to check, which is the operational purpose `G-05` gives
it.*

#### Scenario: Mixed-quality parse

- **WHEN** a resume parses with a clear contact block and an unreadable employment history
- **THEN** the sections carry different confidence values and the roll-up reflects the weakest
  material section rather than an average that hides it

#### Scenario: Roll-up readable by a downstream consumer

- **WHEN** a downstream consumer reads the resume record
- **THEN** a single roll-up confidence value is available on it

#### Scenario: Review surface reads sections

- **WHEN** a recruiter opens a low-confidence resume for review
- **THEN** the per-section values identify which sections to check

### Requirement: Confidence spans both halves of the parse

The confidence recorded for a resume SHALL account for both deterministic extraction and model
enrichment. A confidence value SHALL NOT be published for a resume whose enrichment has neither
completed nor failed.

*Source: `D18`'s hybrid split; `design.md` D7 — a value published while enrichment is still
outstanding describes the deterministic half alone while occupying the field a consumer reads as the
parse's confidence, leaving a provisional number indistinguishable from a final one.*

#### Scenario: Enrichment still pending

- **WHEN** deterministic extraction has completed and enrichment has not yet run
- **THEN** the resume reports confidence as not yet determined rather than reporting the
  deterministic value as the parse's confidence

#### Scenario: Enrichment failed

- **WHEN** enrichment fails permanently for a resume
- **THEN** a confidence value is published that accounts for the absent enrichment

### Requirement: Low confidence is a review state, not a parse failure

A resume whose confidence falls below the configured threshold SHALL remain in the parsed state and
SHALL additionally carry a review state with its reason. Low confidence SHALL NOT be represented as a
parsing failure or as an additional parsing status value.

*Source: `G-05` — "a third state between 'parsed' and 'failed'"; §12.2's four-value `parsing_status`;
`design.md` D7. `parsing_status` answers whether the parse ran; confidence answers whether to trust
it, and conflating them breaks `D18`'s decoupling under which a candidate can exist on deterministic
fields alone.*

#### Scenario: Low-confidence parse

- **WHEN** a resume parses with confidence below the threshold
- **THEN** its parsing status is parsed, and it additionally carries a review state naming the reason

#### Scenario: Consumer filtering on parsing status

- **WHEN** a consumer selects resumes whose parsing status is parsed
- **THEN** low-confidence resumes are included, because they are usable and flagged rather than
  failed

#### Scenario: Additional status value introduced

- **WHEN** the parsing status enumeration is inspected
- **THEN** it carries only pending, parsed, failed and unsupported

### Requirement: Low-confidence parses reach a recruiter review queue

Resumes in the low-confidence review state SHALL appear in a review queue visible to the uploading
recruiter and to authorized users, stating which sections are low-confidence or insufficient, and
SHALL support correction of the parsed values by a permitted human.

*Source: `G-05` — confidence "drives recruiter review of low-confidence parses"; `DOC-011`;
`reference/spec.md` §36's low-quality-parsing mitigation, which names recruiter review and manual
correction; `UI-003`'s requirement that a surface show state, owner, next action and blockers.*

#### Scenario: Queue populated

- **WHEN** a resume enters the low-confidence review state
- **THEN** it appears in the review queue with the sections needing attention named

#### Scenario: Human corrects a parsed value

- **WHEN** a permitted user corrects a low-confidence parsed value
- **THEN** the correction is recorded with actor and timestamp, the prior value is preserved, and the
  resume leaves the review state

#### Scenario: Unauthorized access to the queue

- **WHEN** a user without permission requests the review queue
- **THEN** the request is denied rather than returning an empty list

### Requirement: The confidence threshold is audited configuration

The threshold below which a parse is treated as low-confidence SHALL be held in the audited
runtime-configuration registry, changeable without a deployment, and carry machine-readable
provenance marking its initial value provisional.

*Source: `config.yaml`'s runtime-configuration rule; `S.5`'s position that concrete numbers were never
given and should not be invented, applied by `ai-platform-governance` `design.md` D7 — a
wrong-but-labelled-and-enforced threshold is fixable configuration, an absent one is unenforceable.*

#### Scenario: Threshold changed

- **WHEN** an authorized administrator changes the threshold
- **THEN** the change takes effect without a deployment and is recorded with actor, reason, and
  previous and new value

#### Scenario: Threshold change does not rewrite history

- **WHEN** the threshold is changed
- **THEN** resumes already reviewed and corrected are not returned to the review state by the change
  alone
