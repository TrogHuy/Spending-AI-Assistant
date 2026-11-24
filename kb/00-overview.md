# Overview

## Contest (MoMo TechDay 2026, Tech Contest)

- Theme: "Imagine the Future Workspace (Human Centered. AI Powered)". Imagine how AI changes the way fintech teams work, and through that how people shop, pay, manage money and protect themselves online. Turn it into a product that works today.
- Product must belong to at least one topic: **AI Assistant, Cybersecurity, Anti-scam**. Ours covers AI Assistant (financial assistant) and Anti-scam.
- Teams of 2 to 3 students. User scope is unrestricted; the team picks its user and states it in the description.
- Product requirements: concrete, usable online (not a demo or mock-up), judges can try it without depending on the team's local machine, AI is a core component and not a side feature.

### Submission (4 items)

| Item | Requirement |
|---|---|
| Product | 1-page A4 PDF guide with a live link, plus a test account if login is needed |
| Source code | .zip of all source. README with how to run, tech used, and a list of existing assets, open-source libraries and AI tools used |
| Description | Slides, max 10 pages, PDF. States the problem and solution. Used for the final. Cannot be edited after submission |
| Video | Max 3 minutes, MP4. Show the real product running. Anything past 3 minutes is not watched |

### Timeline

| Stage | Dates |
|---|---|
| Registration | Nov 13 to Nov 27 |
| Qualifying round | Nov 20 to Dec 3 |
| Final | Dec 11, THE ADORA PREMIUM, Ho Chi Minh City |

Today is Nov 24, so about 9 days remain before the Dec 3 deadline. Register before Nov 27.

## Product

Working name: **[Decision]**.

An AI assistant that sits between the person and their payments. It reads their transactions, categorizes them, tells them **who** they paid (individual or business) and how much that deserves a second look. It also answers questions about their own spending. It never moves money.

### Problem

- Many Vietnamese payments are bank transfers, often by QR to another bank's account.
- Notes are usually the bank default or preset by the receiver, so budgeting apps get little from them.
- MoMo has no history export or API for us. MoMo also hides bank transfers from its own spending report ("Ẩn khỏi báo cáo").
- Users cannot easily tell whether they paid a business or a personal account, which is a common trust and fraud signal.

### Pitch angle

Lead with **"know who you are paying, before you pay"**. The spending assistant is the engine behind it. The key differentiators versus MoMo's built-in stats are the payee trust check and the pre-payment check. Basic category charts alone will not win.

### Core use cases

| Priority | Use case |
|---|---|
| P0 | Import (demo data, screenshot upload) then categorize |
| P0 | Payee type and risk flag with explanation |
| P0 | Dashboard: category, individual vs business, flagged items |
| P0 | Chat over the user's own data, using tools so numbers are exact |
| P0 | Correction and one-tap label, saved as a per-user rule |
| P1 | Pre-payment check |
| P1 | Recurring and duplicate detection, monthly summary (mostly deterministic) |
| P2 | Bank email adapter (Vietcombank), family or shared view, subscription detection |

### Scope

In scope: read, analyze, warn, chat. Out of scope: real bank or MoMo integration, live account-name lookup, moving money, Gmail connection (needs Google app review, too slow).

## Architecture (summary)

One deployable service, no queue. Web UI, then FastAPI monolith with: ingest (adapter per source, normalizer, dedupe), enrichment (merchant list, payee profile), categorizer (user rule, merchant list, keyword rules, LLM), payee classifier (rules, LLM only when unsure), risk engine (rule table), insight engine (code detects, LLM words it), chat agent (LLM plus read-only tools), storage (SQLite or Postgres) plus an LLM cache table.

Design principles:
- LLM does semantics (reading images, noisy names, wording, chat). Code does arithmetic, flags and insight detection.
- Always-on hosting. A cold start of 40 seconds loses judges.
- Deterministic demo path: precompute and cache the demo dataset so it works even if the LLM provider is down. Live inputs degrade to rules-only with a visible "AI unavailable" badge.
- GoF patterns kept to three: Adapter (one per input source), Strategy (categorizer and payee classifier swappable, used for baseline comparison), Facade (single `analyze()` entry point).

## Build plan (about 9 days)

| Days | Goal |
|---|---|
| 1 to 3 | Screenshot reader, validate-and-confirm table, bare version deployed. This is the riskiest core, so do it first |
| 4 to 5 | Payee classifier, risk rules, eval script |
| 6 to 7 | Insights, dashboard, chat agent with tools |
| 8 | One-tap label and rules, rate limit and spend cap, demo data |
| 9 | Full eval, fix weak spots, 1-page guide PDF, README |
| Last | Slides (max 10), video (max 3 min), zip, submit |

Suggested split for 3 people: (1) parsing, data generation and eval, (2) backend, LLM and agent, (3) frontend, deployment, PDF, slides and video. One person owns eval from day 1.
