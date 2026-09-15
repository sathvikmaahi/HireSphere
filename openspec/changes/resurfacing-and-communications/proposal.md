## Why

Ten features have built the pipeline from posting to closure. None of them keep a candidate who
doesn't fit their submitted posting — a `NotSelected` or bench Application today simply ends, and a
candidate who was good enough to select but never offered gets no advantage the next time a
matching posting opens. `project.md`'s own workflow diagram names the gap directly: *"Candidates who
don't fit their submitted posting are resurfaced against other open postings rather than lost, with
a priority lane for selected-but-not-offered candidates."* `domain-model.md` already reserves the
entity for it — `MatchSuggestion`, *"lighter than `Application`, promoted into a real `Application`
only when a human acts on it"* — and two already-verified sibling features have each published an
event with this feature named as the intended, still-unbuilt consumer. This is also the last of
the twelve features: it closes the seam every prior feature that reached Phase 5 left open on
purpose (`D.10`'s "clean decomposition, no leftover").

## What Changes

Five backlog items, `TS-BL-075` through `TS-BL-079`, scoped by `D04`/`D06`/`C-04`'s already-settled
split — internal notification early, calendar and candidate email deferred — not reopened here.

- **`TS-BL-075` — MatchSuggestion resurfacing engine.** Registers against **two** events, not one:
  `hiring-postings`' `posting.opened` (named alongside `matching-and-ranking`'s `TS-BL-052` as a
  second, independent subscriber — §25 lists Candidate Ranking and Existing Candidate Resurfacing
  as two separate job types both triggered by "posting open") and `decision-and-offers`' `TS-BL-069`
  `posting.closed` ("move bench to the resurfacing pool"). Both events were published with no
  subscriber registered specifically because this feature did not exist yet; this item is that
  subscriber. Produces `MatchSuggestion` records — evaluated against the AI gateway's `resurfacing`
  prompt family (§16.1: *"Existing candidate database, prior outcomes → Existing candidate list and
  priority lane, Practice Manager review required"*) — and writes resurfaced Applications with
  `source_type = existing_database`, `G-10`'s already-decided vocabulary, not a new one.
- **`TS-BL-076` — Priority lane precedence logic.** Ranks selected-but-not-offered candidates
  against each other and against the rest of the resurfacing pool. Reads the `selection_tag =
  selected_not_offered` tag and retained selection reason that `decision-and-offers`' `TS-BL-064`
  already writes (`SEL-004`, `SEL-005`) — this item does not create that tag, only ranks against it,
  matching `D.10`'s dependency direction.
- **`TS-BL-077` — Internal email notification delivery.** `D04`'s early half: registers
  resurfacing- and priority-lane-specific notification templates and triggers against
  `platform-core`'s already-built internal notification engine (`TS-BL-005`) — the sender-side
  internal-recipient guard, the payload contract, the email channel, and the dispatch registration
  all already exist; this item is a consumer of that substrate, not a second implementation of it.
- **`TS-BL-078` — Candidate-facing communication gate.** Holds `TS-BL-005`'s structural
  candidate-facing exception closed. `D04` kept candidate email out of scope entirely and `C-04`
  gates it on Legal (`OD-005`); `platform-core`'s own notification design built the guard
  structurally for exactly this reason (*"'we will not send to candidates' is not a control on its
  own"*). This item is where that exception would eventually open — it does not open it, and no
  candidate-facing email, even a stub, is built here.
- **`TS-BL-079` — Calendar integration, stub scope only.** `D06` deferred the full calendar API;
  `C-04`'s resolution leaves only a placeholder for the later slice, not a working integration.

**One direct dependency added for legibility, not to close a gap.** `resurfacing` is the tenth and
final AI prompt family (§16.1, §16.3), and `TS-BL-075` calls the AI gateway to produce it — the
same pattern every other AI-invoking item in the backlog carries a *direct* `TS-BL-027` edge for
(`interview-pipeline`'s `TS-BL-058`/`060`, `matching-and-ranking`'s `TS-BL-052`). `D.10` omits that
direct edge, but `interview-pipeline` `design.md` D1's own wiring table had already correctly
recorded `resurfacing` as transitively wired — "Yes — via `TS-BL-052`" — since `TS-BL-075` already
depends on `TS-BL-052`, which already depends on `TS-BL-027`. That is a genuinely different finding
from the same table's two real gaps, `interview_questions` and `interview_note_summary`, both
marked "No" and closed by `interview-pipeline` itself. The direct edge is added so the AI-invoking
relationship reads off this item's own `depends_on` line rather than requiring a reader to trace it
through a sibling's dependency — not because the scheduling order it guarantees was previously
unguaranteed. `design.md` D3.

## Capabilities

### New Capabilities

- `resurfacing/resurfacing-engine`: `MatchSuggestion` creation from the `posting.opened` and
  `posting.closed` events, the `resurfacing` AI prompt invocation and its PM-review gate, promotion
  of a suggestion into a real Application on human action, and the `source_type = existing_database`
  write.
- `resurfacing/priority-lane`: precedence ordering for `selected_not_offered` candidates within the
  resurfacing pool, reading (not writing) `decision-and-offers`' selection tag and reason.
- `communications/internal-notifications`: resurfacing- and priority-lane-specific notification
  templates and triggers registered against `platform-core`'s existing internal delivery engine.
- `communications/candidate-communication-gate`: the deliberately-closed exception point for
  candidate-facing communication, blocked on `OD-005`.
- `communications/calendar-integration`: the stub-scope placeholder for the deferred calendar
  slice, per `D06`'s recommendation and `C-04`'s resolution.

### Modified Capabilities

None. This feature reads `hiring-postings`' `posting.opened`, `decision-and-offers`'
`posting.closed` and `selection_tag`, `candidate-intake`'s `source_type` vocabulary, and
`platform-core`'s notification engine at their existing, already-shipped contracts. Nothing this
feature needs requires a sibling's requirements to change.

## Impact

**Endpoints.** §13.2 names none for resurfacing or communications — a gap this feature's own
scope, not a prior feature's oversight, since §13.2 predates the resurfacing engine existing.
`design.md` D5 adds the routes this feature needs, following the same reasoning
`insight-and-reporting` used for its own two unlisted routes.

**Consumed, not rebuilt.** `hiring-postings` `TS-BL-038` (`posting.opened`); `matching-and-ranking`
`TS-BL-052` (independent second subscriber, same event); `decision-and-offers` `TS-BL-069`
(`posting.closed`) and `TS-BL-064` (`selection_tag`, `SEL-004`/`SEL-005`); `candidate-intake`
`TS-BL-046` (`source_type` field and vocabulary, `G-10`); `ai-platform-governance` `TS-BL-027` (AI
gateway, `resurfacing` prompt family); `platform-core` `TS-BL-005` (internal notification engine,
sender-side internal-recipient guard) and `TS-BL-006` (dispatch substrate).

**Roles.** Practice Manager reviews resurfacing output before any promotion to a real Application
(§16.1's human gate). Recruiter and Practice Manager receive internal notifications. No role gains
candidate-facing communication capability in this change.

**No application code changes in this change.** Planning only.

## Reconciliation with `talentsphere-wave-1-foundation`

Checked, as every feature does. **Clean, as expected.** `KNOWN_ISSUES.md`'s nine entries are
environments not provisioned, the mocked Hubble login contract (`OD-001`), non-durable audit, the
unexercised `audit_logs` grant, hand-applied schema ownership, `/build` reporting unknowns, the
blocked npm registry, and Sprint-0 endpoints without declared permissions — none touch resurfacing,
priority lane, or notifications. `sprint-0-outcome.md` could not be re-read directly in this
conversation (a local file-access limitation, not a content concern); `decision-and-offers`'
`design.md` D12 already reconciled it in full for Phase 3 and found it *"platform-layer only... no
hiring feature exist[ing] yet, by design"* with a carry-forward table entirely Wave 1's own —
nothing in it names selection, offers, closure, or resurfacing. This feature invokes the AI gateway
(`TS-BL-027`), so **`OD-003`** (`§35`, the undecided model provider) applies here as it has to every
other AI-invoking feature: not yet recorded in `KNOWN_ISSUES.md`, landing with `ai-platform-
governance`'s still-unbuilt task 1.18, per the cross-feature obligations table.
