## Purpose

The compliance text status on a posting, its configured default, and the open-validation gate that
reads it — built now and defaulting to not-required, so nothing is blocked while legal sign-off on
jurisdictional disclosure text remains open. Owned by `TS-BL-040`.

## ADDED Requirements

### Requirement: Compliance text status is a field on the posting with three values

A posting SHALL carry a compliance text status of missing, valid or not required.

*Source: `G-09` as adopted, naming `job_postings.compliance_text_status: missing | valid |
not_required`; §12.2's column. `G-09`'s stated purpose is turning "the jurisdictional AI-disclosure
exposure into a concrete field with a gate" rather than an unresolved risk.*

#### Scenario: Status readable on a posting

- **WHEN** a posting is read
- **THEN** its compliance text status is one of the three values

#### Scenario: Status outside the vocabulary

- **WHEN** a value outside the three is written
- **THEN** it is rejected

### Requirement: The status is stamped at creation from configured policy

A posting's compliance text status SHALL be set at creation from the configured default, and SHALL
thereafter be the posting's own field rather than a value computed at read time.

*Source: `G-09`'s decision to build the field now with the default at not-required; `design.md` D8,
which records that a computed-at-read status cannot answer the audit question the field exists for —
whether a posting was validated as compliant against a *specific* required text.*

#### Scenario: Posting created under the shipped default

- **WHEN** a posting is created while the configured default is not required
- **THEN** its status is stamped not required

#### Scenario: Status is a stored fact

- **WHEN** a posting's status is read after the configuration has changed
- **THEN** the stored value is returned, not a recomputed one

### Requirement: The default is audited runtime configuration, not a code constant

The compliance-text default and the required disclosure text SHALL be held in the audited runtime
configuration registry. Changing either SHALL be an audited change carrying a mandatory reason and
previous and new value, and SHALL require no deployment.

*Source: `G-09`'s stated intent — "When Legal defines the required text, the default flips to
`missing` and the gate begins enforcing — a configuration change, not a rebuild"; §29 item 11, which
already lists "compliance disclosure text for job postings" among the items configurable without code
changes; `config.yaml`'s standing rule for the runtime configuration registry. See `design.md` D8.*

#### Scenario: Default flipped to missing

- **WHEN** the configured default is changed from not required to missing
- **THEN** postings created afterwards are stamped missing
- **AND** the change is audited with previous value, new value and reason

#### Scenario: No deployment needed

- **WHEN** the configuration is changed
- **THEN** the new value takes effect without a deployment

#### Scenario: Unaudited setter does not exist

- **WHEN** the configuration path is enumerated
- **THEN** no setter exists that changes these values without writing an audit record

### Requirement: Open validation checks compliance text status

Posting-open validation SHALL reject an open transition when the posting's compliance text status is
missing. A status of valid or not required SHALL satisfy the check.

*Source: `JOB-009`, which names compliance text among the items open validation verifies; `G-09`'s
gate. The gate is part of `TS-BL-038`'s checklist; this capability supplies the value it reads.*

#### Scenario: Opening with status missing

- **WHEN** opening is requested on a posting whose compliance text status is missing
- **THEN** the transition is rejected, naming compliance text among the unsatisfied items

#### Scenario: Opening with status not required

- **WHEN** opening is requested while the status is not required
- **THEN** the compliance check passes

#### Scenario: Opening with status valid

- **WHEN** a human has recorded the required disclosure text as present and the status is valid
- **THEN** the compliance check passes

### Requirement: A configuration change never transitions a posting that is already open

Changing the compliance configuration SHALL NOT alter the state of a posting that is already open.
Postings whose stored status does not satisfy the current configuration SHALL be surfaced for review
by their owners.

*Source: `design.md` D8. Re-evaluating open postings against new configuration would mean a
configuration change silently transitioning live postings out of open — an automatic transition on a
governed state field, which `hiring/posting-lifecycle` prohibits and which `project.md`'s central rule
prohibits generally: material state changes rest with an accountable human.*

#### Scenario: Default flipped while postings are open

- **WHEN** the configured default is flipped to missing while postings are open
- **THEN** no open posting changes state

#### Scenario: Non-compliant open postings are reviewable

- **WHEN** open postings no longer satisfy the current compliance configuration
- **THEN** they are listed for their owners with the reason stated

#### Scenario: A human resolves each one

- **WHEN** an owner acts on a listed posting
- **THEN** the resulting close or amendment is an ordinary permission-gated, audited action

### Requirement: A status change is a recorded human decision

Setting a posting's compliance text status to valid SHALL be an explicit action by an authorized
actor, recorded with actor and timestamp. No AI output SHALL set the status.

*Source: `G-01`'s adopted pattern — AI proposes, a human confirms anything that blocks; `AI-010`;
`hiring/posting-text`'s requirement that generated compliance flags are a proposal. This is the
posting-side half of that pairing.*

#### Scenario: Human marks compliance text valid

- **WHEN** an authorized actor records the required disclosure text as present
- **THEN** the status becomes valid with actor and timestamp recorded

#### Scenario: AI-proposed flags do not set the status

- **WHEN** posting text generation returns compliance flags
- **THEN** the status is unchanged until a human records the decision

#### Scenario: Unauthorized status change

- **WHEN** a user without the required permission attempts to change the status
- **THEN** the request is denied
