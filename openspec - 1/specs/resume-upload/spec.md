# Resume Upload Specification

## Purpose

Allow recruiters to upload candidate resume PDFs with validation, malware scanning, parsing, OCR readiness for scanned documents, and secure storage in GCS. Infected files MUST be deleted immediately; parsing MUST NOT occur before a clean scan result.

## Requirements

### Requirement: PDF-only upload

The system SHALL accept resume uploads only as PDF files up to 10MB via `POST /api/v1/candidates/:id/resumes`.

#### Scenario: Valid PDF upload

- **GIVEN** a recruiter uploads a valid PDF under 10MB
- **WHEN** the upload completes
- **THEN** the system MUST return HTTP 202 with a `resumeFileId` for status polling
- **AND** MUST enqueue async processing

#### Scenario: Reject non-PDF

- **GIVEN** a file that is not PDF (wrong MIME or magic bytes)
- **WHEN** upload is attempted
- **THEN** the system MUST return HTTP 400
- **AND** MUST NOT store the file

#### Scenario: Reject password-protected PDF

- **GIVEN** a password-protected PDF
- **WHEN** upload is attempted
- **THEN** the system MUST return HTTP 400 with a clear error message

### Requirement: Malware scan gate

Every uploaded file MUST pass malware scanning before parsing or permanent storage.

#### Scenario: Clean file

- **GIVEN** a uploaded PDF
- **WHEN** malware scan completes with clean result
- **THEN** `scan_status` MUST be `clean`
- **AND** parsing MAY proceed

#### Scenario: Infected file

- **GIVEN** a uploaded PDF
- **WHEN** malware scan detects infection
- **THEN** `scan_status` MUST be `infected`
- **AND** the blob MUST be deleted immediately
- **AND** an audit event MUST be recorded
- **AND** the recruiter MUST be notified of rejection

### Requirement: Parse and OCR readiness

The worker MUST extract text from PDFs and flag `ocr_required` when text density is below threshold.

#### Scenario: Text-based PDF

- **GIVEN** a clean PDF with extractable text
- **WHEN** parsing completes
- **THEN** `parse_status` MUST be `parsed`
- **AND** structured fields MUST be stored in `resume_versions.parsed_json`

#### Scenario: Scanned PDF

- **GIVEN** a clean PDF with low text density
- **WHEN** parsing detects insufficient text
- **THEN** `ocr_required` MUST be true
- **AND** Document AI OCR MUST be invoked
- **AND** extracted text MUST be stored before marking complete

### Requirement: Secure storage

Resume files MUST be stored in a private GCS bucket. Download MUST use short-lived V4 signed URLs for authorized users only.

#### Scenario: Authorized download

- **GIVEN** a recruiter with `candidates:view` on the candidate
- **WHEN** they request resume download
- **THEN** the system MUST return a signed URL expiring within 15 minutes
- **AND** MUST NOT expose a public bucket URL

### Requirement: Async status polling

Clients MUST poll `GET /api/v1/resumes/:id/status` for processing progress.

#### Scenario: Processing states

- **GIVEN** a resume in processing
- **WHEN** status is polled
- **THEN** response MUST include `scan_status` and `parse_status`
- **AND** MUST transition through `pending` → `clean` → `parsed` (or terminal failure states)
