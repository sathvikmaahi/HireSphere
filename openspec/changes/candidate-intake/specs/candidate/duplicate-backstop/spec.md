## Purpose

The fallback for a duplicate the submission-time check missed: a posting-scoped comparison of the
candidates already on one posting, using the richer parsed signals available by then, surfaced as an
advisory warning a recruiter actions. It never merges anything and it blocks nothing. Owned by
`TS-BL-047`.

**The recommendation this capability implements is recorded in `exploration-notes.md` `D23a` as
"recommendation made, not yet confirmed" — the only unconfirmed decision this feature rests on.**
`design.md` D1 records the explicit call: the two-check recommendation is treated as confirmed on
stated grounds, and `D23a`'s follow-on application-merge rule is treated as genuinely open and
excluded from this capability's scope.

## ADDED Requirements

### Requirement: Comparison is scoped to one posting and uses parsed signals

Duplicate detection at this stage SHALL compare only the candidates holding applications against a
single posting, using the parsed signals available after extraction and enrichment — employer history,
education, dates and skill fingerprint — rather than repeating the submission-time contact-field
check.

*Source: `D23a`'s second check — "when ranking a posting, compare the candidates *within that posting*
for near-duplicates using the **parsed** signals now available"; `D07`, under which a posting holds
tens of candidates so pairwise comparison is affordable.*

#### Scenario: Two near-duplicate candidates on one posting

- **WHEN** two candidates on one posting share employment history, education and dates while
  differing on every contact field
- **THEN** the pair is surfaced as a possible duplicate

#### Scenario: Near-duplicate candidates on different postings

- **WHEN** two near-duplicate candidates hold applications against different postings
- **THEN** this capability raises nothing, since its scope is within a posting

#### Scenario: Contact-field match

- **WHEN** two candidates on one posting share a contact field
- **THEN** that is the submission-time check's outcome and this capability does not duplicate it

### Requirement: The result is a non-blocking advisory warning

A possible duplicate SHALL be surfaced as a warning on the posting's candidate list identifying the
counterpart candidate, and SHALL NOT block ranking, shortlisting, scheduling, selection or any other
action.

*Source: `D23a` — "surfaced as a **non-blocking warning on the ranked list** ('possible duplicate of
#7'), actioned by the Recruiter, never auto-merged at this stage — consistent with the advisory-only
principle"; `project.md`'s central rule; `G-01`'s adopted pattern that AI or heuristic output proposes
and a human confirms anything that blocks.*

#### Scenario: Warning surfaced

- **WHEN** a possible duplicate is detected on a posting
- **THEN** the affected entries carry a warning naming the counterpart, and every action on both
  remains available

#### Scenario: Warning dismissed

- **WHEN** a recruiter dismisses a warning
- **THEN** the pair is recorded as not a duplicate through the same remembered-verdict mechanism the
  submission-time review queue uses, and the warning does not reappear

#### Scenario: Warning acted on

- **WHEN** a recruiter confirms a warning is a real duplicate
- **THEN** the pair enters the candidate-level review and merge path rather than being merged here

### Requirement: This capability merges nothing

Detection SHALL NOT merge candidates, alter an application, change a ranking, or write any
workflow-governed state field.

*Source: `D23a` — "never auto-merged at this stage"; `config.yaml`'s rule that no surface writes a
state field directly. Merge remains the candidate-identity capability's, performed by a human decision
through its audited reversible path.*

#### Scenario: Merge path sought

- **WHEN** a persistence path from detection output to a candidate merge or an application field is
  sought
- **THEN** none exists

#### Scenario: High-similarity pair

- **WHEN** two candidates on a posting are near-identical on every parsed signal
- **THEN** the warning is raised and no merge occurs, regardless of how strong the similarity is

### Requirement: The unresolved-application condition is recorded, not resolved

Where a confirmed duplicate is merged and the merged candidate is left holding more than one
application against the posting, this capability SHALL record and surface that condition for human
resolution and SHALL NOT decide which application survives, which ranking is retained, or whether a
re-rank occurs.

*Source: `D23a`'s follow-on rule, which states that these "all need explicit rules" and which nothing
has written; `design.md` D1(b), which records that two of the three inputs — ranking scores and
application stages — belong to `matching-and-ranking` and `interview-pipeline`, both unproposed.*

#### Scenario: Confirmed duplicate merged

- **WHEN** a confirmed duplicate is merged and two applications against one posting result
- **THEN** the condition is recorded, surfaced to the posting's owner and recruiters, and both
  applications remain exactly as they were

#### Scenario: Resolution rule inferred

- **WHEN** a code path selects a surviving application, retains a ranking, or triggers a re-rank in
  response to this condition
- **THEN** the test suite fails, since no such rule has been decided

#### Scenario: Condition visible until resolved

- **WHEN** the condition has been recorded and not yet resolved by a human
- **THEN** it remains visible as a blocker on the posting rather than expiring silently

### Requirement: Detection runs on the posting's candidate set rather than on submission

Detection SHALL run against a posting's current candidate set, be re-runnable, and SHALL NOT be
triggered per submission — the submission-time check already covers that path.

*Source: `D23a`'s two checks at different depths — check one runs "on every submission against the
whole candidate database," check two runs "when ranking a posting."*

#### Scenario: New candidate added to a posting

- **WHEN** a candidate is added to a posting with existing candidates
- **THEN** detection is eligible to run over the posting's set, comparing the new entry against the
  others

#### Scenario: Detection re-run

- **WHEN** detection is run twice over an unchanged posting
- **THEN** the same pairs are reported once each, and already-dismissed pairs are not re-raised

### Requirement: The capability ships behind its own feature flag, disabled by default

This capability SHALL be gated by a declared feature flag, disabled by default, so it can be enabled
for evaluation and withdrawn without a deployment. While disabled, its surface SHALL answer as an
absent capability.

*Source: `config.yaml`'s feature-flag rules — flags are declared, an unknown flag raises, and a
disabled capability answers 404; `ENG-009`. `design.md` D1 makes this flag the mitigation for
`D23a` being unconfirmed rather than a general precaution: rejection of the recommendation costs a
flag flip.*

#### Scenario: Flag disabled

- **WHEN** the flag is disabled
- **THEN** no warning is computed or rendered and the surface answers as absent

#### Scenario: Flag enabled in one environment

- **WHEN** the flag is enabled in Dev and disabled elsewhere
- **THEN** the behavior differs by environment with no deployment difference

#### Scenario: Flag removed while referenced

- **WHEN** the flag is not present in the declared registry
- **THEN** resolution raises rather than defaulting to disabled
