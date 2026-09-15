# HireSphere — Feature Branch Reference

Quick index of implementation branches grouped by **product domain**.  
**Architecture:** [product-architecture.md](./product-architecture.md) · **Prompts:** [implementation-playbook.md](./implementation-playbook.md) · **Detail:** [implementation-plan.md](./implementation-plan.md)

## Infrastructure (merge first)

| Branch | Description |
|---|---|
| `infra/gcp-terraform` | GCP Terraform, Cloud Build, Dockerfiles |
| `infra/monorepo-scaffold` | pnpm monorepo, docker-compose, platform adapters |
| `infra/database-foundation` | Drizzle, migrations, seeds |
| `infra/design-system-app-shell` | **Design system** — tokens, shell, page templates |

## Product domains → branches

### Identity and access *(HireSphere login — not Hubble)*

| # | Branch | Feature |
|---|---|---|
| 1 | `feat/auth-rest-login` | Custom branded login (username + password, JWT) |
| 2 | `feat/user-activation-access` | Activation and access enforcement |
| 3 | `feat/admin-cockpit-rbac` | Users, roles, permissions, page/action matrix |

### AI platform governance

| # | Branch | Feature |
|---|---|---|
| 19a | `feat/responsible-ai-governance` | **Partial:** prompt registry + AIRun substrate (merge early) |
| 19b | `feat/responsible-ai-governance` | **Full:** governance UI, feedback, eval harness |

### Hiring and postings

| # | Branch | Feature |
|---|---|---|
| 4 | `feat/job-description-workspace` | JD workspace, AI draft, human approval |
| 5 | `feat/job-posting-management` | Finite & unlimited postings |
| 17 | `feat/posting-closure-rules` | Closure rules + evergreen controls |

### Candidate intake

| # | Branch | Feature |
|---|---|---|
| 6 | `feat/resume-upload-pipeline` | PDF upload, scan, parse, OCR, storage |
| 7 | `feat/candidate-database` | Dedup, versions, Applications, history |

### Matching and ranking

| # | Branch | Feature |
|---|---|---|
| 8 | `feat/ai-candidate-ranking` | Embeddings, vector retrieval, ranking board, fitment/gaps |

### Interview pipeline

| # | Branch | Feature |
|---|---|---|
| 10 | `feat/shortlisting-workflow` | Shortlisting with mandatory reason |
| 11 | `feat/interview-scheduling` | Scheduling + internal notification |
| 12 | `feat/interview-console` | Notes, rounds, versioning |
| 13 | `feat/scorecard-center` | Scorecard center (interviewed only) |
| 14 | `feat/ai-assisted-scorecards` | AI scorecards + human approval |

### Decision and offers

| # | Branch | Feature |
|---|---|---|
| 15 | `feat/priority-selection` | Up to 5 picks per finite vacancy |
| 16 | `feat/offer-onboarding-tracker` | Offers, onboarding, optional `hubble_id` |

### Insight and reporting

| # | Branch | Feature |
|---|---|---|
| 18 | `feat/dashboards-reports-audit` | Dashboards, aging, audit, AI logs, exports |

### Communications

| # | Branch | Feature |
|---|---|---|
| 9 | `feat/candidate-resurfacing` | Resurfacing, MatchSuggestion, priority lane |
| — | *(with 11, platform)* | Internal email / in-app notifications; calendar stub |

## Recommended merge order

1. Infra (`gcp-terraform` → `monorepo-scaffold` → `database-foundation` → `design-system-app-shell`)
2. Identity & access (1 → 2 → 3)
3. AI governance **partial** (19a)
4. Hiring & postings (4 → 5)
5. Candidate intake (6 → 7)
6. Matching & ranking (8)
7. Interview pipeline (10 → 14)
8. Decision & offers (15 → 16 → 17)
9. Insight (18) ∥ Communications / resurfacing (9)
10. AI governance **full UI** (19b)

## Excluded from scope

- `feat/hubble-sso-login` — **not in scope** (HireSphere REST auth only)
- Miracle branding — **not in scope**
- Candidate-facing email — gated until product unlock

## Branch workflow

```bash
git checkout main && git pull
git checkout -b feat/auth-rest-login
# implement per docs/implementation-playbook.md
# Open PR → main
```

Naming: lowercase hyphen-separated; `infra/` or `feat/`; one feature per branch.
