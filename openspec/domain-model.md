# Domain Model

The entity relationships underlying TalentSphere, synthesized from `exploration-notes.md` into
one canonical reference. Field-level schemas belong to each feature's own `design.md`, not
here — this file shows what connects to what and why, not column types.

## The spine

```
                    ┌────────────────────────────┐
                    │  JobDescription             │  versioned, PM-drafts,
                    │  DRAFT → REVIEW → APPROVED  │  RM-approves, forks on edit
                    └─────────────┬───────────────┘
                                  │ 1 JD version → N postings
                                  ▼
        ┌─────────────────────────────────────────────────┐
        │  JobPosting                                      │
        │  vacancy_type: FINITE(n) | EVERGREEN              │
        │  status: draft → open → ... → closed              │
        └────────────────────┬──────────────────────────────┘
                             │
              ═══════════ THE SPINE ═══════════
                             ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Application  (candidate × posting)                       │
   │  stage, source_type: recruiter | referral | job_board |    │
   │    campus | agency | direct_application | existing_database│
   │  resume_version_ref                                        │
   └──┬────────┬──────────┬───────────┬──────────┬─────────────┘
      │        │          │           │          │
      ▼        ▼          ▼           ▼          ▼
  Ranking   Shortlist  Interview   Priority    Offer &
  Score     (reason    Rounds      Slot        Onboarding
  (see       required   (versioned  (1-5 ×      Record
  below)     on every   notes)      vacancy)    (Hubble ID)
             disposition)
      ▲
      │
 ┌────┴─────────────┐         ┌──────────────────────────────┐
 │  Candidate        │──1:N──▶│  Resume (version n)          │
 │  dedup identity   │         │  file_ref, malware scan,     │
 └──────────────────┘         │  hybrid-parsed (deterministic │
                                │  contact fields + LLM        │
                                │  enrichment)                 │
                                └──────────────────────────────┘
```

**Why `Application` exists as its own entity, not a status on `Candidate`:** a candidate can be
at different stages on different postings simultaneously — stage has to live on the
candidate-posting pairing, not the candidate.

## Ranking and AI provenance

```
RankingScore = f( resume_version, jd_version, prompt_template_version, model_version )
```

Every AI-produced score, summary, or scorecard draft records this full tuple, not just a number
— without it, a score can't be explained or reproduced once the JD or the model changes
underneath it. Backs onto one `AIRun` record per invocation (see `glossary.md`).

`MatchSuggestion` is the resurfacing entity — lighter than `Application`, promoted into a real
`Application` only when a human acts on it. Keeps the resurfacing pool from silently inflating
pipeline metrics.

## Interview and evaluation

```
Application ──1:N──▶ InterviewRound ──1:N──▶ InterviewNote (versioned per edit,
                          │                    no lock — see C-08)
                          │
                          ▼
                    Scorecard (one per Application, consolidated across all
                               rounds — see C-05 — AI-drafted, human-approved)
```

A `Scorecard` can link to **multiple** Applications when priority-lane carry-forward applies
(D17) — the carried scorecard is cited as prior evidence, and the candidate still completes one
confirmatory round for the new posting.

## Access and governance

```
User ──N:M──▶ Role ──N:M──▶ PagePermission (action: View|Create|Edit|Delete|
                                              Approve|Run AI|Export|Assign|Administer)

User ──1:N──▶ UserPermissionOverride (grant|deny, mandatory reason, deny wins)

Every material write ──▶ AuditLog (actor, action, target, previous/new value,
                                    reason, correlation ID — references only,
                                    never raw personal data — see D6)
```

Nine roles: Recruiter, Practice Manager, Recruitment Manager, Interviewer, Hiring Panel Member,
Application Administrator, System Administrator, Auditor, AI Service Account. See
`glossary.md` and `exploration-notes.md` C-02.

## AI platform

```
PromptTemplate (10 families, versioned) ──▶ AIGateway ──▶ AIRun (logged BEFORE
                                                            invocation, not after)
                                                                │
                                        ┌───────────────────────┼───────────────────────┐
                                        ▼                       ▼                       ▼
                              Advisory insight only —   DisclosureRecord        OverrideRecord
                              no transition capability  (append-only: who saw   (human diverged,
                                        │                this output, when)      reason required)
                                        ▼
                              every claim carries a source label
                              + evidence reference (resume | interview
                              note | scorecard | human decision)
```

The gateway is the *only* egress to a model provider. See `changes/ai-platform-governance/design.md`
D4 for why logging precedes invocation, and D6 for why advisory-only is structural rather than
reviewed — including the three enforcement sides it is split across. Both carry forward decisions
originally recorded as D9 and D10 in `changes/talentsphere-wave-1-foundation/design.md`, **not** in
`exploration-notes.md`, whose D09/D10 are the tech stack and the calendar platform.

**Corrected 2026-08-25 — this line previously read:** *"See `exploration-notes.md` D9/D10 for why
logging precedes invocation and why advisory-only is structural, not reviewed."* That pointed at
the wrong document: `exploration-notes.md`'s own D09 and D10 are the tech-stack and
calendar-platform decisions, not AI governance — the citation was for the right idea, wrong file.
Found and fixed during `ai-platform-governance`'s propose conversation.

**`DisclosureRecord` is a separate record rather than a field on `AIRun`**, because the run is
written before invocation and is append-only: at write time, whether the output will later be shown
to an interviewer is unknowable, and it cannot be set afterwards. One output is also disclosed to
many actors on many occasions. It exists to answer `D05`'s anchoring question — whether a human
judgement was formed before or after seeing AI output. See
[D05](changes/talentsphere/exploration-notes.md#d05--ai-context-visible-to-interviewers)'s dated
correction block, added 2026-08-25.

**Corrected 2026-08-27 — the spine diagram's `source` field previously read:** *"stage, source:
DIRECT | RESURFACED | PRIORITY_LANE"*. That was a placeholder sketch predating `G-10`'s actual
decision. `G-10` (adopted from the reference spec's `CAN-007`) settled the real field as
`candidate_posting_applications.source_type`, enumerated `recruiter | referral | job_board |
campus | agency | direct_application | existing_database` — already the live contract
`candidate-intake`'s `TS-BL-046` (task 6.12) writes, and the substrate
`insight-and-reporting`'s resurfacing-yield metric reads (`design.md` D6 there). The three-value
sketch was never built and does not describe any decided vocabulary; found while proposing
`resurfacing-and-communications`, whose `TS-BL-075` writes `existing_database` into this same
field for resurfaced Applications rather than inventing a parallel classification.

## What's deliberately not modeled yet

No `Client`/Account entity (no evidence TalentSphere needs one). No skills taxonomy beyond
freeform resume text (a v2 concern). No calendar/communication entities beyond what
`Notification` already covers for internal delivery — candidate-facing communication is gated on
legal sign-off (`OD-005`) and out of scope until then.
