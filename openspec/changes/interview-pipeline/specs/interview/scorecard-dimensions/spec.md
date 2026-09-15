## Purpose

The evaluation structure every scorecard uses: eight fixed dimensions and a five-point scale, global
rather than per-job-description, and seeded as audited configuration because the business that
specified them has not approved them. Owned by `TS-BL-063`.

## ADDED Requirements

### Requirement: Scorecard dimensions are the fixed set

Every scorecard SHALL be structured around role fitment, technical depth, innovation practice
relevance, project relevance, communication evidence, risk areas, gap closure status and overall
recommendation. A dimension outside the configured set SHALL be rejected rather than stored.

*Source: `SCR-003`, adopted by `C-06`'s resolution of 2026-08-14 which **amends `D15`**: "AI-suggested
competencies are dropped. Scorecard dimensions (`SCR-003`, fixed)". `C-06`'s supporting argument: with
interview notes embedded (`C-01`), semantic retrieval "works materially better against a stable field
schema than against per-JD-varying competency names".*

#### Scenario: Scorecard structured

- **WHEN** a scorecard is generated
- **THEN** it carries the eight configured dimensions

#### Scenario: Dimension outside the set

- **WHEN** generated output supplies a dimension outside the configured set
- **THEN** it is rejected as a contract violation

#### Scenario: AI-suggested competencies sought

- **WHEN** this capability is inspected for AI-proposed or per-role competency generation
- **THEN** none exists

### Requirement: Dimensions are global and are never scoped to a job description version

The dimension set SHALL be the same for every job description, posting and Application. Editing or
forking a job description SHALL NOT change any scorecard's structure, and no scorecard SHALL pin to a
job description version for its dimensions.

*Source: `C-06`'s explicit consequence — "`D15`'s consequence *'competencies become part of the JD
version'* is **superseded** — dimensions are global, not JD-scoped, so editing a JD no longer touches
scorecard structure." This reverses a `D15` consequence that `hiring-postings`' fork-on-edit behaviour
would otherwise have to honour. `C-06` also closed the competency-vocabulary-seeding open item for the
same reason.*

#### Scenario: Two postings compared

- **WHEN** scorecards for Applications against two different job descriptions are compared
- **THEN** both use the same dimension set

#### Scenario: Job description forked

- **WHEN** an approved job description is edited and forks to a new version
- **THEN** no scorecard's structure changes and no scorecard requires regeneration for that reason

#### Scenario: JD-version pin sought

- **WHEN** a scorecard is inspected for a job-description-version reference determining its dimensions
- **THEN** none exists

### Requirement: The rating scale is five-point and shared with interview notes

Dimension ratings and the overall recommendation SHALL use the five-point scale `strong_yes`, `yes`,
`maybe`, `no`, `strong_no`. The same scale SHALL apply to an interview note's recommendation.

*Source: `D15`'s rating-scale resolution of 2026-08-14, confirmed unchanged by `C-06` ("Rating scale
unchanged from `D15`"), and §12.2, which carries the enum on both `interview_notes.recommendation` and
`scorecards.overall_recommendation`. `exploration-notes.md`'s Open questions records the scorecard
rating scale as resolved by this pair.*

#### Scenario: Rating supplied

- **WHEN** a rating outside the five values is supplied on a dimension or an overall recommendation
- **THEN** it is rejected

#### Scenario: Scales compared

- **WHEN** a note's recommendation scale and a scorecard's are compared
- **THEN** they are the same five values

### Requirement: The dimension set is seeded audited configuration, not a compiled constant

The dimension set, its labels and the rating scale SHALL be held in the platform's audited runtime
configuration, seeded with the specified values. Any change SHALL be audited with previous value, new
value and a mandatory reason. No unaudited setter SHALL exist, and no dimension list SHALL be compiled
into the application.

*Source: `C-06`'s `OD-007` carry-forward — "the dimension set is unapproved even in their organisation,
so expect business stakeholders to revise it. Treat `SCR-003` as a seeded default ... not as settled
product truth"; `OD-007` itself assigns approval of scorecard dimensions and weightings to Innovation
Practice and Recruitment Leadership; `ENG-010`'s seed scripts. **`C-06` cited "§29 #14" as the
configuration vehicle; §29 item 14 is *role and permission matrix values* and §29 carries no
scorecard-dimension item at all** — corrected in `exploration-notes.md` during this feature's propose
conversation, with `platform-core`'s audited runtime-configuration registry named as the correct
vehicle. `design.md` D14(b).*

#### Scenario: Dimension renamed

- **WHEN** an authorized administrator renames a dimension
- **THEN** the change is audited with previous and new value and a mandatory reason, and takes effect
  with no deployment

#### Scenario: Compiled list sought

- **WHEN** the codebase is inspected for a hard-coded dimension list
- **THEN** none exists

#### Scenario: Unaudited setter sought

- **WHEN** the codebase is inspected for a path changing the dimension set without an audit record
- **THEN** none exists

#### Scenario: Historical scorecards after a change

- **WHEN** a dimension is renamed after scorecards have been recorded against it
- **THEN** existing scorecards remain readable as they were recorded

### Requirement: No dimension weighting is applied and none is configured with a value

Scorecard dimensions SHALL carry no weights, and no aggregate numeric score SHALL be computed from
them. Any weighting mechanism introduced later SHALL require an approved set of values before use.

*Source: `OD-007` covers dimensions **and weightings**, and approves neither. `C-05`'s reversal of
`D11` rests on consolidation not being averaging, and a weighted aggregate would be averaging with
extra steps. The same treatment `matching-and-ranking` gave §29 item 8's ranking weights, which are
configurable "where product approved" with no approval in existence: ship the mechanism's absence
honestly rather than provisional numbers that read as decided.*

#### Scenario: Weighting sought

- **WHEN** this capability is inspected for a dimension weight or an aggregate numeric score
- **THEN** none exists

#### Scenario: Overall recommendation derivation

- **WHEN** the overall recommendation's provenance is inspected
- **THEN** it is an AI-proposed, human-decided value rather than a computation over dimension ratings

### Requirement: The dimension set's unapproved status is visible rather than implicit

The seeded dimension set SHALL carry machine-readable provenance marking it as unapproved by the
business, and that provenance SHALL be discoverable by an operator inspecting the configuration.

*Source: `OD-007`; `C-06`'s instruction to treat `SCR-003` as a seeded default "not as settled product
truth"; `ai-platform-governance` `design.md` D7's precedent, where provisional conciseness bounds ship
carrying machine-readable provenance rather than as bare numbers — "a wrong-but-labelled-and-enforced
bound is fixable configuration; an absent one is an unenforceable contract".*

#### Scenario: Provenance inspected

- **WHEN** an operator inspects the seeded dimension configuration
- **THEN** its unapproved provenance is present and machine-readable

#### Scenario: Approval recorded later

- **WHEN** the business approves the set
- **THEN** the provenance is updatable through the same audited path

### Requirement: The organisation-specific dimension is flagged for confirmation rather than silently kept

`innovation practice relevance` SHALL be seeded as specified and SHALL be recorded as pending
confirmation or rename, discoverable without reading the source specification.

*Source: `C-06`'s closing note — "`innovation practice relevance` is Miracle-Labs-specific; confirm it
applies here or rename" — carried as an open item in `exploration-notes.md`. `design.md` Open Questions
records why it ships as specified rather than renamed on a guess: renaming a value in audited
configuration is a configuration change; inventing a replacement is a product decision this feature
does not own.*

#### Scenario: Seeded as specified

- **WHEN** the dimension set is seeded
- **THEN** `innovation practice relevance` is present as specified

#### Scenario: Pending status discoverable

- **WHEN** the seeded configuration is inspected
- **THEN** the dimension is marked as pending confirmation or rename

### Requirement: Interview note fields and scorecard dimensions are separate fixed sets

The eight structured interview note fields and the eight scorecard dimensions SHALL be maintained as
distinct sets. Changing one SHALL NOT change the other.

*Source: `C-06`'s resolution, which fixes both separately — `SCR-003` for scorecard dimensions and
`INT-006` for note fields — and closes the "note templates per interview type" item on `INT-006`'s
side. The two sets overlap in subject and are not the same list; conflating them would make a note
field change silently restructure every scorecard.*

#### Scenario: Sets compared

- **WHEN** the note field set and the dimension set are compared
- **THEN** they are separately maintained

#### Scenario: One set changed

- **WHEN** a note field is renamed through the audited path
- **THEN** the scorecard dimension set is unchanged
