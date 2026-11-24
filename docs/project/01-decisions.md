# Decisions

## Made (working agreements from discussion)

| # | Decision | Why |
|---|---|---|
| 1 | Type is not risk. Risk comes from a combination of signals | Most person transfers are rent, family, meals |
| 2 | Transfer note carries little intent, so it is a weak signal, not a gate | Notes are default or preset, and presets can be written by scammers |
| 3 | Deterministic rule table sets the risk flag. LLM never scores risk | Explainable, testable, immune to injection flipping the flag |
| 4 | LLM reads, classifies ambiguous names, words explanations, runs chat via tools | Where semantics matter |
| 5 | Screenshot upload is the main input | MoMo has no export or API |
| 6 | Payee key order: account number + bank, then phone, then name + bank | Stability of identity |
| 7 | Newness has three states: new, known, unknown | One screenshot has no history |
| 8 | One-tap ask on flagged rows; answer saved as a per-user rule | Real intent only comes from the user |
| 9 | Pre-payment check is P1 | Needs no history, best demo moment |
| 10 | Parse, keep fields, drop the raw image or email text | Privacy |
| 11 | Read-only app, no payment actions | Safety |
| 12 | Gmail connection and email forwarding are out of scope | Google review and mail server cost |
| 13 | Drop CSV as a user input (keep only for our own test data) | Users have no CSV |

## Open

| # | Decision | Default proposed |
|---|---|---|
| 1 | Input scope: MoMo screenshot only, or plus Vietcombank email adapter | MoMo first, then one bank adapter |
| 2 | Floor amount below which nothing is flagged | 200k VND |
| 3 | "Big" threshold | Larger of 2x user median and 2M VND once 5+ transactions; fixed 2M before |
| 4 | Newness: three states plus one-tap ask, or one-tap ask only | Three states plus one-tap ask |
| 5 | UI wording for flags | Vietnamese labels ("nên kiểm tra", "cần cân nhắc") |
| 6 | Vision model | Test 20 real screens on 2 models, then choose |
| 7 | Tech stack and LLM provider | FastAPI, SQLite (or Postgres), thin frontend, LLM adapter |
| 8 | Product name | TBD |
| 9 | Category list and definitions | See glossary draft |
| 10 | Language | Vietnamese-first UI, English only where cheap |
| 11 | Who labels the dataset | Two people on hard cases, one on the rest |
| 12 | Stretch features order | Pre-payment check, then subscription detection |
