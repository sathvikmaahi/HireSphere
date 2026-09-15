## Purpose

The pipeline that turns the traceable resume sections `candidate-intake` already produces into
vector records that trace back to them: source-identified, model-pinned, filterable, and
re-indexable when any of the four things `VEC-005` names changes. Owned by `TS-BL-049`.

## ADDED Requirements

### Requirement: Resume-derived sources are embedded with source identifiers

Embedding SHALL produce one vector record per embedded unit, carrying source type, source
identifier, source version where the source is versioned, and a content hash for traceability. The
source types embedded by this capability SHALL be the resume section and the candidate skill
summary.

*Source: `VEC-001`, which scopes embedding to "resume sections, candidate skill summaries, interview
notes, and scorecard summaries"; §12.2's `vector_index_records` and its `source_type` enum;
`C-01`'s resolution, which adopts §17 in full. `design.md` D2 records why the two interview-derived
source types in that enum are registered by `interview-pipeline` rather than embedded here.*

#### Scenario: Resume section embedded

- **WHEN** a resume section becomes available for embedding
- **THEN** a vector record is created carrying its source type, source identifier, source version
  and content hash

#### Scenario: Skill summary embedded

- **WHEN** a candidate skill summary is produced by enrichment
- **THEN** it is embedded as its own source type, distinguishable from a resume section

#### Scenario: Unsupported source type

- **WHEN** embedding is requested for a source type this capability does not register
- **THEN** the request is refused rather than embedded under an approximate type

### Requirement: This capability builds no second chunker

Embedded units SHALL be the traceable sections produced by deterministic extraction. This
capability SHALL NOT re-segment, re-parse, or otherwise derive its own document segmentation.

*Source: `DOC-008`'s traceable sections; `candidate-intake`'s `design.md` D13 and its
`candidate/resume-extraction` capability, which carries chunking deliberately so that "`TS-BL-044`'s
evidence references need a target" — the same target this capability embeds. §25 lists text chunks
among Resume Parsing's outputs, not among Embedding Generation's.*

#### Scenario: Section boundaries preserved

- **WHEN** a resume's sections are embedded
- **THEN** each vector record corresponds to one extracted section, with the same boundaries the
  extractor recorded

#### Scenario: Segmentation logic sought

- **WHEN** this capability's code is inspected for document segmentation logic
- **THEN** none exists

### Requirement: Every vector record traces back to the authoritative relational record

Each vector record SHALL identify the relational record and object-storage reference it was derived
from. The vector store SHALL NOT be treated as the source of truth for any value.

*Source: `VEC-002`; §12 as cited by `C-01` — "the vector store shall never be the only source of
truth"; `VEC-006`'s derived-data marking. `C-01`'s resolution states the division plainly: vectors
retrieve, the LLM ranks, and the relational database stays authoritative.*

#### Scenario: Record traced to its source

- **WHEN** a vector record is inspected
- **THEN** it names the relational record and storage reference it was derived from, and that
  reference resolves

#### Scenario: Derived data is marked as derived

- **WHEN** an embedded summary or its vector record is read
- **THEN** it is marked as derived data and does not replace or shadow the source record

#### Scenario: Source record deleted or superseded

- **WHEN** a source record referenced by a vector record is superseded
- **THEN** the vector record still names the version it was built from rather than the current one

### Requirement: The embedding model is pinned and recorded per record

Each vector record SHALL record the embedding model identifier used to produce it. The model SHALL
be resolved from configuration rather than defaulted in code, and SHALL NOT be changed implicitly.

*Source: `C-01`'s resolution, which names `embedding_model` pinning as added scope; §12.2's
`embedding_model` column; `VEC-005`, which makes an embedding-model change a re-index trigger — a
trigger that is only detectable if each record says which model produced it.*

#### Scenario: Model recorded

- **WHEN** a vector record is written
- **THEN** it carries the embedding model identifier resolved from configuration

#### Scenario: Records from two models coexist

- **WHEN** the configured embedding model changes and some records predate the change
- **THEN** each record still identifies the model that produced it, and mixed-model records are
  distinguishable

### Requirement: Filter attributes carry decision inputs, never a resolved verdict

Each vector record SHALL carry candidate identifier, posting identifier where applicable, practice,
source type and source version as filter attributes. Authorization metadata stored on a vector
record SHALL consist of the **inputs** to an authorization decision and SHALL NOT contain a
resolved allow or deny outcome.

*Source: `VEC-004`; `exploration-notes.md`
[Interaction A](../../../talentsphere/exploration-notes.md#interaction-a--permission-changes-now-trigger-vector-re-indexing),
which shows that a baked verdict would force a `VEC-005` re-index on every permission edit;
`access-control-and-admin`'s "Authorization decisions are never stored as resolved verdicts"
requirement, which names `TS-BL-050` as the consumer. `design.md` D3.*

#### Scenario: Filter attributes present

- **WHEN** a vector record is written
- **THEN** it carries candidate identifier, posting identifier where applicable, practice, source
  type and source version

#### Scenario: Stored authorization metadata inspected

- **WHEN** authorization metadata on a vector record is inspected
- **THEN** it contains identifiers and attributes only
- **AND** it contains no resolved allow or deny outcome

#### Scenario: Permission matrix change

- **WHEN** an administrator changes the permission matrix
- **THEN** no vector record requires re-embedding for authorization to be correct

### Requirement: Embedding runs as a registered job type and registers no queue of its own

Embedding SHALL execute as a background job type registered against the platform's single dispatch
pattern, declaring its retry policy, carrying the originating correlation identifier, and applying
idempotency at the effect boundary. This capability SHALL NOT implement its own dispatch mechanism.

*Source: §25's Embedding Generation row — triggered by "parsed content or note submitted," retrying
on transient vector or embedding failure; `platform-core`'s async-orchestration requirement, whose
suite fails on background work started outside the shared pattern; `platform-core` D6's
at-least-once delivery, which makes idempotency a requirement rather than a precaution.*

#### Scenario: Embedding dispatched

- **WHEN** parsed resume content becomes available
- **THEN** embedding is dispatched as a registered job type through the shared dispatch pattern

#### Scenario: Duplicate delivery

- **WHEN** the same embedding request is delivered more than once
- **THEN** exactly one vector record set results for that source and version

#### Scenario: Dispatch outside the shared pattern

- **WHEN** any code path in this capability starts embedding work outside the shared dispatch
  pattern
- **THEN** the test suite fails

#### Scenario: Transient embedding failure

- **WHEN** embedding fails transiently
- **THEN** it is retried per the declared policy, and exhausted retries leave the source record
  intact with its failure reason visible

### Requirement: Re-indexing is supported for parsing, model and content changes

The system SHALL support re-indexing when parsing logic, the embedding model, or source content
changes, in bounded batches, without deleting the source records. Re-indexing SHALL be an audited
operation and SHALL be re-runnable.

*Source: `VEC-005`; §29 item 15's vector re-indexing batch size; `C-01`'s resolution, which records
the re-index path as an **ongoing obligation** rather than one-time setup. `design.md` D3 records
that the fourth `VEC-005` trigger — authorization metadata change — is designed out rather than
implemented, because no verdict is stored.*

#### Scenario: Embedding model changed

- **WHEN** the configured embedding model changes
- **THEN** affected records are identifiable and re-indexable without touching source records

#### Scenario: Source content re-parsed

- **WHEN** a resume's sections are re-produced by changed parsing logic
- **THEN** its vector records are re-indexed and the superseded records are replaced rather than
  duplicated

#### Scenario: Batch size respected

- **WHEN** a re-index runs
- **THEN** it processes in batches of the configured size and reports progress

#### Scenario: Authorization change does not re-index

- **WHEN** authorization metadata for a role or user changes
- **THEN** no re-index is required or triggered

### Requirement: Nothing unscanned, unresolved or contact-bearing is embedded

Embedding SHALL NOT occur for a document that has no clean malware verdict, for a resume that has
not resolved to a candidate, or over content containing the structured candidate contact fields.

*Source: `DOC-005`'s ordering, asserted at both ends per `candidate-intake`'s upload requirements;
that feature's `design.md` D5, under which an unresolved resume "is not addressable as a
`resume_version_ref` … and not enrichable"; and its D10, which states the reason for the contact
exclusion in this capability's own terms — "a contact field inside a section becomes a contact field
inside a similarity index once `TS-BL-049` embeds it, which no downstream filter can undo."
`PRV-001`, `PRV-002`, `CAN-008`.*

#### Scenario: Unscanned document

- **WHEN** embedding is attempted for a document with no clean scan verdict
- **THEN** it is refused and the refusal is recorded

#### Scenario: Unresolved resume

- **WHEN** embedding is attempted for a resume not yet resolved to a candidate
- **THEN** it is refused

#### Scenario: Contact field in embedded content

- **WHEN** content destined for embedding is inspected
- **THEN** the structured email, phone and name fields are absent

### Requirement: Every derived vector record is linked back to its source for deletion

Each vector record SHALL be discoverable from the resume version it was derived from, so that
retention and deletion of a source can reach everything derived from it.

*Source: `RET-007`'s source linkage, which `candidate-intake` built "now rather than when `OD-004`
and `OD-009` land, because retrofitting it means finding every derived record without a link."
Vector records are the largest class of derived record in the product, and this capability is where
that reasoning is either honoured or broken.*

#### Scenario: Deriving records from a source

- **WHEN** the derived records of a resume version are enumerated
- **THEN** every vector record built from it is returned

#### Scenario: Source disposal

- **WHEN** a source record is disposed of under retention policy
- **THEN** its vector records are reachable for disposal by the same linkage, and the disposal is
  recorded
