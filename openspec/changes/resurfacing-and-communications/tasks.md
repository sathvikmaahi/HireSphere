## 1. TS-BL-075 — MatchSuggestion resurfacing engine

**Goal:** a candidate who doesn't fit their submitted posting is never simply lost — evaluated
against the whole candidate database when a new posting opens, and against every other open
posting when their own posting closes — surfaced as a reviewable suggestion, never as an automatic
Application. Covers `resurfacing/resurfacing-engine`.

```yaml
backlog_items:
  - id: TS-BL-075
    feature: resurfacing-and-communications
    depends_on: [TS-BL-052, TS-BL-069, TS-BL-027]
    status: not-started
```

**The `TS-BL-027` edge is added here as explicit hygiene, not a gap fix — `D.10` omits the direct
edge, but the transitive path through `TS-BL-052` already guaranteed the gateway exists first.**
`interview-pipeline` `design.md` D1's own wiring table had already correctly recorded `resurfacing`
as transitively wired ("Yes — via `TS-BL-052`"), unlike its two genuinely unwired families
(`interview_questions`, `interview_note_summary`) that it closed itself. Justified on `design.md`
D3, a within-feature refinement under `D.11`'s permission, not a standing grant.

**Read `design.md` D1–D5 before starting.** D1 explains why this item registers against two
independent events rather than splitting into two items. D2 defines the `MatchSuggestion` record,
its review gate, and the staleness windows. D3 is the added gateway dependency. D4 is the
`source_type` write. D5 lists the three endpoints this item adds, none of which exist in §13.2.

- [ ] 1.1 Migrate the `match_suggestions` table: candidate reference, target-posting reference,
      triggering event (`posting_opened` | `posting_closed`), originating-posting and
      terminated-Application references for the closed-posting path, prior-outcome and
      mandatory-criteria-status fields with evidence-source labels, `AIRun` reference, review state
      (`pending` | `promoted` | `dismissed`), acting-user and reason-on-dismissal fields, and the
      `priority_lane`/rank columns `TS-BL-076` will populate (`design.md` D2)
- [ ] 1.2 Register a subscriber on `hiring-postings`' `posting.opened` event, independent of
      `matching-and-ranking`'s own `TS-BL-052` registration on the same event, and evaluate the
      existing candidate database against the newly opened posting through the `resurfacing` AI
      prompt family (§16.1, §16.3, `design.md` D1)
- [ ] 1.3 Register a subscriber on `decision-and-offers`' `posting.closed` event and evaluate that
      posting's terminated bench Applications (carried on the event payload) against other
      currently open postings through the same prompt family (`design.md` D1, `decision-and-offers`
      `design.md` D8)
- [ ] 1.4 Assert neither registration blocks its publisher: a resurfacing-evaluation failure leaves
      the posting open or closed as it already was, and is retried as a background job per §25's
      retry-safe-failures policy (`G-12`, `design.md` D1's Risks)
- [ ] 1.5 Write the run through the AI gateway with the run log written before invocation, carrying
      the resume/posting/template/model tuple, and populate each `MatchSuggestion`'s prior-outcome
      and mandatory-criteria-status fields with evidence-source labels (§16.3, `glossary.md`'s
      evidence-source labeling, `design.md` D2)
- [ ] 1.6 Expose `GET /api/postings/{postingId}/resurfacing`, permission-gated, returning each
      suggestion's prior outcome, mandatory criteria status, and (once `TS-BL-076` lands)
      priority-lane flag and rank (`design.md` D5)
- [ ] 1.7 Implement `POST /api/match-suggestions/{suggestionId}/promote`: Practice-Manager-only,
      creates a real Application on the target posting, and assert **no automated path** creates an
      Application from a suggestion (§16.1's Human Gate, `AI-010`, `design.md` D2)
- [ ] 1.8 Implement `POST /api/match-suggestions/{suggestionId}/dismiss` with a mandatory reason
      (`design.md` D2, D5)
- [ ] 1.9 On promotion, write `source_type = existing_database` using `candidate-intake`'s existing
      `G-10` field and vocabulary — assert no parallel classification field is added
      (`design.md` D4, `candidate-intake` `TS-BL-046` task 6.12)
- [ ] 1.10 Assert a promoted Application requires no resurfacing-specific downstream branch: it is
      visible to the existing ranking board, shortlisting, interview and scorecard surfaces through
      the same code path a direct submission uses (`RANK-007`, `design.md` D2)
- [ ] 1.11 Read the audited, configured 12-month resurfacing recency window (seeded per `D22`/`C-11`)
      and exclude candidates whose most recent resume exceeds it from new suggestion creation,
      resetting on a new resume version; assert the window can never exceed `OD-004`'s retention
      period (`design.md` D2)
- [ ] 1.12 Assert idempotency per (candidate, posting, triggering event): a repeat evaluation updates
      an existing `pending` suggestion's `AIRun` reference rather than duplicating the row
      (`design.md` Risks)

## 2. TS-BL-076 — Priority lane precedence logic

**Goal:** a candidate who was selected but never offered gets precedence the next time they
resurface — read from the tag `decision-and-offers` already writes, never a second tagging
mechanism, and never overriding the target posting's mandatory criteria. Covers
`resurfacing/priority-lane`.

```yaml
backlog_items:
  - id: TS-BL-076
    feature: resurfacing-and-communications
    depends_on: [TS-BL-075, TS-BL-064]
    status: not-started
```

**Read `design.md` D6 before starting.** It fixes the read-only relationship to
`decision-and-offers`' `selection_tag`, the mandatory-criteria qualifier, the TTL's demote-not-
remove semantics, and the boundary with `TS-BL-066`'s carry-forward.

- [ ] 2.1 For each `MatchSuggestion` whose candidate carries `selection_tag = selected_not_offered`
      on a prior Application, set `priority_lane = true` and rank it ahead of standard matches for
      the same posting (`glossary.md`, `BR-015`, `SEL-005`, `design.md` D6)
- [ ] 2.2 Assert this item writes no `selection_tag` and no `selected_not_offered` state anywhere —
      both remain `decision-and-offers`' `TS-BL-064`'s alone (`design.md` D6)
- [ ] 2.3 Gate priority-lane precedence on the target posting's mandatory criteria: a
      `selected_not_offered` candidate who fails them is not ranked ahead of a standard match who
      passes them (`RANK-011`, `BR-015`, `design.md` D6)
- [ ] 2.4 Read the audited, configured 90-day priority-lane TTL (seeded per `D22`/`C-11`) from the
      candidate's `selected_not_offered` tagging date; past it, clear `priority_lane` and leave the
      candidate as a standard match rather than removing them from the pool (`design.md` D6)
- [ ] 2.5 Assert this item invokes no carry-forward logic and mandates no confirmatory round —
      `TS-BL-066` evaluates carry-forward eligibility independently when a promoted lane candidate
      is later shortlisted (`decision-and-offers` `design.md` D8, `design.md` D6)
- [ ] 2.6 Add the cross-feature contract scenario asserting `decision-and-offers`' spec still defines
      `selection_tag = selected_not_offered` as this item's read dependency, so a future change to
      that tag's values is visible as a spec conflict rather than a silent ranking failure
      (`design.md` Risks)

## 3. TS-BL-077 — Internal email notification delivery

**Goal:** a Practice Manager finds out about new resurfacing matches and priority-lane entries
without checking a screen manually — delivered through the internal notification engine that
already exists, with no new transport, retry loop, or recipient check built here. Covers
`communications/internal-notifications`.

```yaml
backlog_items:
  - id: TS-BL-077
    feature: resurfacing-and-communications
    depends_on: [TS-BL-005]
    status: not-started
```

**Read `design.md` D7 before starting.** It fixes the scope as two triggers and their templates
registered against `platform-core`'s existing engine — no channel, payload shape, retry policy, or
recipient guard is redefined here.

- [ ] 3.1 Register a notification trigger for `resurfacing/resurfacing-engine` creating one or more
      `MatchSuggestion`s on a posting, addressed to that posting's owning Practice Manager, through
      `platform-core`'s existing in-app and internal-email channels (`D04`, `design.md` D7)
- [ ] 3.2 Register a notification trigger for `resurfacing/priority-lane` setting
      `priority_lane = true` on a suggestion, addressed to the target posting's owning Practice
      Manager, naming the candidate and the prior selection (`design.md` D7)
- [ ] 3.3 Define both triggers' templates using `TS-BL-005`'s existing payload contract (severity,
      title, body, originating event reference, action reference) — add no new payload shape
      (`platform-core` `TS-BL-005` task 5.4, `design.md` D7)
- [ ] 3.4 Assert both triggers dispatch through `platform-core`'s existing engine and rely entirely
      on its sender-side internal-recipient guard: no recipient-resolution logic is added here, and
      no notification this item triggers is addressed to a candidate record (`platform-core`
      `design.md` D9, `design.md` D7)

## 4. TS-BL-078 — Candidate-facing communication gate

**Goal:** the exception `platform-core`'s notification guard was built to eventually admit has a
named, tracked, audited landing point — closed in every environment until Legal's disclosure
sign-off (`OD-005`) resolves, with no candidate-reachable channel built as a stub in the meantime.
Covers `communications/candidate-communication-gate`.

```yaml
backlog_items:
  - id: TS-BL-078
    feature: resurfacing-and-communications
    depends_on: [TS-BL-077]
    status: not-started
```

**Read `design.md` D8 before starting.** It fixes this item's entire scope: declaring the flag and
asserting the gate holds, not building candidate email or a template.

- [ ] 4.1 Declare `candidate_communication` in the audited runtime-configuration registry, seeded
      `disabled`, requiring a mandatory reason and an audit record on any change
      (`platform-core` `design.md` D9, `design.md` D8)
- [ ] 4.2 Assert no code path in this feature delivers a notification, email, or message to a
      candidate while the flag is `disabled` — every environment, until `OD-005` resolves
      (`D04`, `design.md` D8)
- [ ] 4.3 Assert `communications/internal-notifications`' triggers are unaffected by this flag —
      they deliver to internal recipients regardless of its state (`design.md` D8)
- [ ] 4.4 Assert this feature defines no candidate-facing message template or disclosure text —
      that is Legal's governance artifact to define after `OD-005`, in a future change
      (`D04`'s recorded consequence, `design.md` D8)

## 5. TS-BL-079 — Calendar integration, stub scope only

**Goal:** the calendar slice `D06` deferred has a seam waiting for it — a disabled flag and a
one-way-projection adapter interface shaped from the recorded recommendation — with no live
Calendar API call, free/busy lookup, or `.ics` generation built now. Covers
`communications/calendar-integration`.

```yaml
backlog_items:
  - id: TS-BL-079
    feature: resurfacing-and-communications
    depends_on: [TS-BL-057]
    status: not-started
```

**Read `design.md` D9 before starting — "stub" is a disabled seam, not a partial calendar client.**
`.ics` attachments are not adopted at all (`C-04`); the full API is deferred, not partially built.

- [ ] 5.1 Declare `calendar_integration` in the audited runtime-configuration registry, seeded
      `disabled`; assert any route behind it answers 404 while disabled (`design.md` D9)
- [ ] 5.2 Define an adapter interface accepting a scheduled interview round's date, time, and
      panelist set (from `interview-pipeline`'s `TS-BL-057`) as input, with no inbound path for a
      calendar-side change to reach TalentSphere — the one-way-projection shape `D06` recorded
      (`design.md` D9)
- [ ] 5.3 Assert the adapter interface is never invoked while `calendar_integration` is `disabled`
      (`design.md` D9)
- [ ] 5.4 Assert this item performs no Calendar API call, no free/busy lookup, and generates no
      `.ics` attachment (`C-04`'s resolution table, `design.md` D9)
- [ ] 5.5 Assert the accepted interim gap holds: a panelist assigned to a round still receives
      `communications/internal-notifications`' assignment notification with no calendar entry
      created (`C-04`, `design.md` D9)
