## Purpose

The deterministic half of hybrid parsing: PDF text extraction, then rule-based recovery of the
identity-bearing fields candidate deduplication depends on, plus the traceable content sections that
later evidence references and embeddings point at. No language model participates. Owned by
`TS-BL-043`.

## ADDED Requirements

### Requirement: The identity-bearing fields are extracted deterministically

Email address, phone number, name, dates and links SHALL be extracted by deterministic rules over
the document's extracted text, and SHALL NOT be produced, corrected or supplemented by a language
model.

*Source: `D18` — the hybrid split exists so that "dedup keys come from deterministic logic rather
than model output. Candidate identity should not depend on a model's mood." `DOC-006`, `CAN-004`.*

#### Scenario: Contact fields recovered

- **WHEN** a clean resume containing an email address and phone number is parsed
- **THEN** both are extracted by deterministic rules and recorded on the resume record

#### Scenario: Model asked to supply an identity field

- **WHEN** a code path attempts to populate any of these fields from model output
- **THEN** the test suite fails

### Requirement: Extraction runs as a registered background job triggered by a clean scan

Deterministic extraction SHALL execute as a job type registered against the shared dispatch pattern,
triggered by a clean malware-scan verdict, with its own declared retry policy for transient parser
failure. It SHALL NOT run inside an interactive request.

*Source: `reference/spec.md` §25's Resume Parsing row (trigger "Clean PDF", "retry on transient
parser failure"); `NFR-002`, which names resume parsing asynchronous by ID;
`platform-core`'s `platform/async-orchestration`, whose purpose names resume parsing as an inheritor
of the shared pattern; `design.md` D2.*

#### Scenario: Triggered by a clean verdict

- **WHEN** a scan returns a clean verdict
- **THEN** the extraction job is dispatched through the shared pattern

#### Scenario: Transient parser failure

- **WHEN** extraction fails transiently
- **THEN** it is retried per its declared policy

#### Scenario: Duplicate delivery

- **WHEN** the same extraction event is delivered more than once
- **THEN** the resume record ends with one extraction result, not two

#### Scenario: Extraction attempted inline

- **WHEN** a code path performs document text extraction inside an interactive request
- **THEN** the test suite fails

#### Scenario: Retries exhausted

- **WHEN** extraction fails past its retry policy
- **THEN** the resume is retained with its failure reason and surfaced to the uploading recruiter
  and authorized administrators, rather than silently dropped

### Requirement: Extracted content is chunked into traceable sections

Extracted text SHALL be stored as sections, each carrying a source reference locating it within the
document, so that a later claim or embedding can cite the section it came from.

*Source: `DOC-008`; `reference/spec.md` §25, which lists text chunks among Resume Parsing's outputs;
`VEC-001`, which requires embedded resume sections to carry source identifiers; `design.md` D13,
which records why chunking belongs to this item rather than to the embedding item that consumes it.*

#### Scenario: Sections produced

- **WHEN** a resume is parsed
- **THEN** its content is stored as sections, each with a source reference

#### Scenario: Section cited later

- **WHEN** a downstream claim references a section
- **THEN** the reference resolves to that section's content and its position in the source document

### Requirement: The scoring artifact excludes the structured contact fields

The extracted-text artifact and the section records that downstream enrichment, embedding and
ranking read SHALL NOT contain the structured contact fields extracted by this capability.

*Source: `CAN-008`, `PRV-002`, `PRV-001`; `design.md` D10. Enforced by where the data is written, not
by filtering on read — a contact field inside an embedded section is retrievable by resemblance
rather than by permission, which no downstream filter can undo.*

#### Scenario: Sections inspected

- **WHEN** the stored sections for a parsed resume are inspected
- **THEN** the structured email, phone and name fields are absent from them

#### Scenario: Scoring artifact requested

- **WHEN** a downstream consumer requests the artifact used for scoring input
- **THEN** it receives content with the structured contact fields excluded

### Requirement: A document with no extractable text layer is detected and recorded, and no OCR runs

Where a document yields no usable text layer, the resume SHALL be recorded as requiring OCR with the
OCR status flag set, and no OCR SHALL be performed. The OCR enablement switch SHALL exist and SHALL
default to disabled.

*Source: `DOC-007`; `project.md` and `D18`'s consequences, both of which scope this product to OCR
readiness only; `reference/spec.md` §25, which lists OCR as its own job type, and §29 item 4, which
makes enablement configuration.*

#### Scenario: Scanned image PDF

- **WHEN** a clean PDF containing no text layer is parsed
- **THEN** the resume records that OCR would be required, sets the OCR flag, and no OCR is attempted

#### Scenario: OCR job registered

- **WHEN** the registered job types are enumerated
- **THEN** no OCR job type is registered

### Requirement: Absent contact fields block candidate creation and require manual entry

Where deterministic extraction yields none of the contact fields deduplication depends on, the resume
SHALL be recorded as requiring manual entry, and no candidate record SHALL be created from it until a
human supplies those fields.

*Source: `D18` — "deterministic extraction fails → contact fields empty → **manual entry required
before the candidate record can be created**, because dedup depends on those fields." `design.md` D5,
D9.*

#### Scenario: No contact fields found

- **WHEN** extraction completes and yields no email, phone or name
- **THEN** the resume is marked as requiring manual entry and no candidate record is created

#### Scenario: Manual entry supplied

- **WHEN** an authorized user supplies the missing contact fields
- **THEN** identity resolution becomes eligible to run and the entry is audited with its actor

#### Scenario: Candidate created without contact fields

- **WHEN** a code path attempts to create a candidate record with none of the contact fields present
- **THEN** it is refused

### Requirement: Extraction failure does not block the candidate where fields are present

Where deterministic extraction recovers the contact fields but fails on any other field, the resume
SHALL remain usable for identity resolution rather than being treated as a whole-document failure.

*Source: `D18`'s decoupling — "this usefully decouples the pipeline: a candidate can exist on
deterministic fields alone."*

#### Scenario: Partial extraction

- **WHEN** extraction recovers an email and phone but no dates or links
- **THEN** the resume is usable for identity resolution and the absent fields are recorded as absent
