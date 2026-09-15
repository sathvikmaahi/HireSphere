# HireSphere — Product Architecture

Canonical map of **product feature domains** layered onto the HireSphere platform stack.
Behavioral detail lives in OpenSpec (`openspec/specs/`). Branch delivery lives in
[feature-branches.md](./feature-branches.md) and [implementation-playbook.md](./implementation-playbook.md).

**Stack (locked):** React 19 + Vite + Tailwind v4 · Fastify on Cloud Run · Pub/Sub workers ·
Cloud SQL PostgreSQL 15 + Drizzle · GCS · Vertex AI (Gemini) · Terraform · Cloud Build.

**Auth (locked):** HireSphere-owned **username + password** login with JWT + bcrypt.
**Not** Hubble SSO / external IdP. Optional `hubble_id` remains an onboarding reference field only.

**Governing rule:** AI recommends. A human decides — structurally (AI cannot trigger hiring
transitions), not by convention.

---

## Domain map

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Design system          App shell, tokens, tables, forms, notifications UI │
├─────────────────────────────────────────────────────────────────────────┤
│  Identity & access      Custom login · activation · sessions · RBAC         │
│  AI platform governance Gateway · prompt registry · AIRun · advisory-only │
├─────────────────────────────────────────────────────────────────────────┤
│  Hiring & postings      JD workspace · postings · vacancy · closure         │
│  Candidate intake       Upload · scan · parse · OCR · dedup · versions      │
│  Matching & ranking     Embeddings · vector retrieval · ranking board       │
│  Interview pipeline     Shortlist · schedule · console · scorecards         │
│  Decision & offers      Priority picks · offers · onboarding · closure      │
├─────────────────────────────────────────────────────────────────────────┤
│  Insight & reporting    Dashboards · aging · audit · AI agreement · export  │
│  Communications         Internal notify · resurfacing · calendar stub       │
└─────────────────────────────────────────────────────────────────────────┘
         ▲                         ▲
         │                         │
   Platform core            Async workers
   (API, DB, audit)         (Pub/Sub → Cloud Run)
```

---

## 1. Identity and access

**Purpose:** Own the login experience and who may act in the system.

| Capability | Behavior |
|---|---|
| Custom login page | Branded `/login` on bare shell — username + password (not Hubble) |
| Sessions | Short-lived access JWT + rotated refresh tokens |
| Activation | `pending → active`; non-active users blocked at API |
| RBAC | Roles × permissions; page/action matrix; server-side enforcement |
| Admin cockpit | Users, roles, matrix, permission explanation |
| Audit | Auth events and material writes → `audit_events` |

**Branches:** `feat/auth-rest-login`, `feat/user-activation-access`, `feat/admin-cockpit-rbac`  
**Specs:** `auth`, `user-activation`, `admin-rbac`

---

## 2. AI platform governance

**Purpose:** One governed path for every model call before any hiring AI feature runs.

| Capability | Behavior |
|---|---|
| AI gateway | Sole egress to Vertex AI; rate limits; graceful degradation |
| Prompt registry | Versioned templates; one active version per name |
| AIRun logging | Write `ai_runs` **before** invoke; model + prompt versions pinned |
| Advisory-only | No AI path can shortlist, select, offer, hire, or close |
| Feedback | Thumbs + comment on AI outputs |
| Eval harness | Regression corpus gates prompt promotion (CI) |

**Prompt families (minimum):** `jd_draft`, `posting_draft`, `resume_extraction`,
`candidate_rank`, `fitment_summary`, `gap_summary`, `interview_questions`,
`interview_note_summary`, `scorecard_draft`, `resurfacing`

**Branches:** `feat/responsible-ai-governance` (partial early + full UI late)  
**Spec:** `responsible-ai`

---

## 3. Design system

**Purpose:** One visual and layout system so every feature ships into the same shell.

| Capability | Behavior |
|---|---|
| Foundations | Design tokens (light/dark), Plus Jakarta Sans, JetBrains Mono |
| App shell | Authenticated / bare / presentation; fixed navy sidebar |
| Patterns | Browsable grid, detail split (280px rail), dense data table, forms |
| Notifications UI | Toasts / in-app notification surfaces |

**Branches:** `infra/design-system-app-shell`  
**Specs:** `app-shell` · guides: `design-spec.md`, `layout-spec.md`

---

## 4. Hiring and postings

**Purpose:** Create the jobs candidates are measured against.

| Capability | Behavior |
|---|---|
| JD workspace | AI draft + human approval; versioned; approved before open |
| Postings | Finite or unlimited vacancies; draft → open → hold → closed |
| Compliance text | Posting body / EEO blocks as required |
| Closure | Auto-close finite when filled; evergreen manual only |

**Branches:** `feat/job-description-workspace`, `feat/job-posting-management`,
`feat/posting-closure-rules`  
**Specs:** `job-descriptions`, `job-postings`

---

## 5. Candidate intake

**Purpose:** Securely ingest and structure candidate evidence.

| Capability | Behavior |
|---|---|
| Resume upload | PDF-only, size/MIME gates, async processing |
| Malware scan | Scan before parse; delete infected objects |
| Parse / OCR | Text extract; Document AI when density is low |
| Identity / dedup | Email + normalized phone; merge flow |
| Versioning | Resume versions linked to applications |

**Domain spine:** **Application** = candidate × posting (stage lives here, not on Candidate alone).

**Branches:** `feat/resume-upload-pipeline`, `feat/candidate-database`  
**Specs:** `resume-upload`, `candidates`

---

## 6. Matching and ranking

**Purpose:** Retrieve and rank fitment with full provenance.

| Capability | Behavior |
|---|---|
| Embeddings | Resume sections → vectors (async worker) |
| Vector retrieval | Vertex AI Vector Search; auth applied at **query time** |
| Ranking board | Score + fitment + gaps + interview Qs |
| Provenance | `RankingScore = f(resume_ver, jd_ver, prompt_ver, model_ver)` |
| Human override | Re-rank / ignore AI; never auto-advance stage |

**Branches:** `feat/ai-candidate-ranking` (+ embedding/vector tasks in same domain)  
**Spec:** `ai-ranking`

---

## 7. Interview pipeline

**Purpose:** First human dispositions after ranking.

| Capability | Behavior |
|---|---|
| Shortlisting | Mandatory reason on every disposition |
| Scheduling | Interview tasks; assignee notify (internal) |
| Console | Multi-round structured notes; versioned (never overwrite) |
| Scorecards | Interviewed-only; AI draft; human approve |
| Consolidation | One scorecard per Application across rounds |

**Branches:** `feat/shortlisting-workflow`, `feat/interview-scheduling`,
`feat/interview-console`, `feat/scorecard-center`, `feat/ai-assisted-scorecards`  
**Specs:** `shortlisting`, `interview-scheduling`, `interview-console`, `scorecards`

---

## 8. Decision and offers

**Purpose:** Select, offer, onboard, and close the requisition.

| Capability | Behavior |
|---|---|
| Priority selection | Up to 5 ranked picks per finite vacancy |
| Offers | draft → extended → accepted / declined / withdrawn |
| Onboarding | Checklist + optional external employee ID (`hubble_id`) |
| Closure | Finite filled → auto-close; lifecycle events |

**Branches:** `feat/priority-selection`, `feat/offer-onboarding-tracker`,
`feat/posting-closure-rules`  
**Specs:** `priority-selection`, `offers-onboarding`, `job-postings`

---

## 9. Insight and reporting

**Purpose:** Aggregate “how are we doing” without dumping raw PII.

| Capability | Behavior |
|---|---|
| Workload / pipeline dashboards | Recruiter, HM, admin views |
| Closure & aging | SLA / aging thresholds → nudges and views |
| AI audit reporting | Agreement rates, model/prompt usage |
| Exports | Async CSV/PDF to GCS; role-based redaction |
| Audit search | Actor, action, resource, time range |

**Branches:** `feat/dashboards-reports-audit`  
**Spec:** `dashboards-audit`

---

## 10. Communications

**Purpose:** Keep humans in the loop; resurface talent; defer candidate email until gated.

| Capability | Behavior |
|---|---|
| Internal notifications | In-app + email to HireSphere users only |
| Resurfacing | Suggest existing candidates for new/open postings |
| Priority lane | Selected-but-not-offered candidates ranked first |
| MatchSuggestion | Light record → Application only when a human acts |
| Calendar | Stub / integration placeholder in v1 |
| Candidate-facing email | **Gated** — not enabled until product/legal sign-off |

**Branches:** `feat/candidate-resurfacing`, notification work in platform + interview scheduling  
**Specs:** `candidate-resurfacing`, `interview-scheduling` (notify), `dashboards-audit` (as needed)

---

## Dependency order (architecture)

```
Infrastructure → Design system → Identity & access → AI platform governance
        → Hiring & postings → Candidate intake → Matching & ranking
        → Interview pipeline → Decision & offers
        → Insight & reporting ∥ Communications (resurfacing after ranking + decisions)
```

AI governance (prompt registry + AIRun) must land **before** the first real Vertex call
(JD draft or ranking).

---

## Related documents

| Document | Role |
|---|---|
| [architecture.md](./architecture.md) | System context, data flows, security |
| [implementation-playbook.md](./implementation-playbook.md) | Step prompts |
| [feature-branches.md](./feature-branches.md) | Branch names |
| [../openspec/project.md](../openspec/project.md) | Spec index |
| [../share/hiresphere-overview.html](../share/hiresphere-overview.html) | Investor / stakeholder overview |
