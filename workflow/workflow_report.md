# Workflow Report: How We Got to the Design

Date: Nov 26, 2026. A short story of the discussion. The detailed steps are in `Workflow_draft.md`.

---

## Where we started

Your teammate is building a feature for MoMo that labels each transaction with a spending category, so people can set budgets without tagging everything by hand. When a payment has no shop information (a friend, a street vendor), the clues are weak, and an AI has to weigh them.

His files described the idea well, but you said they were "not specific enough" to draw a workflow. He also added a new request: show the user's monthly income, a spending table, AI advice on what to cut, and suggested vouchers.

So we went through the open questions one by one.

---

## Question 1: How should the AI work?

**The problem.** His files left three options open: the AI gives one answer, the AI can ask for more data first, or the AI also decides what to do with the payment. The choice changes every step after it.

**My suggestion.** I explained all three in simple terms and asked which one he had chosen.

**Your choice.** Option 2: the AI can ask for more data before it answers.

---

## Question 2: What does the AI see first?

**The problem.** His design already gave the AI everything up front, including the receiver's full payment pattern. If the AI already has all the data, it has nothing to ask for, and "ask for more data" would be only for show.

**My suggestion.** Send a small first package (amount, time, name, message). Keep the receiver's pattern behind a tool the AI calls only when the first clues are weak.

**Your choice.** Yes, the small first package. The tool call is real, and it shows in the side panel.

---

## Question 3: What if the AI is not available?

**The problem.** You added a rule: AI is the core, and the product should not work without it. But his design had a rules-based fallback that would quietly label things without AI, which goes against that rule. Judges running the zip also have no API key.

**My suggestion.** Show recorded AI answers for the four demo scenarios only. For anything else, show "AI required". I gave an example and explained what a recorded answer is: a real AI answer saved ahead of time.

**Your choice.** Yes, recorded answers for the four scenarios, no fallback labels.

---

## Question 4: How does data come in?

**The problem.** His files showed one transaction arriving at a time. But the new spending table and advice need a whole month of data, and nobody will type 40 transactions into a demo.

**My suggestion.** Start the demo user with about a month of fake, already-labeled history. New transactions then arrive one by one through the scenario buttons or form.

**Your choice.** Yes, pre-loaded history.

---

## Question 5: When does the advice run?

**The problem.** The AI writes advice from the spending table. If it re-runs after every transaction, it costs more and the text may flicker while judges click around.

**My suggestion.** Run it once on first load, then only when the user clicks "Get advice".

**Your choice.** Yes, the button.

---

## Question 6: Who picks the vouchers?

**The problem.** Vouchers could be matched by plain code (category matches, voucher shows), but then the AI plays no part, which goes against "AI is the core".

**My suggestion.** The AI picks 2 to 3 from a fixed list and explains why. Code rejects any voucher that is not on the list, so the AI cannot invent offers. The advice and voucher choice can share one AI call, so there is no extra cost. I said the simple code version is a fine fallback if time runs short.

**Your choice.** The AI picks.

---

## Where we ended

Two flows feed one screen:

1. **Label a transaction.** The transaction comes in, gets cleaned, and goes through lookups. If it is still unknown, the AI looks at a small package of clues, and may ask for the receiver's pattern. Then a code guard decides the final action: label silently, label as an editable suggestion, or ask the user later.
2. **Advice and vouchers.** Code builds the category table from labeled transactions and the user's income. The AI writes advice and picks vouchers. Code checks the numbers, the categories, and the voucher IDs before anything is shown.

One idea runs through both: **the AI proposes, and code decides and checks.**

---

## Suggestions that still need your teammate's approval

- Let the guard read crowd numbers from the database itself, since the AI might answer without asking for them.
- Drop `own_label_for_receiver`, which is always empty at the AI step.
- Mark rent, bills and health or education as fixed costs, so the AI never says "spend less on rent".
- Decide the numbers: minimum senders, agreement level, score cutoffs, and the minimum data needed for advice.
