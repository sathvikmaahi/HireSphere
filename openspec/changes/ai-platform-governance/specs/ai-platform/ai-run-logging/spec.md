## Purpose

The permanent record of every AI invocation, of every human decision that diverges from one, and
of every occasion AI output was shown to a person before they formed their own judgement. This
history cannot be reconstructed after the fact, which is why it exists before the first AI feature
ships.

## ADDED Requirements

### Requirement: Every AI run is logged, and logging precedes invocation

Every AI run SHALL record model provider, model name, model version, prompt template identifier
and version, input object references, output object references, token usage where available,
safety flags, status, start and completion timestamps, and failure detail where applicable. The
run record SHALL be written **before** the provider is invoked. If the record cannot be written,
the provider SHALL NOT be called.

*Source: `AI-002`; `glossary.md`'s `AIRun` entry — "cannot be backfilled — this is why the AI
governance substrate must exist before the first real AI call, not after"; inherited `D9`, which
notes that logging after the fact loses exactly the runs most worth having, the ones that crashed.
Ordering the write first converts "we log AI calls" from a convention into an invariant.*

#### Scenario: Successful run

- **WHEN** an AI run completes successfully
- **THEN** a run record exists carrying provider, model, model version, template version, input and
  output references, and timing

#### Scenario: Failed run

- **WHEN** an AI run fails
- **THEN** the run record is retained with status failed and the failure detail preserved

#### Scenario: Logging is not optional

- **WHEN** a run cannot be logged
- **THEN** the run is not issued to the provider

#### Scenario: No path bypasses the log

- **WHEN** invocation paths are enumerated
- **THEN** every one writes a run record before reaching the provider, and a test fails if a path
  is added that does not

### Requirement: Run records are append-only

Run records SHALL be append-only. The application database role SHALL hold insert and select
rights on the run log and SHALL NOT hold update or delete rights.

*Source: `config.yaml`'s standing rule that "audit and AI run tables are append-only: the
application role holds INSERT and SELECT only"; `access-control-and-admin`'s D7, which makes the
same guarantee for `audit_logs` and establishes that a grant, not application convention, is what
enforces it. `platform-core`'s D3 two-identity split is what keeps that grant from being
decorative.*

#### Scenario: Update attempt on a run record

- **WHEN** the application role attempts to update or delete a run record
- **THEN** the database refuses the statement

#### Scenario: A later-known fact is not written onto a run

- **WHEN** a fact about a run becomes known after the run completed
- **THEN** it is recorded as a separate linked record rather than by amending the run

### Requirement: Reference-only inputs

Run records SHALL store references to source objects rather than raw sensitive content. Resume
text, candidate contact details, and other personal data SHALL NOT be stored in the run record.

*Source: `PRV`-level minimization, `D16`, `D6`, and `glossary.md`'s evidence-source labeling entry,
which states the reason plainly: this is "required so an Admin's audit-log access (config-only)
doesn't become a PII back door through AI run logs."*

#### Scenario: Administrator reads AI run logs

- **WHEN** a user with AI run log access reads a run record for a candidate-related run
- **THEN** the record identifies the source objects by reference
- **AND** no resume text or candidate contact data is present in the payload

#### Scenario: Reconstructing a run's inputs

- **WHEN** an investigator needs the actual inputs behind a run
- **THEN** the references resolve to the source records, and reading those records requires
  permission on them

### Requirement: Safety flags

Safety and policy filter results SHALL be recorded on the run.

*Source: `AI-002`'s safety-flag field; `C-09`'s adopted security coverage, which makes injection
and unsafe-output detection a tested concern rather than an incidental one.*

#### Scenario: Safety filter triggers

- **WHEN** a safety or policy filter flags a run
- **THEN** the flag and its category are recorded on the run record and the run is retrievable by
  that flag

### Requirement: Reproducibility tuple

Each run SHALL be reconstructible from the recorded combination of input references, prompt
template version, and model version.

*Source: `domain-model.md`'s ranking provenance —
`RankingScore = f(resume_version, jd_version, prompt_template_version, model_version)` — and its
statement that without the full tuple "a score can't be explained or reproduced once the JD or the
model changes underneath it". `G-07`, `D12`'s fork-on-edit rule which exists for the same reason.*

#### Scenario: Explaining a past output

- **WHEN** a past AI output is questioned
- **THEN** the run record identifies which input versions, template version, and model version
  produced it

#### Scenario: Inputs changed underneath a past run

- **WHEN** a source record referenced by a run has since been versioned forward
- **THEN** the run still names the version it actually consumed, not the current one

### Requirement: AI content labelled until approved

Content produced by an AI run SHALL be marked as AI-generated or AI-assisted, and SHALL retain
that marking until a human user approves it.

*Source: `AI-001`, `RANK-004`. `design-system`'s AI-disclosure component renders this marking; the
marking itself, and the approval that clears it, are recorded here.*

#### Scenario: Unapproved AI content

- **WHEN** AI-produced content is stored or displayed before human approval
- **THEN** it carries an AI-generated or AI-assisted marker

#### Scenario: Approved content

- **WHEN** an authorized human approves AI-produced content
- **THEN** the approval is recorded with actor and timestamp, and the content is no longer
  presented as unreviewed

### Requirement: Disclosure records

The system SHALL record, separately from the run and linked to it, each occasion on which AI
output was disclosed to a human actor: the actor, the output disclosed, the time, and the context
of disclosure. Disclosure records SHALL be append-only and SHALL NOT modify the run.

*Source: `D05`, whose accepted trade-off — full AI context shown to an Interviewer before the
interview — carries the agreed mitigation that "every `AIRun` records that the score was visible
pre-interview, so a later review can account for it." That mitigation cannot be a field on the run:
the run is written before invocation and is append-only, while disclosure happens later and may
happen many times. It is therefore a separate linked record. See `design.md` D11, and the dated
correction block added to `D05` in `exploration-notes.md`. `matching-and-ranking`'s `TS-BL-054`
owns the visibility *rules*; this requirement is the substrate that makes them reviewable.*

#### Scenario: AI output shown before an interview

- **WHEN** AI ranking output is presented to an actor in a pre-interview context
- **THEN** a disclosure record is written naming the actor, the output, the time, and the context
- **AND** the run record is unchanged

#### Scenario: Multiple disclosures of one output

- **WHEN** the same AI output is disclosed to several actors on several occasions
- **THEN** each disclosure is recorded separately, and all are retrievable against the run

#### Scenario: Reviewing anchoring after the fact

- **WHEN** a reviewer asks whether a human judgement was formed before or after seeing AI output
- **THEN** the disclosure records answer it from stored data alone

### Requirement: Human override records

The system SHALL provide a first-class override record for any human decision that departs from AI
output. An override SHALL require a reason and SHALL NOT delete or overwrite the original AI
output.

*Source: `G-06`, `RANK-006`, `BR-016`. The reason requirement matches `access-control-and-admin`'s
mandatory-reason rule on permission overrides — in both cases the record exists to make a
discretionary act reviewable rather than to prevent it.*

#### Scenario: Override recorded

- **WHEN** a user overrides AI output
- **THEN** an override record captures the actor, timestamp, reason, the original AI output
  reference, and the human value
- **AND** the original AI output remains retrievable

#### Scenario: Override without a reason

- **WHEN** an override is submitted with no reason
- **THEN** it is rejected

#### Scenario: Divergence is measurable

- **WHEN** override records are queried over a period
- **THEN** the rate at which humans diverged from AI output can be computed from the stored records
  alone

### Requirement: Output feedback

The system SHALL provide a mechanism for users to flag AI output as inaccurate, incomplete,
biased, unsafe, or not useful, and SHALL retain feedback linked to the run.

*Source: `AI-011`. Feedback is also the intake path for `ai-platform/ai-evaluation`'s corpus
maintenance requirement — a real failure found in use becomes a corpus case.*

#### Scenario: Flagging output

- **WHEN** a user flags an AI output with a category and optional comment
- **THEN** the feedback is stored against the run and is retrievable for governance review

#### Scenario: Feedback does not mutate output

- **WHEN** feedback is submitted
- **THEN** the original output is unchanged

### Requirement: Correlation with the audit trail

AI run records SHALL carry the same correlation identifier as the audit records produced by the
same activity, so a single investigation can span both.

*Source: `reference/spec.md` §26 Audit Correlation; `access-control-and-admin`'s audit-review
capability fixes the shared identifier and states that this feature's `TS-BL-028` owns the run
record. `platform-core` assigns the identifier at the edge and carries it across asynchronous
boundaries, which is what makes it available to a background AI run at all.*

#### Scenario: Tracing an AI-triggered change

- **WHEN** an investigator holds the correlation identifier from an audit record produced by a
  service-account action
- **THEN** the corresponding AI run record can be retrieved using that identifier

#### Scenario: Correlation survives background dispatch

- **WHEN** an AI run executes as background work triggered by an interactive request
- **THEN** its run record carries the correlation identifier of the originating request

### Requirement: Run log access and search

AI run records SHALL be searchable by authorized users, filterable by family, template version,
model version, status, safety flag, and time range, and SHALL be readable by the Auditor role
without granting access to candidate personal data.

*Source: `C-02`'s read-only Auditor role; `D16`'s config-only Administrator. The reporting surface
built on this search is `insight-and-reporting`'s `TS-BL-074`; this requirement is the queryable
substrate, not a dashboard.*

#### Scenario: Auditor reviews AI activity

- **WHEN** an Auditor searches AI run records
- **THEN** matching records are returned with governance metadata and no candidate personal data

#### Scenario: Unauthorized access

- **WHEN** a user without AI run log permission calls the run log endpoint
- **THEN** the request is denied

### Requirement: Retention

AI run records SHALL be retained long enough to support auditability, debugging, and model
governance review, according to configured policy, and their disposal SHALL itself be recorded.

*Source: `RET`-level policy, which defers the interval to company policy;
`access-control-and-admin` applies the same mechanism-specified, interval-deferred treatment to
audit retention. `D22` caps configurable windows by the retention period, so the two cannot be set
inconsistently.*

#### Scenario: Retention policy applied

- **WHEN** the retention policy is configured
- **THEN** run records are retained for at least that period and their disposal is itself recorded
