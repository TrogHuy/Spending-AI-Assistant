# Context for Claude: Spending Assistant project

Paste this into a Claude project (instructions or knowledge) so new chats start informed. Last updated Nov 24, 2026. Details live in the other KB files.

## Project

Team of students building a product for the MoMo TechDay 2026 Tech Contest. Topics covered: AI Assistant and Anti-scam. Deadlines: register by Nov 27, submit by Dec 3, final Dec 11. Submission: 1-page A4 PDF guide with live link, source .zip with README (including assets, libraries and AI tools used), slides PDF (max 10), MP4 video (max 3 min). The product must be live online, judges must be able to try it, and AI must be core.

## Product

An AI spending assistant that reads transactions, categorizes them, classifies each payee as individual, business or uncertain, flags payments worth a second look, answers questions about the user's own spending through read-only tools, and offers a pre-payment check. Read-only, never moves money. Pitch: "know who you are paying, before you pay".

## Key facts and decisions

- MoMo has no export or API, so **screenshot upload is the main input**. Also a demo-data button and paste text. A Vietcombank email adapter is under consideration. CSV is not a user input.
- Real samples: MoMo bank-out screen (receiver name and bank, no account number), wallet-in screen (sender name and phone), bill screen (provider, period), Vietcombank email (includes receiver account number). Notes are almost always default or preset, so they carry little intent.
- Payee key priority: account + bank, phone, name + bank.
- Type is not risk. A deterministic rule table sets the flag (none, review, caution); the LLM never scores risk. Intent comes from the user (pre-payment check or one-tap label saved as a per-user rule).
- Newness has three states: new, known, unknown.
- LLM does semantics (reading, ambiguous names, wording, chat). Code does arithmetic, detection, flags.
- Security: all text from images, names and notes is untrusted data (prompt injection). Tools are read-only. Parse then drop raw images and email text. Fake data only in demo, video and slides.
- Eval: 200+ fake transactions in real shapes, dev/test split with test locked, rules-only vs rules + LLM comparison, chat test with known answers, adversarial tests.
- Architecture: one FastAPI monolith, SQLite or Postgres, thin frontend, always-on hosting, cached demo path, Adapter, Strategy and Facade patterns only.

## Open decisions

Input scope (bank email adapter or not), floor amount (default 200k), "big" threshold (default larger of 2x median and 2M), newness handling, UI flag wording, vision model, stack and LLM provider, product name, category list, language, who labels data. See `01-decisions.md`.

## How to help

- Be concrete and brief. The team has about 9 days.
- Respect the principles above unless the team changes them in `01-decisions.md`.
- Flag any suggestion that would put real personal data into demos or slides.
- Mark anything undecided as **[Decision]**.
