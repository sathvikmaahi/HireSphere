## Purpose

The authority for where an AI-surfaced claim came from: a closed source vocabulary, the
evidence-reference model behind each claim, and permission-scoped resolution of a reference to the
record it names. It is what lets a reader see that a claim rests on a resume rather than an earlier
interview — and what keeps AI run logs from becoming a route to personal data.

## ADDED Requirements

### Requirement: The source vocabulary is closed and defined once

The system SHALL define one closed vocabulary of evidence sources: resume, interview note,
scorecard, and human decision. No other component SHALL define, extend, or derive a source value.

*Source: `G-02`, `UI-005`, `BR-008`, `glossary.md`'s evidence-source labeling entry.
`design-system`'s evidence-source label component specifies that it "renders the source value it is
given without interpreting or deriving it" — that non-derivation clause is only satisfiable if
exactly one component does the deriving, which is this one.*

#### Scenario: Vocabulary is authoritative

- **WHEN** any component needs a source value
- **THEN** it uses one from this vocabulary rather than defining its own

#### Scenario: Unknown source value

- **WHEN** a claim carries a source value outside the vocabulary
- **THEN** it is rejected rather than stored or rendered

### Requirement: Every claim carries a source label and a resolvable reference

Every AI-surfaced claim about a candidate SHALL carry a source label from the vocabulary and at
least one evidence reference identifying the record the claim rests on, including that record's
version where the record is versioned.

*Source: `AI-007`, `G-02`, `RANK-002`, which requires the ranking board to display evidence
references alongside fitment and gap summaries. Versioned references matter because `D12` forks a
job description on edit and resumes are versioned per `D18` — an unversioned reference would
resolve to content the claim was never made about.*

#### Scenario: Claim with evidence

- **WHEN** an AI-surfaced claim is stored
- **THEN** it carries a source label and at least one evidence reference

#### Scenario: Claim about a versioned record

- **WHEN** a claim rests on a record that has versions
- **THEN** the reference names the version the claim was made against

#### Scenario: Insufficiency instead of an unsupported claim

- **WHEN** no evidence supports a conclusion
- **THEN** an insufficiency result is recorded identifying what was missing, rather than a claim
  with no reference

### Requirement: Reference resolution is permission-scoped

Resolving an evidence reference to the record it names SHALL be evaluated on that record's own
terms. A reader SHALL see the label and the reference regardless, and SHALL be denied the
referenced content without permission on it.

*Source: `D16`'s config-only Administrator, `C-02`'s read-only Auditor, and
`access-control-and-admin`'s matching rule for audit records — "access to that record is evaluated
on its own terms and denied without the required permission". Governance metadata and the data it
points at are separately permissioned, which is what makes a governance role useful without making
it a data role.*

#### Scenario: Auditor reads a labelled claim

- **WHEN** an Auditor reads a claim carrying a resume reference
- **THEN** the label and the reference identifier are returned
- **AND** the resume content itself is not

#### Scenario: Recruiter resolves the same reference

- **WHEN** a user holding permission on the referenced record resolves the reference
- **THEN** the referenced record is returned

#### Scenario: Reference to a record the reader cannot see

- **WHEN** a reader without permission resolves a reference
- **THEN** the request is denied, and the denial does not disclose the referenced record's contents

### Requirement: Labels and references carry no personal data

A source label and an evidence reference SHALL consist of a source value, a record identifier, and
a version where applicable. Neither SHALL embed resume text, candidate contact details, or any
other personal data.

*Source: `glossary.md`'s evidence-source labeling entry states the purpose exactly — labeling is
"required so an Admin's audit-log access (config-only) doesn't become a PII back door through AI
run logs." A reference that carried a snippet of what it points at would reopen the hole the
reference-only rule closes, through a smaller opening.*

#### Scenario: Reference payload inspected

- **WHEN** a stored evidence reference is inspected
- **THEN** it contains an identifier and version only, with no excerpt of the referenced content

#### Scenario: Governance surface carries no personal data

- **WHEN** claims and their labels are read through a governance surface
- **THEN** no personal data is present in the response

### Requirement: A human decision is a source like any other

A human decision SHALL be representable as an evidence source, so that a claim resting on a prior
human judgement is distinguishable from one resting on a document.

*Source: `G-02`'s four-way vocabulary, and `G-06`'s override records — the human value that
replaced an AI output is itself evidence a later claim may rest on. `D17`/`D21`'s priority-lane
carry-forward makes this concrete: a carried scorecard is cited as prior evidence for a new
posting, and a reader needs to see that it is carried rather than freshly observed.*

#### Scenario: Claim resting on a prior decision

- **WHEN** a claim rests on a recorded human decision
- **THEN** its source label identifies it as a human decision and its reference names that record

#### Scenario: Carried evidence is distinguishable

- **WHEN** evidence originating on a different Application is cited
- **THEN** the reference identifies the record it came from, so its origin is visible rather than
  implied

### Requirement: Mixed-source claims are separable

Where a single insight presents claims from more than one source, each claim SHALL carry its own
label and references rather than the insight carrying one label for all of them.

*Source: `G-02`, which notes this "makes the `D05` mitigation real — an interviewer can see which
claims rest on the resume versus an earlier interview". `design-system`'s mixed-evidence scenario
requires per-claim labels at the presentation layer, which is only possible if they are stored per
claim.*

#### Scenario: Insight drawing on two sources

- **WHEN** an insight contains claims drawn from a resume and from an interview note
- **THEN** each claim carries its own source label and references

#### Scenario: Available before its consumers ship

- **WHEN** this feature is deployed
- **THEN** the vocabulary, the reference model, and permission-scoped resolution exist and are
  exercisable
- **AND** the screens and families that consume them arrive in later features
