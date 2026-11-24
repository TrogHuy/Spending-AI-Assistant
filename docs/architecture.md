# Architecture

Status: draft v1, Nov 24, 2026. Stack items marked **[Decision]** are open. Principles are settled (see `project/01-decisions.md`).

## 1. Goals and constraints

- Live, usable online by judges with no setup. AI is core, not a side feature.
- Read-only: the app analyzes and warns, it never moves money.
- About 9 days to build, 2 to 3 people. So: one deployable service, no queue, no microservices.
- Unreliable inputs (screenshots, bank emails) and untrusted text (names, notes, text inside images).
- Must survive judges stress-testing it: rate limits, spend cap, graceful failure, no cold starts.

## 2. Principles

1. **LLM for semantics, code for decisions.** The LLM reads images, resolves ambiguous names, words explanations and runs chat. Code does arithmetic, detection and risk flags.
2. **Cascade, cheapest first.** User rule, then merchant list, then keyword rules, then LLM. The LLM sees only what is left.
3. **Abstain, never force.** `uncertain` payee type, `unknown` newness, `null` fields.
4. **Evidence everywhere.** Every flag, insight and answer carries the rule or transaction IDs that produced it.
5. **Untrusted text is data.** Delimited in prompts, never used to pick tools or arguments.
6. **Degrade, do not die.** LLM down means rules-only with a visible badge. Demo data works without any LLM call.
7. **Data minimization.** Parse, keep needed fields, drop raw images and email text.

## 3. System context

```
              +-----------+
   User ----> |  Web UI   |  upload screenshots / paste text / demo button / chat / pre-pay check
              +-----+-----+
                    |  HTTPS (REST + SSE for progress)
              +-----v------------------------------+
              |         Backend (FastAPI)          |
              |  API -> Services -> Domain         |
              +--+----------+-----------+----------+
                 |          |           |
          +------v--+  +----v-----+  +--v-----------+
          | Storage |  | LLM      |  | Data files   |
          | (DB)    |  | provider |  | (data/*.json)|
          +---------+  +----------+  +--------------+
```

External dependencies: one LLM provider (vision and text, JSON output), one database, one host. Nothing connects to MoMo or any bank.

## 4. Layers

```
API layer        routes, request validation, auth/session, rate limit, error mapping
Service layer    ImportService, AnalysisService, ChatService, PreCheckService, LabelService
Domain layer     Transaction, Payee, Rule engine, Risk engine, Insight detectors (pure, no I/O)
Adapters         input adapters (screens, emails, text), LLM adapter, storage repositories
```

Dependency direction: API, then services, then domain. Adapters implement interfaces the domain and services define. **The domain layer never imports the LLM adapter or the database.** That keeps the risk engine pure and unit-testable, and makes it impossible for LLM output to set a flag directly.

## 5. Processing pipeline

```
[Input] -> [Read] -> [Validate] -> [Confirm] -> [Normalize] -> [Dedupe] -> [Enrich]
        -> [Categorize] -> [Payee check] -> [Risk] -> [Insights] -> [Store] -> [UI]
```

| Stage | What happens | Uses LLM? |
|---|---|---|
| Read | Adapter per source turns image, email or text into a raw record plus confidence | Screens: yes (vision). Emails and text: rules first, LLM fallback |
| Validate | Code checks amount, date, required fields, screen type | No |
| Confirm | User sees a table, fixes wrong cells, confirms. Low-confidence cells highlighted | No |
| Normalize | Strip accents, expand abbreviations, parse amounts and dates, clean names | No |
| Dedupe | Key = transaction ID or order number, else date + amount + payee | No |
| Enrich | Merge payee by key (account + bank, phone, name + bank), compute history, newness, amount ratio | No |
| Categorize | User rule, merchant list, keyword rules, LLM (batched, cached) | Only for the remainder |
| Payee check | Name rules score; LLM only in the uncertain band | Only when unsure |
| Risk | First-match rule table over type, newness, size, user label | No, never |
| Insights | Deterministic detectors; LLM or templates word them from evidence | Wording only |
| Store | Fields only, session-scoped | No |

Import is capped (about 30 images, about 300 rows). Images are processed in parallel with progress streamed to the UI.

## 6. Components

### 6.1 Input adapters (Adapter pattern)

One adapter per source, all returning the same `RawTransaction`:

| Adapter | Source | Method |
|---|---|---|
| `MomoBankOutAdapter` | Screenshot, title "CHUYỂN ĐẾN" | Vision LLM to JSON |
| `MomoWalletInAdapter` | Screenshot, title "NHẬN TIỀN TỪ" | Vision LLM to JSON |
| `MomoBillAdapter` | Screenshot, provider and period | Vision LLM to JSON, drops customer code and address |
| `BankEmailAdapter` (Vietcombank) **[Decision]** | Pasted text, PDF, .eml | Regex over labeled fields |
| `PastedTextAdapter` | Free text | Rules, LLM fallback |
| `DemoAdapter` | Built-in fake dataset | Precomputed |

A router identifies screen type first (title marker, cheap check) and rejects unsupported images.

### 6.2 Normalizer and enricher

Pure functions driven by `data/normalization.json`. Enrichment builds the payee profile (first seen, count, total, typical amount range) and assigns `newness` as new, known, or unknown (unknown when there is too little history).

### 6.3 Categorizer (Strategy pattern)

Interface `Categorizer.categorize(batch) -> results`. Implementations: `RulesOnly`, `RulesPlusLLM`. Swapped for the baseline comparison. Precedence: user rule, `merchants.json`, keyword rules, LLM. LLM calls: batch of 20 to 50, temperature 0, JSON schema, category definitions from `categories.json`, few-shots from `few_shots.json` plus the top-k similar user corrections, retry once on schema failure, cache by normalized (payee, note).

### 6.4 Payee classifier (Strategy pattern)

Same interface idea. Rule score from `name_patterns.json` and `merchants.json`. LLM only when the score falls in the uncertain band. Output: `individual | business | uncertain`, confidence, signals, short reason. Cache by name + bank.

### 6.5 Risk engine

Pure Python reading `data/risk_config.json`. First matching rule wins. Output: flag, reason code, signals, whether to ask the user for a label, explanation from `data/reason_templates.json`. Bumps (late-night new payee, same account under two names, seller-chat red flags) raise the flag one level, max `caution`.

### 6.6 Insight engine

Deterministic detectors emit structured objects with evidence transaction IDs: category spend vs trailing average, new payee with large amount, recurrence, duplicate charge within a short window, month-over-month bill change. Wording comes from templates; the LLM may rephrase but a code check verifies every number against the evidence.

### 6.7 Chat agent

LLM plus a fixed read-only toolset (not text-to-SQL): `aggregate(group_by, metric, filters, period)`, `query_transactions(filters, limit)`, `list_payees(filters)`, `get_flagged()`, `compare_periods(a, b)`. Today's date injected and relative periods resolved in code. Responses return data plus text so the UI can render a chart or table. Questions about data that does not exist must return "no data".

### 6.8 Pre-payment check

User enters payee name, bank, amount, purpose, optional seller chat. Runs the same payee classifier and risk engine with `user_label = purchase`. An LLM reads the seller chat for red flags (asks to pay first, rush, price too low, move to another app); those flags only bump severity in code.

### 6.9 LLM adapter

One interface (`complete_json`, `complete_vision_json`), provider implementations behind it, so the model can be swapped. Responsibilities: prompt loading by version, JSON schema validation and one retry, timeout, response cache, per-session and global budget accounting, redaction of anything that should not be logged.

## 7. Data model

```
Session(id, created_at, expires_at)
Transaction(id, session_id, source, tx_ref, timestamp, amount, direction,
            payee_id, note, category, category_confidence, category_source,
            payee_type, payee_confidence, risk_flag, reason_code, signals_json,
            parse_confidence, user_confirmed)
Payee(id, session_id, key_type, key_value, display_name, bank, first_seen,
      count, total_amount, typical_low, typical_high, user_label)
UserRule(id, session_id, payee_id | keyword, forced_category, forced_label)
LLMCache(hash, task, prompt_version, output_json, created_at)
AuditLog(id, session_id, event, meta_json)   -- no raw financial text
```

Everything is session-scoped with a TTL and a delete button. No raw image or email text is stored.

## 8. API sketch

| Method and path | Purpose |
|---|---|
| `POST /session` | Start a session |
| `POST /import` | Upload images or paste text; returns parsed rows for confirmation (progress over SSE) |
| `POST /import/confirm` | Confirm edited rows and run the analysis |
| `POST /demo/load` | Load the demo dataset |
| `GET /transactions` | List with filters |
| `PATCH /transactions/{id}` | Correct category or payee type |
| `POST /payees/{id}/label` | One-tap label; saves a per-user rule |
| `GET /insights` | Insights with evidence |
| `GET /dashboard` | Aggregates for the UI |
| `POST /chat` | Chat turn |
| `POST /precheck` | Pre-payment check |
| `DELETE /session` | Delete all data |

No endpoint moves money. `analyze(transactions)` is a Facade that the confirm and demo routes call, hiding stages Normalize to Insights from the UI.

## 9. Security architecture

| Threat | Control |
|---|---|
| Prompt injection via names, notes, image text | Delimited data blocks; LLM never sets flags; tools read-only; text never chooses tool arguments |
| Fake or edited receipts | Code validation; user confirmation table; sender-domain check for bank emails; app states it reads receipts, never proves them |
| Misread numbers | Show every row, highlight low confidence, dedupe by transaction ID |
| Abuse and cost | Per-session rate limit, global spend cap, image count and size limits, request timeouts |
| Data exposure | Parse then drop raw inputs, session TTL, delete endpoint, no raw text in logs or analytics, secrets in env only |
| Hosting | HTTPS only, CSP headers, no wildcard CORS |

## 10. Reliability and degradation

| Failure | Behavior |
|---|---|
| LLM slow or down | Rules-only path for live input with an "AI unavailable" badge; demo data unaffected (precomputed and cached) |
| Spend cap reached | New LLM calls refused with a clear message; cached and rules paths keep working |
| Bad image or schema failure | One retry, then mark the row as needs review instead of failing the import |
| Host sleeps | Use an always-on instance; keep a health check |
| DB error | Return a clear error, never partial analysis |

## 11. Observability

Structured logs with event names and IDs only (no financial text). Counters: imports, rows parsed, parse failures, LLM calls and cost, cache hit rate, flag rate, one-tap ask rate, errors, p95 latency per stage. A small admin view or log query is enough. These counters also feed the slides.

## 12. Deployment

- One container image, one always-on instance, managed database (SQLite on a volume is acceptable for the contest if the host keeps it) **[Decision]**.
- Environment variables for LLM key, spend cap, rate limits, session TTL. Never commit secrets.
- Deploy a bare version by day 2 or 3, then deploy every merge to main. Record the video on the live deployment, not localhost.

## 13. Design patterns in use

| Pattern | Where |
|---|---|
| Pipeline | Processing stages |
| Layered architecture | Whole system |
| Adapter (GoF) | Input sources, LLM provider |
| Strategy (GoF) | Categorizer and payee classifier versions; baseline comparison |
| Facade (GoF) | `analyze()` entry point |
| Cascade / tiered classifier | Categorize, payee check |
| Classification with abstention | Payee type, newness, parsing |
| Tool-calling agent | Chat |
| Human-in-the-loop | Confirmation table, one-tap label, corrections |
| Read-only guardrail | Whole API |

Keep to these. More patterns would over-engineer a 9-day build.

## 14. Open architecture decisions

1. Backend language and framework (default Python, FastAPI).
2. Database (default SQLite on a persistent volume; Postgres if the host makes it easy).
3. Frontend approach (default a thin single-page app or server-rendered pages).
4. LLM provider and vision model (test 20 real screens on 2 models).
5. Hosting provider (must be always-on).
6. Include the bank email adapter in the contest build or not.
