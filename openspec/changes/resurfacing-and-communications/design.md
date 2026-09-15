## Context

This is the twelfth and final feature. Every substrate it consumes is already built and verified:
`hiring-postings`' `posting.opened`, `decision-and-offers`' `posting.closed` and `selection_tag`,
`candidate-intake`'s `source_type` vocabulary (`G-10`), `ai-platform-governance`'s AI gateway
(`TS-BL-027`), and `platform-core`'s internal notification engine (`TS-BL-005`). Two of those five
were published as events with **no subscriber**, specifically because this feature did not exist
yet — `hiring-postings` `design.md` D7 and `decision-and-offers` `design.md` D8 both name `TS-BL-075`
as the intended, deliberately-deferred consumer. Nothing here changes a sibling's shipped contract;
see proposal.md's Capabilities section for why no capability is Modified.

`domain-model.md`'s spine names the entity this feature builds: `MatchSuggestion`, *"lighter than
`Application`, promoted into a real `Application` only when a human acts on it."* Its own diagram's
`source` field was corrected during this propose conversation — see that file's dated 2026-08-27
note — to match `G-10`'s actual decided vocabulary rather than a placeholder sketch.

## Goals / Non-Goals

**Goals:**
- Build the `MatchSuggestion` pool from both events that feed it, rank the selected-not-offered
  priority lane within it, and promote entries into real Applications only on explicit human action.
- Deliver internal notifications for resurfacing and priority-lane events through the existing
  notification substrate.
- Hold the candidate-facing communication gate closed, structurally, until `OD-005` resolves.
- Leave a calendar-integration seam that a later slice can implement without a call-site change.

**Non-Goals:**
- No candidate-facing email, SMS, or any candidate-reachable channel — not even a stub. `TS-BL-078`
  is a closed gate, not a partial implementation.
- No live calendar API call, free/busy lookup, or `.ics` generation. `C-04` rejected `.ics`
  outright (*"Not adopted"*) and deferred the API; `TS-BL-079` builds neither.
- No new AI prompt family, ranking algorithm, or vector-retrieval mechanism. The `resurfacing`
  template (§16.3) and Vertex AI Vector Search (`matching-and-ranking`'s `TS-BL-050`) are consumed
  as-is.
- No re-implementation of `decision-and-offers`' `selected_not_offered` tagging or
  `candidate-intake`'s dedup/source-type logic. Both are read, not written, by this feature (except
  the one field this feature does write — see D4).

## Decisions

### D1 — `TS-BL-075` registers against two independent events, and §25's one trigger row covers both

`posting.opened` and `posting.closed` are both named, in already-verified sibling `design.md`s, as
events this feature is the intended consumer of — `hiring-postings` D7 for the first,
`decision-and-offers` D8 for the second. They are not alternate names for the same trigger; they
produce different `MatchSuggestion` populations:

| Event | Population evaluated | Why |
|---|---|---|
| `posting.opened` | The existing candidate database against the newly opened posting | §16.1: *"Candidate Resurfacing \| New posting opened \| Existing candidate database, prior outcomes"* |
| `posting.closed` | That posting's just-terminated bench Applications against other currently-open postings | `decision-and-offers` D8: *"move bench to the resurfacing pool"* — the fourth cascade consequence, published with the terminated Applications' references |

Both registrations belong to one item, `TS-BL-075`, because they produce the same entity through the
same prompt family and the same review gate — two triggers of one capability, not two capabilities.
§25 lists a single row, *"Existing Candidate Resurfacing \| Posting open"*, but a job-type row naming
one trigger does not mean it is exhaustive: the **Notifications** row already covers plural triggers
under one name (*"trigger: workflow events"*), and `platform-core` D9 relies on exactly that reading.
The same reading applies here without needing a second row.

*Alternative considered:* split into two items, one per event. Rejected — `D.10` already gives
`TS-BL-075` a single ID with both `TS-BL-052` (matching-and-ranking, whose own `posting.opened`
registration is independent of this one) and `TS-BL-069` (decision-and-offers) as dependencies, and
splitting would duplicate the prompt invocation, the review gate, and the promotion mechanism across
two items for no behavioral difference.

*The property that must hold:* neither posting-state transition may become dependent on this
feature's delivery — `G-12`'s degradation reasoning, already asserted by both publishers. A
resurfacing-run failure never blocks a posting opening or closing; it is retried as a background job
per §25's retry-safe-failures policy.

### D2 — `MatchSuggestion` carries the AI output, the review gate, and the promotion boundary

A `MatchSuggestion` record is created per (candidate, target posting) pair the `resurfacing` prompt
family surfaces, carrying:

- Candidate and target-posting references.
- The triggering event (`posting_opened` | `posting_closed`) and, for the closed-posting path, the
  originating posting and terminated-Application reference the event carried.
- **Prior outcome** and **mandatory criteria status** — §16.3's exact output contract for this
  template family (*"Candidate list with prior outcome and mandatory criteria status"*), each with
  the evidence-source label `evidence-source labeling` (`glossary.md`) requires.
- The AI run reference (`AIRun`), so the suggestion is reproducible against the
  resume/JD/template/model tuple that produced it (`domain-model.md`'s `RankingScore` provenance
  reasoning applies identically here — the same tuple discipline, not a re-derivation of it).
- A review state: `pending` → `promoted` | `dismissed`, with the acting user and, for `dismissed`,
  a mandatory reason (`UI-003`'s "next action" visibility, and consistent with every other
  disposition in the pipeline requiring a reason on removal).
- A `priority_lane` flag and rank, populated by `TS-BL-076` — see D6.

**Promotion is the only path to a real Application**, and it is never automatic. §16.1's human gate
for this capability is explicit: *"Practice Manager review required."* Promoting a suggestion
creates an Application on the target posting with `source_type = existing_database` (D4) and
`stage` at the same entry point a direct submission starts from — from there it is an ordinary
Application, visible to `matching-and-ranking`'s existing ranking board and pipeline surfaces with
no separate code path. `RANK-007`'s requirement — *"Existing candidate matches shall appear before
or alongside new submissions when a posting opens"* — is satisfied by D1's `posting.opened`
registration running as its own independent subscriber, not by merging the two lists into one
screen; §14.2 keeps *Existing Candidate Resurfacing* and *AI Ranking Board* as distinct screens.

**Staleness governs the pool, not a manual cleanup.** `D22`, as amended by `C-11`, seeds two audited
runtime-configuration values this feature reads rather than hard-codes: a 12-month resurfacing
recency window (a resume older than that drops out of the pool unless a new version resets the
clock) and a 90-day priority-lane TTL (past which a selected-not-offered candidate still resurfaces,
but as a standard match — D6). Neither window may exceed the retention period `OD-004` sets; a
shorter retention period narrows the seeded value, never the reverse.

*Alternative considered:* skip the review gate for high-confidence matches. Rejected — §16.1 makes
review a Human Gate on the capability itself, not a confidence-graded control, and `AI-010` forbids
any code path where AI output alone advances a candidate's state.

### D3 — `TS-BL-075` gains a direct dependency on `ai-platform-governance`'s `TS-BL-027`, as explicit hygiene, not a gap fix

`D.10`'s table gives `TS-BL-075` no *direct* edge to `TS-BL-027`, the AI gateway. Checked against
`interview-pipeline` `design.md` D1's own wiring table before treating this as a gap — that table
already lists `resurfacing | resurfacing-and-communications TS-BL-075 | Yes — via TS-BL-052`,
correctly identifying it as transitively wired: `TS-BL-075` already depends on
`matching-and-ranking`'s `TS-BL-052`, and `TS-BL-052` already depends on `TS-BL-027`
(`[TS-BL-037, TS-BL-051, TS-BL-027]`). That is a different finding from the same table's two real
gaps, `interview_questions` and `interview_note_summary`, both correctly marked "No" and closed by
`interview-pipeline` itself. `resurfacing`'s scheduling-order guarantee — that `TS-BL-075` cannot
be scheduled before the gateway exists — already held before this design touched anything.

The edge is added anyway, for legibility rather than correctness: `resurfacing` is the tenth and
final entry in §16.3's prompt template registry, and every other prompt-authoring item in the
backlog carries a *direct* edge to the gateway it calls (`matching-and-ranking`'s `TS-BL-052`,
`interview-pipeline`'s `TS-BL-058`/`TS-BL-060`) rather than one a reader has to trace through a
sibling's own dependency. Added here as a within-feature refinement under `D.11`'s permission —
`TS-BL-075`'s `depends_on` gains `TS-BL-027` alongside its existing `TS-BL-052`, `TS-BL-069`.

*What this is not:* not a correctness fix. Nothing about `TS-BL-075`'s buildable scheduling order
changes — the transitive path already guaranteed the gateway exists first. The direct edge makes
that guarantee readable from this item's own `depends_on` line; it does not create it.

### D4 — Resurfaced Applications write `G-10`'s existing `source_type` field, not a new one

`candidate_posting_applications.source_type` is `candidate-intake`'s `TS-BL-046` (task 6.12), already
built and already the substrate `insight-and-reporting`'s resurfacing-yield metric reads (its
`design.md` D6: *"the metric therefore needs no dependency edge to `TS-BL-075`... until `TS-BL-075`
populates the value, the surface renders a distinct 'no resurfaced applications yet' state"*).
Promotion (D2) writes `existing_database` into that column. No parallel `source` enum, and no new
migration to the `candidate_posting_applications` table for this purpose — `domain-model.md`'s
spine diagram was corrected during this conversation to stop showing the placeholder three-value
sketch that predated this decision (see that file's 2026-08-27 note).

*Alternative considered:* a `resurfaced: boolean` flag alongside the existing enum. Rejected —
it would let a row disagree with itself (`source_type = recruiter` and `resurfaced = true`
simultaneously), and `G-10`'s enum already has the value this needs.

### D5 — Endpoints not named in §13.2, added with the same reasoning `insight-and-reporting` used

§13.2 predates this feature's existence and names no resurfacing or communications routes. Three
are added:

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/postings/{postingId}/resurfacing` | List `MatchSuggestion`s for a posting, each carrying prior outcome, mandatory criteria status, priority-lane flag and rank |
| POST | `/api/match-suggestions/{suggestionId}/promote` | Practice Manager promotes a suggestion to a real Application (`source_type = existing_database`) |
| POST | `/api/match-suggestions/{suggestionId}/dismiss` | Dismiss with mandatory reason |

No manual resurfacing-trigger route is added — unlike `POST /api/postings/{postingId}/ai/rank`,
§25's row for this job type names only event triggers (`posting.opened`, and per D1,
`posting.closed`), not a manual one, and neither sibling `design.md` that names this feature as a
consumer describes a manual trigger either. No new endpoint is added for `TS-BL-077`'s
notifications: it registers templates and triggers against `platform-core`'s existing notification
surface, which already owns its own endpoints.

### D6 — `TS-BL-076` ranks within the pool D1/D2 already build; it does not create the tag it reads

The priority lane is an ordering, not a second pool. It reads `decision-and-offers`' `TS-BL-064`
`selection_tag = selected_not_offered` and retained reason (`SEL-004`, `SEL-005`) on a candidate's
prior Application, and — where that candidate also appears among `TS-BL-075`'s `MatchSuggestion`s
for a different, open posting — sets `priority_lane = true` and a rank ahead of standard matches,
**subject to the new posting's mandatory criteria**: `RANK-011`'s permitted signal is *"Prior
selected-not-offered status, subject to mandatory criteria checks,"* and `BR-015` states the same
qualifier. A candidate failing the new posting's mandatory criteria is not lane-prioritized over one
who passes them, regardless of prior selection history.

**This item does not invoke carry-forward.** `decision-and-offers` D8 is explicit about the
boundary: *"`TS-BL-066` builds carry-forward as a mechanism that operates on a selected-not-offered
Application however it was surfaced... `TS-BL-076` builds the lane that surfaces candidates into
it. Neither depends on the other's internals."* When a lane candidate is promoted (D2) and later
shortlisted, `TS-BL-066`'s existing carry-forward eligibility check (`D17`/`D21`) runs exactly as it
would for any other route into a selected-not-offered candidate's re-consideration — this item's
job ends at surfacing and ranking.

**The 90-day TTL demotes, it does not remove** (D2, `D22`/`C-11`). Past the TTL, the candidate still
resurfaces through D1's ordinary path; only the `priority_lane` flag and rank stop applying.

### D7 — `TS-BL-077` registers templates and triggers; it builds no delivery mechanism

`platform-core`'s `TS-BL-005` already ships the sender-side internal-recipient guard, the delivery
payload contract, the in-app and internal-email channels, and the dispatch registration against
`TS-BL-006`. This item's scope is exactly two triggers and their templates, registered against that
existing surface:

- A `MatchSuggestion` pool changes for a posting a Practice Manager owns (new suggestions from
  either D1 trigger) → in-app + internal email, addressed to the owning Practice Manager.
- A candidate enters the priority lane (`priority_lane` flips to `true`) → in-app + internal email,
  same recipient rule.

Both are internal-recipient only, enforced by `TS-BL-005`'s existing guard — this item adds no
recipient-resolution logic of its own. `D04`'s early half (*"internal email, no candidate email"*)
is satisfied by consuming that guard, not by re-asserting it.

### D8 — `TS-BL-078` holds the candidate-facing exception closed; nothing here opens it

`platform-core` `design.md` D9 built the guard structurally *because* a future feature would need to
open a candidate-facing exception, and named this item as where: *"`resurfacing-and-communications`'
`TS-BL-078` is where the gate is deliberately opened, and it remains blocked on `OD-005`."* This
design does not implement that opening. `TS-BL-078`'s entire scope in this change is:

1. Recording the gate as a named, tracked exception point (a runtime-configuration flag, seeded
   `disabled`, requiring the same audited-change discipline every other governed flag in this
   project uses) rather than leaving `OD-005`'s resolution with no landing spot.
2. Asserting, as a testable property, that no candidate-reachable channel exists anywhere in this
   feature's surface while the flag is `disabled` — which is every environment until `OD-005`
   resolves.

Building the candidate template, the disclosure text, or the send path itself is out of scope until
`OD-005` — Legal's jurisdictional AI-disclosure sign-off — resolves, per `C-04`'s resolution and
`D04`'s recorded consequence that the candidate email template is *"a governance artifact, not just
copy."*

### D9 — `TS-BL-079`'s "stub" is a disabled seam, not a partial calendar client

`D06`'s decision (full calendar API) was superseded by `C-04`'s resolution, which is precise about
what survives: the Google Calendar platform choice (`D10`) is kept **for when the calendar slice
arrives**; `.ics` attachments are **not adopted** at all; and the accepted interim gap is explicit —
*"panelists receive an assignment email but no calendar entry; they add the meeting themselves."*
`TS-BL-077` already delivers that assignment email (D7). So there is nothing left for a "stub" to
partially build toward except the seam a later slice will fill:

- A `calendar_integration` feature flag, declared and seeded `disabled` — per this project's
  standing rule, an unknown flag raises rather than resolving false, and a disabled capability
  answers 404, so `TS-BL-079` adds the declaration and no reachable route behind it.
- An adapter interface shaped for D06's recorded recommendation — TalentSphere as source of truth,
  the calendar a one-way projection — so the later slice implements against an existing seam
  instead of one designed under deadline pressure. The interface is never called while the flag is
  disabled.

No Calendar API call, no free/busy lookup, no `.ics` generation. `TS-BL-079`'s dependency on
`interview-pipeline`'s `TS-BL-057` (round scheduling) is the seam's input shape — the scheduled
round's date, time and panelist set are what a future projection would carry — not a functional
call from one item into the other.

### D10 — Reconciliation: what was checked, and one thing recorded rather than assumed

`KNOWN_ISSUES.md`'s nine entries (read directly) are environments not provisioned, `OD-001`, non-
durable audit, the unexercised `audit_logs` grant, hand-applied schema ownership, `/build`
reporting unknowns, the blocked npm registry, and undeclared Sprint-0 endpoint permissions — none
touch resurfacing, notifications, or communications. `sprint-0-outcome.md` could not be re-read
directly in this conversation — a local dataless-file access limitation, not a content gap — so this
relies on `decision-and-offers` `design.md` D12's own direct reconciliation of the same file, which
found it *"platform-layer only... no hiring feature exist[ing] yet, by design"* with a
carry-forward table entirely Wave 1's own. **Expected clean, and consistent with what a sibling
feature already verified directly.**

This feature calls the AI gateway (D3), so **`OD-003`** — `reference/spec.md` §35's undecided AI
model provider — applies to it exactly as it applies to every other AI-invoking feature. It is not
yet recorded in `KNOWN_ISSUES.md`; `ai-platform-governance`'s still-unbuilt task 1.18 is what will
record it, per the cross-feature obligations table this conversation checked and found no entry
naming this feature.

## Risks / Trade-offs

- **Two independent triggers on one item (D1)** risks a suggestion being evaluated twice if a
  posting closes and reopens in a way that re-fires both paths for the same candidate/posting pair.
  → Promotion and suggestion creation are both idempotent per (candidate, posting, triggering
  event) — a repeat evaluation updates the existing `pending` suggestion's AI-run reference rather
  than creating a duplicate, the same at-least-once-with-idempotent-consumer contract both
  publishing sides already require.
- **The priority lane (D6) reads a tag it does not own.** If `decision-and-offers` ever changes
  `SEL-004`'s tag values, this feature's ranking silently stops finding lane candidates rather than
  failing loudly. → A scenario asserting the tag value it reads exists in `decision-and-offers`'
  own spec (already true today) is included in this feature's spec as a documented cross-feature
  contract, not a runtime check this feature can itself enforce.
- **A disabled calendar seam (D9) can bit-rot** if the later slice's actual requirements diverge
  from D06's recorded recommendation before it is ever implemented. → Accepted; the alternative —
  building speculative depth now — is exactly what `C-04` deferred and costs real effort against an
  interface nobody has used yet.

## Open Questions

None. Every deferral in this design (candidate communication, calendar depth) is a scoped, cited
gate rather than an unresolved technical unknown — `TS-BL-078` and `TS-BL-079`'s Non-Goals are the
record of what is deliberately not decided here, not a question left open.
