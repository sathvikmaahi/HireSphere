## Purpose

Retrieval stage one: filtered nearest-neighbour lookup over the vector index that finds the
evidence a ranking run will reason about, applying the same authorization rules as relational
access and evaluating them at query time so a permission change never invalidates the index. Owned
by `TS-BL-050`.

## ADDED Requirements

### Requirement: Vectors retrieve, the ranking model ranks, and the relational store stays authoritative

Retrieval SHALL return candidate evidence for a ranking task and SHALL NOT produce a score, an
ordering presented as a ranking, or any value stored as authoritative. Every value retrieved SHALL
be resolvable to its authoritative relational record.

*Source: `C-01`'s resolution — "Vectors **retrieve**, the LLM **ranks**, and the relational database
stays authoritative" — and §12 as quoted there: "the vector store shall never be the only source of
truth." `VEC-002`, `VEC-006`. `design.md` D2 records the two-stage split and why collapsing it would
make similarity a scoring signal by accident.*

#### Scenario: Retrieval result inspected

- **WHEN** a retrieval result is returned
- **THEN** it carries evidence references and source labels, and no score or rank

#### Scenario: Similarity is not a score

- **WHEN** ranking output is traced to its inputs
- **THEN** no similarity distance appears as a ranking signal or a persisted score component

#### Scenario: Retrieved value read as authoritative

- **WHEN** a retrieved chunk's content is needed
- **THEN** it is read from the authoritative relational or object-storage record the vector record
  names, not from the index

### Requirement: Retrieval applies the same authorization rules as relational access

Vector retrieval SHALL apply the authorization rules that govern relational access to the same
records. The verdict SHALL be produced by the central permission evaluator, and the evaluator's
scope filter SHALL be applied as a query-time predicate rather than by discarding retrieved
results.

*Source: `VEC-003`; `AUTHZ-002`/`AUTHZ-003`'s single server-side evaluator;
`access-control-and-admin`'s "Scope is applied as a query filter, not by discarding fetched rows"
requirement — post-filtering "leaks counts and breaks pagination," and over a kNN result set it also
silently shortens the candidate evidence a ranking run sees. `design.md` D3.*

#### Scenario: Scoped retrieval

- **WHEN** a user with limited read scope triggers retrieval
- **THEN** the query carries the evaluator's filter and returns only in-scope records

#### Scenario: Verdict source

- **WHEN** retrieval evaluates authorization
- **THEN** the verdict comes from the central permission evaluator, not from logic local to this
  capability

#### Scenario: Retrieval without a filter

- **WHEN** a retrieval path is exercised without applying the evaluator's scope filter
- **THEN** the test suite fails

#### Scenario: Result counts reflect scope

- **WHEN** a scoped user retrieves evidence
- **THEN** the reported result count reflects only in-scope records, disclosing nothing about
  records outside scope

### Requirement: A permission change requires no re-index

Changing the permission matrix, a role's scope, or a user's overrides SHALL take effect on the next
retrieval without re-embedding or re-indexing any record.

*Source: `exploration-notes.md`
[Interaction A](../../../talentsphere/exploration-notes.md#interaction-a--permission-changes-now-trigger-vector-re-indexing),
whose stated design choice is "evaluate authorization at query time rather than baking scope into
the index"; `access-control-and-admin`'s D4, which named this capability as the consumer and noted
that "getting this wrong there is expensive and getting it wrong here is cheap to avoid";
`AUTHZ`-level uncached evaluation, so revocation bites on the next request.*

#### Scenario: Override added

- **WHEN** a deny override is added for a user
- **THEN** their next retrieval excludes the newly denied records with no re-index

#### Scenario: Grant widened

- **WHEN** a role's read scope is widened
- **THEN** the next retrieval includes the newly permitted records with no re-index

#### Scenario: Re-index triggers enumerated

- **WHEN** the conditions that trigger a re-index are enumerated
- **THEN** authorization change is not among them

### Requirement: Filters are applied on the index, and only the minimum chunks are retrieved

Retrieval SHALL support filtering by candidate identifier, posting identifier, practice, source
type and source version, and SHALL retrieve only the minimum chunks required for the task. A
configured retrieval bound SHALL apply.

*Source: `VEC-004`, `VEC-007`. `D07`'s low volumes — fewer than 5,000 candidates and 30 postings —
mean an unbounded retrieval is a plausible accident rather than a load problem, which is the same
argument the gateway's consumption bounds rest on.*

#### Scenario: Posting-scoped retrieval

- **WHEN** evidence is retrieved for one posting's candidates
- **THEN** the filter restricts results to that posting's candidate set

#### Scenario: Minimum retrieval

- **WHEN** a task needs evidence for one candidate
- **THEN** only that candidate's chunks are retrieved, within the configured bound

#### Scenario: Bound exceeded

- **WHEN** a retrieval request would exceed the configured bound
- **THEN** it is refused or truncated with the truncation reported, rather than silently returning a
  partial set as complete

### Requirement: Retrieval output carries source labels

Every retrieved item SHALL carry a source label drawn from the closed evidence-source vocabulary.

*Source: `VEC-008`, `G-02`; `ai-platform-governance`'s `ai-platform/evidence-labeling` capability,
whose vocabulary is closed and defined once — this capability consumes it and derives no source
value of its own.*

#### Scenario: Labelled result

- **WHEN** retrieval returns an item
- **THEN** it carries a source label from the closed vocabulary

#### Scenario: Label derived locally

- **WHEN** this capability is inspected for source-value derivation
- **THEN** none exists, and labels come from the vocabulary owner

### Requirement: Retrieval unavailability degrades ranking and breaks nothing else

When the vector backend is unavailable, existing rankings, boards, candidate records and every
non-AI action SHALL remain fully usable, and a ranking request SHALL report the capability as
temporarily unavailable rather than producing a ranking from an empty evidence set.

*Source: `G-12`, `NFR-004`, `DEP-008`; `ai-platform-governance`'s graceful-degradation requirement,
whose reasoning applies with one addition specific here — a retrieval failure that returned an empty
set would produce a ranking that looks complete and is evidence-free, which is worse than a reported
failure.*

#### Scenario: Backend unavailable

- **WHEN** the vector backend is unavailable
- **THEN** existing rankings and boards render, non-AI actions work, and the ranking control reports
  temporary unavailability

#### Scenario: Empty evidence set from a failure

- **WHEN** retrieval fails
- **THEN** the ranking run is recorded as failed rather than proceeding on an empty evidence set

### Requirement: Retrieval is permission-gated and audited as a read of candidate evidence

Triggering retrieval SHALL require the caller to hold the applicable action on the surface it is
invoked from, evaluated server-side, and SHALL be denied for an unpermitted caller rather than
returning an empty result.

*Source: `AUTHZ-003`, `AUTHZ-004` — a direct API call fails even when the UI hides the control;
`API-002`. Returning empty rather than denying is the pattern every Phase-2 feature has been
required to avoid, because an empty result and a denial are indistinguishable to the caller and only
one of them is true.*

#### Scenario: Unpermitted caller

- **WHEN** a caller without the required action requests retrieval directly
- **THEN** the request is denied server-side

#### Scenario: Denial is distinguishable from emptiness

- **WHEN** a denied retrieval is compared with a permitted retrieval that matched nothing
- **THEN** the two responses are distinguishable
