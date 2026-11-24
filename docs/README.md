# Docs

Status: draft v1, Nov 24, 2026. Docs are for people and AI agents. The app itself reads `../data/`.

| Doc | Read it when you want to know |
|---|---|
| [architecture.md](architecture.md) | How the system works: layers, pipeline, components, data model, API, security, reliability |
| [codebase-structure.md](codebase-structure.md) | Where code lives, dependency rules, interfaces, conventions, testing, "what do I edit" lookup |
| [organization.md](organization.md) | Timeline, roles, working agreements, definition of done, risks, submission checklist |
| [maintenance.md](maintenance.md) | How to keep docs, data, prompts and the live product in sync and healthy; incident playbook |

## Project knowledge (the why and the what)

| Doc | Content |
|---|---|
| [project/00-overview.md](project/00-overview.md) | Contest, product, pitch, scope, build plan |
| [project/01-decisions.md](project/01-decisions.md) | Decisions made and still open (update first when the team decides) |
| [project/02-data-sources.md](project/02-data-sources.md) | Real shapes of MoMo screens and bank emails |
| [project/03-risk-rules.md](project/03-risk-rules.md) | Risk logic, rule table, reason codes |
| [project/04-prompts.md](project/04-prompts.md) | Prompt design for each LLM job |
| [project/05-eval-plan.md](project/05-eval-plan.md) | Dataset, metrics, baseline comparison, adversarial tests |
| [project/06-glossary.md](project/06-glossary.md) | Vietnamese terms, categories, reason codes |
| [project/claude-context.md](project/claude-context.md) | One-file summary to paste into a Claude project |

## Docs vs data

- `docs/` = humans read it to understand.
- `data/` = code loads it at runtime (categories, merchants, name patterns, risk thresholds, message templates, few-shots). Run `python data/validate.py` after editing.
- A fact may live in both (for example the risk rules). The doc explains it, the data executes it. Change both together; see [maintenance.md](maintenance.md).

## Conventions

- **[Decision]** marks anything the team has not finalized.
- No real personal data in any doc, example or fixture.
- Start every doc with a status line and keep it current.
