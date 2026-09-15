# Glossary

Terms with a specific meaning in this project — especially ones that could otherwise be
misread, or that changed meaning partway through planning. When this glossary and general
usage disagree, this glossary wins for TalentSphere work.

## Delivery model terms

**Phase** — formerly called "Wave." One of five macro-groupings of the product (Foundation &
Governance, Core Hiring Loop, Funnel/Evaluation/Decision, Insight & Reporting, Resurfacing &
Communications). Spans many sprints. A narrative/roadmap grouping (`delivery/roadmap.md`), not a
formal OpenSpec artifact boundary. **Renamed specifically to free up the word "Wave" for its new
meaning below** — using "Wave" for this older, bigger concept anywhere in new material is a
mistake.

**Sprint** — exactly one calendar week. A time-box, nothing more. Numbered `TS-SPR-XXX`.

**Wave** *(current meaning — do not confuse with Phase, above)* — a grouping of backlog items
being worked in the **same sprint**. Every item inside a wave must be independent of every other
item in it, so they can run in parallel; a wave groups by independence, not by feature, so two
items from unrelated features can share a wave. Numbered **globally and sequentially across the
whole product** (`Wave 1`, `Wave 2`, ... `Wave 11`...), never restarted per sprint, even though a
wave's file still nests under its sprint (`delivery/sprints/TS-SPR-XXX/waves/TS-SPR-XXX-WV-XXX.md`).
A sprint defaults to **one** wave unless a genuine dependency reason forces a split (see
`AGENTS.md`). Delivery happens per item, not per wave — a wave is a tracking lens, not a bundled
release.

**Feature** — the unit `/opsx:propose` actually runs against: one OpenSpec change
(`proposal.md`/`design.md`/`specs/`/`tasks.md`) covering one design-cohesion domain (e.g.
`identity-and-access`, `design-system`). There are ~12-15 of them for the whole product. A
feature's `tasks.md` is its complete backlog — every backlog item that belongs to it, each still
independently deployable even though several may share the feature's one `design.md`. See
`AGENTS.md`'s "Propose at the FEATURE level" section for why this superseded an earlier,
backlog-item-level model.

**Backlog item** — *not* the unit `/opsx:propose` runs against (that's a **Feature**, above) — a
tracked, numbered entry inside a feature's `tasks.md` and in `delivery/`'s scheduling. *"A
cohesive unit of work/code which can travel independently to dev, uat and prod"* — not a small
implementation task (that's a step inside building a backlog item), and not an entire Phase or
Feature (too large to be one deployable increment). Permanently and globally numbered `TS-BL-XXX`
— the number never changes even if the item spills into a later sprint; only its sprint/wave
assignment moves. "Independently deployable" is a code/CI-CD property of the item, not an
OpenSpec-planning property — it does not require its own change directory.

**Slice** — the original S1–S14 decomposition from the explore stage, before the manager's
sprint/wave/backlog model arrived. Superseded by backlog items (see
`exploration-notes.md` Part D), but referenced constantly in `design.md` files and spec content
written before the rename — a backlog item's spec will often say "per S8" to point at where its
reasoning originally came from. Not a live concept for new planning.

**Change** — an OpenSpec `changes/<id>/` directory (`proposal.md`, `design.md`, `tasks.md`,
`specs/`). One change per **feature** (not per backlog item — that was an earlier, superseded
model; see `AGENTS.md`). `talentsphere/` is the one permanent exception — it holds
`exploration-notes.md` only and is never proposed.

## Domain terms

**Application** *(capitalized — not the software product)* — the pairing of one candidate with
one job posting. The pipeline spine: stage, ranking score, shortlist status, interview rounds,
scorecards, and offer status all attach here, not to the candidate directly, because a candidate
can be at different stages on different postings simultaneously.

**Practice** — a data attribute on postings for filtering and reporting. Not a permission
boundary — data visibility is global-within-role except for Interviewer/Hiring Panel Member,
who are assignment-scoped.

**Hubble** — the enterprise identity/HR system TalentSphere authenticates against. Login only;
TalentSphere's own authorization is entirely separate and local. Hubble ID is also captured at
onboarding, manually, as proof of hire — not the same integration point as login.

**AIRun** — one logged record per AI invocation: model, prompt template version, input/output
*references* (never raw sensitive content), status, safety flags. Cannot be backfilled — this is
why the AI governance substrate must exist before the first real AI call, not after.

**Advisory-only** — the rule that AI output can never by itself reject, shortlist, select, offer,
hire, or close a candidate. Enforced structurally (no code path has that capability), not by
review or convention.

**Break-glass** — the audited, time-boxed permission override that lets an Administrator (who is
config-only by default) temporarily access candidate PII for a genuine support case. An ordinary
permission override with an expiry and mandatory reason, not a separate access mode.

**Resurfacing / priority lane** — candidates who didn't fit their submitted posting get
resurfaced against other open postings automatically. Candidates who were selected but never
offered get precedence in that resurfacing (the priority lane) and may carry a prior scorecard
forward as evidence, subject to completing one confirmatory interview round for the new posting.

**Evidence-source labeling** — every AI-surfaced claim (fitment, gap, scorecard dimension) must
say whether it's resume-sourced, interview-sourced, scorecard-sourced, or a human decision.
Required so an Admin's audit-log access (config-only) doesn't become a PII back door through
AI run logs.

## Where these terms get used

Full reasoning for every term above lives in `changes/talentsphere/exploration-notes.md` —
search for the term or its `D`/`C`/`G`/`S` decision ID. This file only disambiguates; it doesn't
re-argue anything.
