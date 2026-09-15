## Purpose

The one way a resume enters TalentSphere: a posting-scoped upload by an authorized recruiter, a
validation set that refuses anything that is not a readable unencrypted PDF within limits, and a
malware scan that must return clean before the document is parsed, embedded, prompted or displayed.
Owned by `TS-BL-041`.

## ADDED Requirements

### Requirement: Upload is scoped to a posting the uploader is authorized for

A resume upload SHALL be accepted only against a specific job posting, and only from an actor whose
verdict from the central permission evaluator permits it for that posting. The verdict SHALL come
from the evaluator, never from logic local to this capability.

*Source: `CAN-001`, `API-002`, `SEC-005`. "Authorized postings" resolves to
`access-control-and-admin`'s `assigned-postings` scope predicate over
`job_postings.recruiter_ids[]`, per `C-03` as resolved — writes are assignment-scoped.*

#### Scenario: Recruiter uploads to an assigned posting

- **WHEN** a Recruiter assigned to a posting uploads a resume against it
- **THEN** the upload is accepted

#### Scenario: Recruiter uploads to an unassigned posting

- **WHEN** a Recruiter not assigned to a posting uploads a resume against it
- **THEN** the upload is denied server-side, and the denial is not merely a hidden control in the
  interface

#### Scenario: Upload with no posting

- **WHEN** an upload is submitted with no posting reference
- **THEN** it is rejected, since a resume has no valid intake context outside a posting

### Requirement: Only readable, unencrypted PDFs within the configured limits are accepted

The upload SHALL reject a file whose extension is not PDF, whose MIME type is not an allowed PDF
type, which is encrypted, which is structurally unreadable as a PDF, or which exceeds the configured
size limit. These checks SHALL complete within the upload request itself.

*Source: `DOC-001`, `DOC-002`, `DOC-003`, `CAN-002`; `design.md` D2, which splits `CAN-002`'s six
checks into five synchronous and one asynchronous.*

#### Scenario: Non-PDF extension

- **WHEN** a file with a non-PDF extension is uploaded
- **THEN** it is rejected in the response with a field-level validation error

#### Scenario: PDF extension with a non-PDF MIME type

- **WHEN** a file named with a `.pdf` extension is uploaded with a MIME type outside the allowed set
- **THEN** it is rejected, and the extension alone does not qualify it

#### Scenario: Encrypted PDF

- **WHEN** an encrypted PDF is uploaded
- **THEN** it is rejected, and no attempt is made to extract its content

#### Scenario: Oversize file

- **WHEN** a file larger than the configured size limit is uploaded
- **THEN** it is rejected, naming the limit

### Requirement: Structural validation does not extract text

The synchronous readability and encryption checks SHALL open the document structure only, under a
declared time and memory bound, and SHALL NOT extract document text. Exceeding the bound SHALL
reject the upload rather than continuing.

*Source: `design.md` D2. Read literally, `CAN-002`'s "PDF readability" check invites synchronous
parsing, which §25 and `NFR-002` prohibit. Full text extraction is the parsing job's work, over an
attacker-influenced file at the maximum permitted size.*

#### Scenario: Malformed document that consumes the bound

- **WHEN** a file's structure cannot be read within the declared bound
- **THEN** the upload is rejected as unreadable rather than the request continuing to consume
  capacity

#### Scenario: Structural check on a valid document

- **WHEN** a valid PDF passes the structural check
- **THEN** no document text has been extracted at that point

### Requirement: The malware scan is a registered background job, not inline work

The malware scan SHALL execute as a job type registered against the shared dispatch pattern, with
its own declared retry policy, and SHALL NOT run inside the upload request.

*Source: `reference/spec.md` §25, which gives Malware Scan its own trigger ("Resume upload") and
retry policy ("retry on scanner outage, do not parse until clean"); `platform-core`'s
`platform/async-orchestration` requirement that a feature registers against the shared pattern
rather than building dispatch; `design.md` D2. The scanner endpoint and timeout are configuration
(§29 item 5).*

#### Scenario: Scan dispatched

- **WHEN** an upload passes synchronous validation
- **THEN** the scan is dispatched through the shared pattern and the interactive request completes
  promptly

#### Scenario: Scanner unavailable

- **WHEN** the malware scanner is unavailable
- **THEN** the scan is retried per its declared policy, the upload is retained pending a verdict,
  and existing candidate records remain fully readable

#### Scenario: Dispatch built locally

- **WHEN** a code path in this capability starts the scan outside the shared dispatch pattern
- **THEN** the test suite fails

### Requirement: An unsafe or unscanned document reaches nothing downstream

A document SHALL be held in a quarantine location on upload and SHALL be promoted to the resume
object location only on a clean scan verdict. A document without a clean verdict SHALL NOT be sent
to parsing, chunking, OCR, embedding, a prompt, or any interface that renders its content.

*Source: `DOC-004`, `DOC-005`, `SEC-006`, `SEC-004`; `design.md` D2. `DOC-004`'s "stored only after
secure validation and scanning" is satisfied by the resume record's object reference being populated
only after a clean verdict, which is what makes an asynchronous scan possible at all.*

#### Scenario: Unsafe verdict

- **WHEN** the scan returns an unsafe verdict
- **THEN** the document remains quarantined, its scan status records the verdict, and no parsing,
  chunking, embedding or prompt path can read it

#### Scenario: Clean verdict

- **WHEN** the scan returns a clean verdict
- **THEN** the document is promoted to the resume object location and parsing becomes eligible to run

#### Scenario: Parsing attempted before a verdict

- **WHEN** a parse is attempted on a document whose scan status is not clean
- **THEN** it is refused and the refusal is recorded

#### Scenario: Object access

- **WHEN** any actor reads a stored resume object
- **THEN** access is signed or proxied through the application, and no object is publicly readable

### Requirement: The upload returns a job identifier and reports progress asynchronously

The upload response SHALL carry a job identifier, and the intake surface SHALL report scan and
parsing progress against it rather than blocking.

*Source: `API-007`, `NFR-002`, `NFR-001`; `platform-core`'s async-orchestration status requirement.*

#### Scenario: Upload accepted

- **WHEN** an upload passes synchronous validation
- **THEN** the response carries a job identifier and the intake screen returns within the
  interactive performance threshold

#### Scenario: Progress queried

- **WHEN** the intake surface queries the job identifier
- **THEN** the current scan and parsing state is returned, with a failure reason where one failed

### Requirement: Intake failures are distinguishable by kind

An intake failure SHALL be reported as one of validation, scan, parsing, OCR or storage failure,
with a retry path where the failure is safely retryable and a stated reason where it is not.

*Source: `ERR-005`, `ERR-003`, `DOC-011`, `NFR-006`.*

#### Scenario: Two different failures

- **WHEN** one upload fails validation and another fails scanning
- **THEN** the two report distinguishable failure kinds rather than one generic upload error

#### Scenario: Failure visible with retry

- **WHEN** a document processing failure occurs
- **THEN** it is visible to the uploading recruiter and to authorized administrators, with retry
  offered where the failure is retryable

### Requirement: A resume record exists from the first upload and is addressable by version

Each accepted upload SHALL create a resume record carrying its uploader, source posting, original
file name, size, MIME type, storage reference, scan state and a sequential version number, and that
record SHALL be addressable as identifier-and-version.

*Source: `RET-002`; §12.2's `candidate_resumes`; `design.md` D3 and D13 — `TS-BL-044`'s already-fixed
envelope takes `resume_version_ref: <id@version>` and sits earlier than the versioning item, so the
version row is created here.*

#### Scenario: First upload for a new person

- **WHEN** a resume is uploaded for someone with no existing candidate record
- **THEN** a resume record is created with a sequential version number and no candidate reference

#### Scenario: Record addressable

- **WHEN** a resume record exists
- **THEN** it can be addressed as identifier-and-version by a caller that needs to reference a
  specific version

### Requirement: Upload limits and scanner settings are audited configuration

The permitted file size limit, the allowed PDF MIME types, the OCR enablement switch, and the
malware scanner endpoint and timeout SHALL be held in the audited runtime-configuration registry and
changeable without a deployment. No unaudited setter SHALL exist for any of them.

*Source: `reference/spec.md` §29 items 2, 3, 4 and 5; `config.yaml`'s rule that runtime
configuration changes are audited with a mandatory reason.*

#### Scenario: Size limit changed

- **WHEN** an authorized administrator changes the file size limit
- **THEN** the change takes effect without a deployment and is recorded with actor, reason and
  previous and new value

#### Scenario: Unaudited setter

- **WHEN** a code path sets any of these values outside the audited configuration path
- **THEN** the test suite fails

### Requirement: Upload is rate-limited and audited

The upload endpoint SHALL enforce a rate limit, and every accepted upload SHALL produce an audit
record in the same transaction as the resume record, carrying references rather than candidate
personal data. A failed audit write SHALL fail the upload.

*Source: `SEC-013`, `API-003`, `config.yaml`'s audit rules, `D6`'s references-not-personal-data
constraint.*

#### Scenario: Rate limit exceeded

- **WHEN** uploads from one actor exceed the configured rate
- **THEN** further uploads are refused with a stated reason until the window resets

#### Scenario: Audit write fails

- **WHEN** the audit record for an upload cannot be written
- **THEN** the upload fails and no resume record persists

#### Scenario: Audit content

- **WHEN** an upload's audit record is inspected
- **THEN** it carries the resume and posting references and contains no candidate name, contact
  detail or resume text
