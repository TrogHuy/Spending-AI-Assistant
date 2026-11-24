# Organization

How the team works for the contest. Status: proposal, Nov 24, 2026. Adjust to the real team size and update this file.

## 1. Timeline and milestones

| Date | Milestone |
|---|---|
| Nov 24 | Docs and decisions agreed; repo created |
| Nov 27 | **Registration closes.** Confirm the team is registered |
| Nov 25 to 26 | Screenshot reader working on fake screens; bare version deployed |
| Nov 28 to 29 | Payee classifier, risk engine, eval script, dashboard |
| Nov 30 to Dec 1 | Chat agent, one-tap label, pre-payment check, limits and spend cap |
| Dec 2 | Full eval, fix weak spots, PDF guide, README, slides, video |
| **Dec 3** | **Qualifying round deadline.** Submit early in the day, not at the last hour |
| Dec 4 to 10 | Keep the live product up; fix bugs; prepare final pitch and demo |
| **Dec 11** | **Final**, THE ADORA PREMIUM, Ho Chi Minh City |

Slides cannot be edited after submission, so freeze them only when the numbers are final.

## 2. Roles (for 3 people; merge if 2)

| Role | Owns | Also |
|---|---|---|
| **A: Data and eval** | Sample data generation, screen and email parsers, labeling, eval harness, metrics | Fixtures for tests |
| **B: Backend and LLM** | API, domain logic, risk engine, LLM adapter, prompts, chat agent, security limits | Spend cap and budgets |
| **C: Frontend and delivery** | UI (upload, confirm table, dashboard, chat, pre-check), deployment, 1-page PDF, slides, video | README assembly |

Everyone: write tests for what they build, review one other person's PR per day, update the docs for what they change.

Each decision in `docs/project/01-decisions.md` has one owner who records the outcome.

## 3. Working agreements

- **Trunk-based:** short-lived branches off `main`, merged within a day. Name: `feat/<topic>`, `fix/<topic>`.
- **Every merge to `main` deploys.** Keep `main` working; the live URL is what judges see.
- **PR rules:** small PRs, one reviewer, tests pass, `python data/validate.py` passes if data changed, docs updated if behavior changed. Self-merge only for docs typos.
- **Commit style:** short imperative messages (`add momo bill adapter`).
- **No secrets in git.** Use `.env` locally and host secrets in production.
- **No real personal data anywhere** (repo, tests, demo, slides, video). Fake names only.
- **Test set is locked.** Nobody tunes prompts or rules against it.

## 4. Rhythm

- 10-minute standup daily: done, next, blocked.
- One shared task board with columns Backlog, Doing, Review, Done. Tasks small enough to finish in a day.
- Decisions are made in writing: edit `01-decisions.md`, move the item from Open to Made with the reason and date.
- Demo to each other every two days on the live deployment, not localhost.

## 5. Definition of done

A task is done when:
1. It works on the live deployment.
2. Tests exist and pass (unit for domain logic, fixture-based for parsers).
3. Errors are handled (bad input gives a clear message, no crash).
4. Docs and data files are updated and `validate.py` passes.
5. No real personal data was added.

## 6. Priorities and scope control

- P0 first (import, categorize, payee type and risk, dashboard, chat, one-tap label), then P1 (pre-payment check, recurring and duplicate detection, monthly summary), then P2 only if ahead.
- Scope cuts, in order: P2 items, bank email adapter, subscription detection, extra insights, polish. Never cut: the live deployment, the eval numbers, the privacy statement.
- If a feature is not demoable in the 3-minute video or not backed by eval numbers, it gets lower priority.

## 7. Top risks and owners

| Risk | Mitigation | Owner |
|---|---|---|
| Screenshot reader accuracy | Build first; validate in code; user confirms rows; test 2 models | B with A |
| No real data | Synthetic generator in real layouts; friends' anonymized samples | A |
| "AI core" looks weak | Baseline comparison slide; LLM on ambiguity, reading and chat | B |
| Judges stress-test live app | Rate limit, spend cap, size limits, always-on host | B and C |
| Wrong numbers in chat | Tool calls over the database, exact-number tests | B |
| Privacy slip in demo | Fake data only; check video frame by frame | C |
| Time overrun | Daily scope check; cut order above | All |

## 8. Submission checklist (owner: C, checked by all)

- [ ] Team registered before Nov 27
- [ ] 1-page A4 PDF guide with live link (and test account if login is needed)
- [ ] Source .zip with README: how to run, tech used, list of existing assets, open-source libraries, AI tools used
- [ ] Slides PDF, max 10 pages, final numbers, no real personal data
- [ ] Video MP4, max 3 minutes, recorded on the live deployment, no real personal data
- [ ] Live link tested from a different network and device
- [ ] Submitted via the official form before Dec 3
- [ ] Product kept running through Dec 11
