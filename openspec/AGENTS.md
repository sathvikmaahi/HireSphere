# Agent Conventions

For any agent (or person) proposing, designing, or building a TalentSphere feature or backlog
item. This file is the consistency mechanism across ~12-15 independent `/opsx:propose` runs that
share no conversation memory with each other — read it every time, not just once.

## Read before doing anything

In this order:

1. `project.md` — what TalentSphere is, one page
2. `glossary.md` — precise terms, especially "Phase" vs. "Wave" (they mean different things —
   get this backwards and every sprint/wave reference you write will be wrong)
3. `domain-model.md` — entities and how they relate
4. `changes/talentsphere/exploration-notes.md` — the full decision record. Read the relevant
   sections for whatever backlog item you're working on, not necessarily the whole 2000+ lines
   cold, but never skip it in favor of guessing
5. `../reference/spec.md`, `../reference/design-spec.md`, `../reference/layout-spec.md` — precise
   detail (schemas, requirement wording, design tokens) the exploration notes reference but don't
   reproduce in full

**Wherever `exploration-notes.md` and a reference spec disagree, the exploration notes win.**
Several reference-spec recommendations were deliberately amended or rejected — nine roles not
four, consolidated scorecards not per-round, Vertex AI Vector Search not pgvector. Don't silently
default to the reference spec's version.

## Propose at the FEATURE level — this superseded an earlier, wrong decision

**One `/opsx:propose` run = one feature = one OpenSpec change.** Not one per backlog item. An
earlier version of this file said the opposite; that was corrected after realizing a feature
(e.g. "Identity & Access," "Organizations & Teams" in the reference example this project's
delivery model was checked against) is the natural design-cohesion unit — matching what an
OpenSpec change actually is — while a backlog item is a much smaller, independently-deployable
*task* that lives inside a feature's `tasks.md`, not a change of its own.

A feature's change (`proposal.md`, `design.md`, `specs/`, `tasks.md`) covers the shared reasoning
for that whole domain. Its `tasks.md` is that feature's **complete backlog** — every one of its
backlog items, each uniquely numbered (`TS-BL-XXXX`), each described precisely enough to be built
and deployed independently of its siblings, even though they share one proposal and one design.

Run `/opsx:propose` normally for each feature; do not attempt to generate only `proposal.md` and
stop — the manager's instruction is that every feature gets its full artifact set.

**Why this matters for "independently deployable":** that property is a **code and CI/CD**
property, not an OpenSpec-planning property. A backlog item can share its specification with
siblings inside one feature's `design.md` while still being its own separable, independently
mergeable and deployable slice of code. Don't conflate "shares a design document" with "isn't
independently shippable" — they're different layers.

## Naming

- Feature/change directory: `<feature-slug>/` (e.g. `identity-and-access`,
  `access-control-and-admin`, `ai-platform-governance`) — **not** `ts-bl-XXXX-...`; that naming
  belonged to the earlier, superseded per-backlog-item model.
- Backlog item ID (used inside a feature's `tasks.md` and in `delivery/`): `TS-BL-XXX` (three
  digits, globally unique across the whole product, never reused even if a feature is later
  split or renamed).
- Sprint: `TS-SPR-XXX` (three digits, e.g. `TS-SPR-001`).
- Wave: numbered **globally and sequentially across the whole product** (`Wave 1`, `Wave 2`, ...
  `Wave 11`...), never restarted per sprint. A wave's file path still nests under its sprint
  (`delivery/sprints/TS-SPR-XXX/waves/TS-SPR-XXX-WV-XXX.md`) but the wave *number* itself doesn't
  reset there — check the highest wave number already used across all of `delivery/sprints/`
  before assigning a new one.

## Dependency notation, not named assignment — and waves cut across features freely

A backlog item's entry in its feature's `tasks.md`, and its entry in the relevant
`delivery/sprints/.../waves/*.md` file, records what it's blocked on by ID, never by a person's
name, and records **which feature it belongs to** (since it no longer has its own dedicated
change directory to point to):

```yaml
backlog_items:
  - id: TS-BL-004
    feature: identity-and-access
    depends_on: []
    status: not-started | in-progress | done
```

Who actually picks up an item in a given week is a staffing decision made in `sprint.md`, not a
property of the backlog item itself.

**A wave groups by independence, not by feature.** Two items from completely different features
(e.g. `identity-and-access` and `design-system`) belong in the same wave whenever they're
mutually independent — waves were never a feature concept, they're pure scheduling. A wave
**defaults to one per sprint** whenever everything concurrent that sprint is mutually
independent; it only splits into a second wave when there's a genuine reason (most often: an
item depends on something else scheduled the same sprint via deliberate stub-and-integrate
overlap, so it cannot share a wave with the thing it depends on even though they share a week).
Never put dependent items in the same wave "because they're in the same sprint anyway" — that
was a real mistake made once already (see `exploration-notes.md` Part D's schedule-correction
history) and produced a schedule that silently violated its own stated rule.

**Delivery happens per item, not per wave.** A wave is a tracking lens, not a bundled release —
if one item in a wave spills into the next sprint, its wave-mates that finished on time still
move toward UAT without waiting for it.

## Traceability is not optional

Every non-obvious decision in a `design.md`, and every requirement in a `specs/*/spec.md`, should
be one step from *why* — a `D`/`C`/`G`/`S` ID from `exploration-notes.md`, or a requirement ID
from `reference/spec.md` (`AUTH-001`, `AUTHZ-006`, etc.). If you can't cite one, that's a signal
the decision hasn't actually been made yet — go check the exploration notes rather than
inventing a justification.

## Correcting another feature's artifacts — from a build, or from your own propose run

Full `proposal.md`/`design.md`/`specs/`/`tasks.md` are being generated for every feature across
all five Phases before most of them are built — an explicit, accepted risk (see
`exploration-notes.md` D.4). Two different situations lead to needing a fix in a feature that
isn't the one you're working on. **Both take the same route:** `/opsx:update` against that
specific **feature's** change, in its own conversation — never by hand-editing another change's
files, and never by creating a duplicate "v2" change.

- **Apply-stage discovery.** An earlier feature's *actual build* reveals a real technical detail
  that a not-yet-built later feature's `design.md` guessed differently.
- **Propose-stage discovery.** Your own propose run finds that an *already-verified sibling
  feature* is wrong or incomplete — a miscitation, or work its `tasks.md` declares but never
  invokes. This has happened repeatedly and is expected, not exceptional: `decision-and-offers`
  found that `interview-pipeline` declares no caller for the posting-state advances it depends on;
  and `interview-pipeline`, having found a miscitation of `OD-003` in its own `D13`, flagged the
  identical one in `matching-and-ranking` rather than reaching over and editing it.

If the correction only affects one backlog item's entry inside a feature's `tasks.md` rather than
the feature's shared design, it's still the same feature-level change that gets updated — backlog
items don't have their own change to target individually.

**Whichever situation you're in, do not fix it inline and do not let it live only in your own
feature's Open Questions.** The change that needs fixing does not know it needs fixing — an apply
conversation reading only *that* feature's artifacts would never see the gap. Record it in the
**Pending cross-feature obligations** table near the top of `exploration-notes.md` (beside the
Propose progress tracker), naming what's owed, by which change, and which change found it. That
table is the one place every stage reads, and it is what keeps a flagged obligation from being
silently dropped between propose and apply.

What `/opsx:update` actually does, precisely (verified against `.claude/commands/opsx/update.md`,
not assumed from its one-line description): it operates on **one change at a time**, reads that
change's existing artifacts, applies the correction, then checks every other artifact in that
*same* change for coherence in either direction — and confirms each proposed revision with the
user before writing anything. It never touches application code.

Three things worth knowing before invoking it:

- **It's per-change, not bulk.** A correction that ripples across several future backlog items
  needs one `/opsx:update` run per affected item, not a single sweep.
- **If the target item was already built** (`/opsx:apply` already ran for it), fixing its plan
  with `/opsx:update` does not update its code — a follow-up `/opsx:apply` is still needed to
  bring the implementation in line with the revised plan.
- **It's tagged Experimental** in this repo's command list. Still the right tool for this job —
  just review its proposed diffs a bit more carefully than a fully mature command.

## Finding a genuine gap in a shared foundational document, not just in a feature

`/opsx:update` is for correcting a **feature's** artifacts. It doesn't apply to the shared
documents every feature reads — `exploration-notes.md`, `domain-model.md`, `glossary.md`,
`project.md` — none of which are part of any change. If a propose conversation discovers a real,
previously-unreconciled error *inside one of these* — not a design choice you'd prefer
differently, an actual gap or a wrong citation — fix it **in the document where the error actually
lives**, not by working around it in your own feature's artifacts. Two rules, no exceptions:

- **Add a dated note explaining what was wrong, why it was missed, and what's true now.** Don't
  just assert a correction happened — say so at the point of the fix.
- **Quote the prior wording before replacing it.** A note that says "this used to say something
  else" without saying what is not meaningfully different from silently editing it away — the
  point is that a future reader can see what changed and judge the fix themselves, not just trust
  that one happened.

Then continue your propose work; don't block on a separate conversation for this.

This isn't hypothetical, and it isn't confined to `exploration-notes.md`. The `platform-core`
propose conversation found `exploration-notes.md` still listing Cloud Tasks as an open
background-jobs option (2026-08-25) while D.9 had already committed elsewhere to the landing
zone's Pub/Sub chain — fixed inline with a dated `SUPERSEDED` block (see D09 in
`exploration-notes.md`). The `ai-platform-governance` propose conversation later found the error
sitting in `domain-model.md` itself — its "AI platform" section cited `exploration-notes.md`
D9/D10 for the AI-governance reasoning, when those decisions actually live in
`talentsphere-wave-1-foundation/design.md` D9/D10 (`exploration-notes.md`'s own D09/D10 are
unrelated — tech stack and calendar platform). Fixed in `domain-model.md` directly, since that's
where the wrong citation actually was. If a later feature's propose conversation hits a similar
error in any of the four shared documents, do the same, in whichever document the error lives.

## Marking your own feature done in the Propose progress table

`exploration-notes.md`'s "Propose progress" table (near the top of the document) tracks all 12
features across two separate orderings — `Plan #` (the fixed D.9/D.10 dependency order, never
renumbered) and `Seq` (the actual order features get proposed in, which can diverge from `Plan #`
if features aren't done strictly in dependency order). When you finish:

- Update your own row's status to `🟡 Proposed, pending verification` — you may not mark it ✅.
  Verification is an independent audit run separately, after the fact, from the explore
  conversation; a change certifying its own correctness isn't verification, and D.11's
  "finalization item" the whole propose sequence is working toward depends on that distinction
  holding. Leave the ✅ for that later step.
- Fill in your own `Seq` number: check the highest `Seq` already assigned in the table and take
  the next integer (same convention as wave numbering — check the max already used, don't guess
  or reuse your `Plan #`, since the two numberings are expected to drift apart over time).

## Standing product-quality bar (applies to every backlog item that touches AI output or UI)

- **AI output is concise, precise, and never long-form.** Every prompt family's output contract
  needs an explicit, machine-checkable length bound (sentence count, word cap), not just a JSON
  shape. See `exploration-notes.md` S.5 and D11.
- **Visual work follows the token system in `reference/design-spec.md` and the layout system in
  `reference/layout-spec.md`.** No raw hex values, no one-off spacing values, no component that
  bypasses the design tokens.
- **Reuse existing shared patterns before inventing new ones.** The dense-data-table pattern and
  the Authenticated Shell live in the `design-system` feature specifically so later features
  don't reinvent them — check `domain-model.md` and the `design-system` feature's own `design.md`
  before building a new table or shell pattern anywhere else.

## When writing several features' worth of backlog items in one conversation

Work in dependency order at the **feature** level first, then within each feature respect its own
backlog items' internal dependency order — a later feature's `design.md` should be able to
reference an earlier feature's *actual* written design, not guess at it, if that earlier feature
was proposed earlier in the same conversation. (`exploration-notes.md` Part D's backlog table was
written at the earlier, superseded per-backlog-item granularity — treat it as a source of
dependency *relationships* to preserve, not as the literal change list to create.)

## What you are not doing here

Writing `proposal.md`/`design.md`/`specs/`/`tasks.md` is planning, not implementation. Do not
write application code as part of a propose run. That's a separate stage (`/opsx:apply`), reading
the backlog item's own finished artifacts, not `exploration-notes.md` directly — see
`exploration-notes.md`'s note on one-conversation-per-stage discipline.
