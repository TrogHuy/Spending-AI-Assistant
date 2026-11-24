# Codebase Structure

Status: proposal, Nov 24, 2026. Assumes Python + FastAPI + a thin frontend. If the team picks another stack, keep the same layers and folder responsibilities. **[Decision]** stack.

## 1. Repository layout

```
/
├── README.md                 how to run, stack, assets, libraries, AI tools used (contest requirement)
├── docs/                     docs for people and agents (see docs/README.md)
├── data/                     JSON the app loads at runtime + validate.py
├── prompts/                  versioned prompt files
├── app/                      backend
│   ├── main.py               app factory, routers, middleware
│   ├── config.py             env vars and settings (one place)
│   ├── api/                  routes and request/response schemas
│   │   ├── session.py
│   │   ├── imports.py
│   │   ├── transactions.py
│   │   ├── insights.py
│   │   ├── chat.py
│   │   ├── precheck.py
│   │   └── deps.py           session, rate limit, auth dependencies
│   ├── services/             use-case orchestration, no business rules inside
│   │   ├── import_service.py
│   │   ├── analysis_service.py     the analyze() Facade
│   │   ├── chat_service.py
│   │   ├── precheck_service.py
│   │   └── label_service.py
│   ├── domain/               PURE logic: no HTTP, no DB, no LLM imports
│   │   ├── models.py         Transaction, Payee, RawTransaction, enums
│   │   ├── normalize.py
│   │   ├── dedupe.py
│   │   ├── enrich.py         payee profile, newness, amount ratio
│   │   ├── categorize/       interface + rules_only + merchant lookup + keyword rules
│   │   ├── payee/            interface + name rules + scoring
│   │   ├── risk.py           rule table engine (reads data/risk_config.json)
│   │   ├── insights/         one file per detector
│   │   └── explain.py        reason-code templates
│   ├── adapters/
│   │   ├── inputs/           one adapter per source
│   │   │   ├── base.py       InputAdapter protocol
│   │   │   ├── router.py     identify screen type, reject unsupported
│   │   │   ├── momo_bank_out.py
│   │   │   ├── momo_wallet_in.py
│   │   │   ├── momo_bill.py
│   │   │   ├── bank_email_vcb.py
│   │   │   └── pasted_text.py
│   │   ├── llm/
│   │   │   ├── base.py       LLMClient protocol
│   │   │   ├── provider_x.py concrete provider
│   │   │   ├── cache.py
│   │   │   └── budget.py     per-session and global spend accounting
│   │   └── storage/
│   │       ├── db.py
│   │       └── repositories.py
│   ├── llm_tasks/            thin wrappers that call prompts + validate output
│   │   ├── read_screen.py
│   │   ├── classify_payee.py
│   │   ├── categorize_batch.py
│   │   ├── polish_explanation.py
│   │   ├── chat_agent.py     tool definitions and loop
│   │   └── read_seller_chat.py
│   └── security/
│       ├── limits.py         rate limit, size limits
│       └── sanitize.py       delimit untrusted text, redact for logs
├── web/                      frontend
│   ├── pages or src/         upload, confirm table, dashboard, chat, precheck
│   └── i18n/                 vi.json (primary), en.json
├── eval/                     evaluation harness (see docs/project/05-eval-plan.md)
│   ├── generate/             HTML templates + script to render fake screens and emails
│   ├── dataset/              labeled fake data, dev/ and test/ (test locked)
│   ├── run_eval.py           one command, prints the metrics table
│   ├── baselines.py          rules-only vs rules+LLM via Strategy swap
│   └── results/              scores per prompt version
├── tests/
│   ├── unit/                 domain tests (risk rules, normalize, scoring)
│   ├── adapters/             parser tests on fake fixtures
│   ├── api/                  route tests with LLM mocked
│   └── security/             injection and abuse cases
├── scripts/                  seed_demo.py, precompute_demo_cache.py, delete_expired_sessions.py
├── .env.example              keys and limits, no real secrets
└── Dockerfile / deploy config
```

## 2. Dependency rules (enforce in review)

| Rule | Why |
|---|---|
| `domain/` imports nothing from `api/`, `services/`, `adapters/`, `llm_tasks/` | Pure, unit-testable, no path from LLM text to a flag |
| `risk.py` has no LLM call and no network access | Flags are deterministic |
| `api/` calls `services/` only | Routes stay thin |
| `services/` orchestrate; rules live in `domain/` | One place for each decision |
| Only `adapters/llm/` talks to a provider | Swap providers in one file |
| Only `adapters/storage/` talks to the database | Easy to swap SQLite and Postgres |
| `llm_tasks/` validate every model output against a schema before returning | Untrusted output never reaches the domain unchecked |
| Untrusted text enters prompts only through `security/sanitize.py` | One audited path |
| Data files are loaded once at startup through one loader | No scattered file reads |

## 3. Key interfaces (sketch)

```python
class InputAdapter(Protocol):
    def can_handle(self, payload) -> float: ...        # confidence it matches
    def parse(self, payload) -> list[RawTransaction]: ...

class Categorizer(Protocol):
    def categorize(self, txs: list[Transaction]) -> list[CategoryResult]: ...

class PayeeClassifier(Protocol):
    def classify(self, payee: Payee) -> PayeeResult: ...   # may return uncertain

class LLMClient(Protocol):
    def complete_json(self, task: str, prompt_version: str, data: dict, schema: dict) -> dict: ...
    def complete_vision_json(self, task: str, prompt_version: str, image: bytes, schema: dict) -> dict: ...
```

The risk engine is a function: `evaluate(tx_context, config) -> RiskResult`. No class hierarchy needed.

## 4. Naming and conventions

- Python: snake_case modules, type hints on public functions, one concept per file, small functions.
- Reason codes, category IDs and flag values are constants from one module, matching `data/*.json`. Never retype the strings.
- Prompts: `prompts/<task>_v<N>.md`. The code names the version explicitly; changing a version is a reviewed change.
- Money is an integer number of VND. Timestamps are timezone-aware (Asia/Ho_Chi_Minh).
- UI text lives in `web/i18n/`, never hard-coded in components.
- No real personal data in fixtures, tests, logs or examples. Fake names only.

## 5. Configuration

| Variable | Purpose |
|---|---|
| `LLM_API_KEY`, `LLM_MODEL_TEXT`, `LLM_MODEL_VISION` | Provider access |
| `LLM_GLOBAL_BUDGET_VND` or USD | Hard spend cap |
| `LLM_SESSION_BUDGET` | Per-session cap |
| `RATE_LIMIT_PER_MIN` | Per-session request limit |
| `MAX_IMAGES_PER_IMPORT`, `MAX_IMAGE_MB` | Input limits |
| `SESSION_TTL_HOURS` | Data retention |
| `DATABASE_URL` | Storage |
| `ENABLE_LLM` | Kill switch for rules-only mode |

Thresholds for the risk rules are not env vars. They live in `data/risk_config.json` so they are versioned and reviewed.

## 6. Testing strategy

| Level | What | Notes |
|---|---|---|
| Unit | Each risk rule (one test per reason code), normalize (amounts like 1tr5), scoring, dedupe | Fast, no LLM |
| Adapter | Parsers against fake fixtures of each screen and email shape | Include cut-off, dark mode, wrong screen |
| API | Routes with the LLM client mocked | Check limits and errors |
| Security | Injection strings in name, note and image text; oversized input | Flag must not change |
| Eval | `eval/run_eval.py` on dev; test run once per release | Metrics table and baseline comparison |
| Smoke | After each deploy: load demo, run a chat question, run a pre-check | Scripted |

A change to `data/*.json` runs `python data/validate.py` in CI. A change to prompts or rules runs the dev-set eval.

## 7. What lives where (quick lookup)

| I want to change... | Edit |
|---|---|
| A risk threshold or rule | `data/risk_config.json` + tests + `docs/project/03-risk-rules.md` |
| A category | `data/categories.json` + prompts if definitions changed |
| A merchant | `data/merchants.json` |
| Wording of a warning | `data/reason_templates.json` |
| A prompt | `prompts/<task>_vN.md` (new version), then eval |
| How a screen is read | `app/adapters/inputs/<adapter>.py` + fixture + test |
| An API route | `app/api/` + `app/services/` |
| UI text | `web/i18n/` |
