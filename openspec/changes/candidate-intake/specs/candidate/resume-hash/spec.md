## Purpose

The document-level deduplication signal: a hash computed over the bytes a recruiter actually
uploaded, stored and indexed so that identical documents can be recognized. It produces the signal
and decides nothing with it. Owned by `TS-BL-042`.

## ADDED Requirements

### Requirement: The hash is computed over the bytes as uploaded

A resume's file hash SHALL be computed over the uploaded bytes, before the document is promoted out
of quarantine and independently of the scan verdict, and SHALL be stored on the resume record.

*Source: `G-04`, `CAN-004`; §12.2's `candidate_resumes.file_hash`; `design.md` D6. Hashing what the
recruiter sent rather than what storage holds means the signal is available even for a document that
never becomes clean.*

#### Scenario: Hash computed on upload

- **WHEN** an upload passes synchronous validation
- **THEN** its hash is computed and stored before any scan verdict is known

#### Scenario: Hash on a quarantined document

- **WHEN** a document is quarantined with an unsafe verdict
- **THEN** its hash is still recorded

#### Scenario: Identical bytes

- **WHEN** the same file is uploaded twice
- **THEN** both resume records carry the same hash

### Requirement: The hash is queryable across the whole candidate database

A lookup by hash SHALL return the resume records carrying it, together with the candidate each is
attached to where one is attached, so that a caller can distinguish a same-candidate match from a
cross-candidate match.

*Source: `G-04`, whose two cases differ only by whether the matching record belongs to the same
candidate; `CAN-004`, which lists resume hash among the signals deduplication evaluates.*

#### Scenario: Lookup with a same-candidate match

- **WHEN** a hash is looked up and a matching resume belongs to the same candidate
- **THEN** the result identifies the match and the candidate it belongs to

#### Scenario: Lookup with a cross-candidate match

- **WHEN** a hash is looked up and a matching resume belongs to a different candidate
- **THEN** the result identifies the match and the differing candidate

#### Scenario: Lookup with an unattached match

- **WHEN** a hash is looked up and a matching resume has no candidate attached yet
- **THEN** the result reports the match with no candidate, rather than omitting it

### Requirement: This capability takes no deduplication action

The hash capability SHALL NOT merge candidates, create or suppress resume versions, or raise review
items. It exposes the signal; the identity capability evaluates it and the versioning capability acts
on it.

*Source: `design.md` D6 — `G-04`'s single sentence names a computation, a decision and a consequence,
and `D.10` places them in three items. Stated as a prohibition so no item half-builds another's third.*

#### Scenario: Cross-candidate hash match found

- **WHEN** a lookup finds the same hash under a different candidate
- **THEN** this capability records the fact and performs no merge

#### Scenario: Same-candidate hash match found

- **WHEN** a lookup finds the same hash under the same candidate
- **THEN** this capability performs no version suppression itself

### Requirement: A known-unsafe document is refused without a second scan

Where an uploaded file's hash matches a document already recorded as unsafe, the upload SHALL be
refused immediately with the scan-failure reason rather than dispatched for scanning again.

*Source: `DOC-003`, `DOC-005`; `design.md` D6, which records this as a convenience that removes a
pointless round trip and explicitly not as a security control — a hash is trivially changed, and
anything not already known-unsafe is still scanned.*

#### Scenario: Re-upload of a known-unsafe file

- **WHEN** a file whose hash matches a quarantined unsafe document is uploaded
- **THEN** it is refused with the recorded scan-failure reason and no scan job is dispatched

#### Scenario: A file not known to be unsafe

- **WHEN** a file whose hash matches nothing unsafe is uploaded
- **THEN** it is dispatched for scanning normally, and no hash match substitutes for a scan
