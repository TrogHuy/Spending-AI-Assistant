# Maintenance

How to keep the docs, data, prompts, code and the live product healthy, during the contest (through Dec 11) and after. Status: proposal, Nov 24, 2026.

## 1. Single source of truth

| Fact | Lives in | Mirrors in |
|---|---|---|
| Why we chose something | `docs/project/01-decisions.md` | Other docs link to it |
| Risk logic explanation | `docs/project/03-risk-rules.md` | `data/risk_config.json` (executable version) |
| Categories | `data/categories.json` | `docs/project/06-glossary.md` (readable list) |
| Reason codes | `data/reason_templates.json` | Constants in code, glossary |
| Architecture | `docs/architecture.md` | `docs/codebase-structure.md` references it |
| Prompts | `prompts/*.md` | `docs/project/04-prompts.md` explains them |

Rule: **change the source, then update its mirrors in the same PR.**

## 2. When X changes, update Y

| Change | Update |
|---|---|
| A risk threshold or rule | `risk_config.json`, unit test for that rule, `03-risk-rules.md`, re-run dev eval |
| Add or rename a category | `categories.json`, merchant categories, few-shots, glossary, dashboard labels; run `validate.py` |
| Add a merchant | `merchants.json`; run `validate.py` (checks duplicate aliases) |
| New prompt version | New file `prompts/<task>_vN.md`, point the code at it, run dev eval, log the score, update `04-prompts.md` if the contract changed |
| New input source | Adapter + fixtures + tests, `02-data-sources.md`, `architecture.md` adapter table |
| Open decision resolved | Move it in `01-decisions.md`; update every doc that said **[Decision]** about it |
| API route added or changed | `architecture.md` API table, tests |
| New environment variable | `.env.example`, `codebase-structure.md` config table, deploy settings |

## 3. Prompt and rule versioning

- Never edit a released prompt in place. Create `_vN+1`, evaluate on dev, compare, then switch.
- Keep a short score log per version in `eval/results/` (date, version, metrics, note).
- Few-shot examples come from the dev set only. Never from the test set.
- A threshold change is a data change and gets the same review as code.

## 4. Eval as a gate

- Run `eval/run_eval.py` on the dev set after every change to prompts, rules, data files or parsers.
- Do not merge a change that drops a key metric (amount accuracy, payee precision, flag rate band, injection-blocked rate) without a written reason.
- Run the locked test set once per release candidate and once for the final slides. Record the date and commit hash next to the numbers.
- When user corrections or judge-found failures appear, add them to the dev set as new cases.

## 5. Dataset hygiene

- All data is fake and generated or hand-written in real layouts. No real names, phones, account numbers, customer codes or addresses. Re-check before each commit and before any screen recording.
- Keep the generator in git so the dataset is reproducible. Keep dev and test in separate folders; the test folder is locked.
- If a teammate shares a real screenshot to learn the layout, keep it out of the repo and delete it after building the fake template.

## 6. Dependencies and secrets

- Pin versions. Add a dependency only if it saves real time. List every library and AI tool in the README (contest requirement).
- Secrets only in env vars and host secret storage. Rotate the LLM key if it leaks. Never print keys or raw inputs in logs.
- Keep `.env.example` current.

## 7. Operating the live product (now through Dec 11)

**Before the judging window (by Nov 30):** always-on host, health check, spend cap set, rate limits set, session TTL set, demo dataset precomputed and cached, smoke test passes.

**Daily:** open the live URL from a different device; run the smoke test (load demo, one chat question, one pre-check, one screenshot import); check error counts and LLM spend.

**Release:** merge to `main` deploys. After deploy, run the smoke test. Keep the previous version redeployable.

**Rollback:** redeploy the last good commit first, investigate second.

## 8. Incident playbook

| Symptom | Action |
|---|---|
| LLM provider errors or slow | Set `ENABLE_LLM=false`; app falls back to rules-only with a badge; demo data still works; restore when healthy |
| Spend cap hit | Confirm it is real usage, not abuse; raise only deliberately; cached and rules paths continue |
| Site down or slow | Check host status, restart, redeploy last good commit |
| Reader misreads many screens | Disable the failing adapter path or fall back to paste text; add the failing case to the dev set |
| Suspected abuse or injection attempt | Check logs by event and ID (no raw text); tighten rate limits; add the case to `tests/security/` |
| Real personal data found in repo or demo | Remove it, rewrite the commit if needed, rotate anything exposed, re-check slides and video |

## 9. Data retention and privacy operations

- Sessions expire after the configured TTL; run the cleanup script on a schedule. The delete endpoint removes a session immediately.
- Raw images and email text are dropped after parsing. Check this in a test.
- Logs contain event names and IDs only. Review a log sample once before the judging window.
- The slides and README state what is stored and for how long. Keep that statement true.

## 10. Documentation upkeep

- Every doc starts with a status line (draft or final, date). Update it when you edit.
- Mark unresolved items **[Decision]** and keep them listed in `01-decisions.md`.
- Keep docs short and concrete: tables for facts, prose for reasons.
- Update `docs/project/claude-context.md` whenever a major decision changes, so new AI chats start with the right context.
- Weekly during the contest (and once more before the final): read `docs/README.md` top to bottom and fix anything stale.

## 11. After the contest

- Freeze a tagged release matching the submission (zip, slides, video).
- Decide whether to continue; if yes, revisit the cut features (bank email adapters, subscription detection, family view), replace the seed merchant list with real data, and re-run the full eval on new data.
- If not continuing, take the live product down and delete stored data.
