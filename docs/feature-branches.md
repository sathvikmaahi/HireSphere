# HireSphere — Feature Branch Reference

Quick index of all implementation branches. **Step-by-step prompts:** [implementation-playbook.md](./implementation-playbook.md). Full detail in [implementation-plan.md](./implementation-plan.md).

## Infrastructure (merge first)

| Branch | Description |
|---|---|
| `infra/gcp-terraform` | **All GCP infra** — Terraform modules, environments (dev/staging/prod), Cloud Build, Dockerfiles |
| `infra/monorepo-scaffold` | App monorepo, docker-compose local dev, platform adapters |
| `infra/database-foundation` | Drizzle, migrations, seeds |
| `infra/design-system-app-shell` | Design tokens, sidebar shell, page templates |

See [gcp-infrastructure.md](./gcp-infrastructure.md) for Terraform layout and [../openspec/project.md](../openspec/project.md) for OpenSpec requirements.

## Features (in recommended merge order)

| # | Branch | Feature |
|---|---|---|
| 1 | `feat/auth-rest-login` | REST Login API (username & password) |
| 2 | `feat/user-activation-access` | User activation and access enforcement |
| 3 | `feat/admin-cockpit-rbac` | Admin cockpit: users, roles, permissions, page/action matrix |
| 4 | `feat/job-description-workspace` | Job Description Workspace with AI drafting and human approval |
| 5 | `feat/job-posting-management` | Job Posting Management (finite & unlimited vacancies) |
| 6 | `feat/resume-upload-pipeline` | PDF resume upload: validation, malware scan, parse, OCR, storage |
| 7 | `feat/candidate-database` | Candidate DB: dedup, resume versions, posting links, history |
| 8 | `feat/ai-candidate-ranking` | AI ranking, fitment/gap summaries, interview questions |
| 9 | `feat/candidate-resurfacing` | Existing-candidate resurfacing + selected-not-offered priority lane |
| 10 | `feat/shortlisting-workflow` | Shortlisting with mandatory reason capture |
| 11 | `feat/interview-scheduling` | Interview scheduling task workflow |
| 12 | `feat/interview-console` | Interview console: notes, rounds, versioning, AI-readable storage |
| 13 | `feat/scorecard-center` | Scorecard Center (interviewed candidates only) |
| 14 | `feat/ai-assisted-scorecards` | AI-assisted scorecards with human review and approval |
| 15 | `feat/priority-selection` | Priority selection (up to 5 per finite vacancy) |
| 16 | `feat/offer-onboarding-tracker` | Offer & Onboarding Tracker with Hubble ID capture |
| 17 | `feat/posting-closure-rules` | Closure rules for finite postings; manual evergreen controls |
| 18 | `feat/dashboards-reports-audit` | Dashboards, reports, audit logs, AI run logs, exports |
| 19 | `feat/responsible-ai-governance` | Responsible AI: prompt governance, model versioning, feedback |

## Documentation

| Branch | Description |
|---|---|
| `feat/docs` | Planning docs, OpenSpec specs (`openspec/`), architecture |

## Excluded from scope

- `feat/hubble-sso-login` — **not in scope** (use REST auth instead)
- Miracle branding assets or theming — **not in scope** (HireSphere branding only)

## Branch workflow

```bash
# Example: start feature 1 after infra branches are merged to main
git checkout main && git pull
git checkout -b feat/auth-rest-login
# ... implement ...
# Open PR → feat/auth-rest-login → main
```

Naming rules:
- All lowercase, hyphen-separated slugs
- Prefix `infra/` for infrastructure branches
- Prefix `feat/` for product feature branches
- One feature per branch; no bundling unrelated work
