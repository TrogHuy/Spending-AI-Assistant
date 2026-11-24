# Evaluation Plan

## Questions to answer

1. Does the reader get fields right?
2. Do category and payee labels hold up?
3. Does the risk flag catch risky cases and spare safe ones?
4. Does the LLM add value over rules?

## Dataset

- Generate fake data in the real shapes: MoMo screens, bank email, bill screen. Render from an HTML template to images. Vary name, amount, date, bank, dark mode, crop.
- Three personas (student, office worker, small seller), 2 to 3 months each.
- Target 200+ transactions, about 100 as images.
- 70 to 80 percent of notes are default or preset, to match real life.
- Hand label each transaction: category, payee type, true risk (safe, review, caution).
- Two people label; report agreement. Disagreements become hard cases. **[Decision]** two people on hard cases only, one on the rest.
- Split 60 percent dev and 40 percent test. Lock test. Do not tune on it.

### Hard case set (about 30)

- Shop under a personal name with a preset note
- Name without accents, abbreviations
- Rent vs purchase that look the same
- Late-night new payee
- Same account under two names
- Cut-off, blurred, dark-mode and cropped images
- Prompt injection text in a note and inside an image
- Junk image (not a MoMo screen)

## Metrics

| Stage | Metric | Target |
|---|---|---|
| Reader | Field accuracy, especially amount and date | Amount near 100% |
| Reader | Full-row success rate | 90%+ |
| Category | Accuracy overall and per category | Report honestly |
| Payee type | Precision, recall, uncertain rate | Report all three |
| Risk | Flag rate | About 5 to 10% |
| Risk | Miss rate on truly risky cases | As low as possible |
| Risk | False-alarm rate on safe cases | As low as possible |
| Risk | One-tap ask rate | Lower is better |
| Chat | 20 to 30 questions with known answers, plus 5 "no data" traps | Numbers exact |
| Security | Injection cases blocked | 100% |

## Baseline comparison (key slide)

Run rules-only vs rules + LLM on the same test set by swapping the Strategy. Also report how often the LLM was called, cost per 100 transactions, and time per import. If the LLM only wins on hard cases, say so; that is the point of using it.

## Chat test

25 questions with known answers from the dataset: sums, filters, period comparisons, payee lists, "summarize my month". Score: exact numbers, right tool, no invented facts. Include 5 traps about data that does not exist; the app must say it has no data.

## Adversarial tests

- Injection in note, name and image text: flag not flipped, tools not misused.
- Edited or fake receipt: the app reads it and never claims it is genuine.
- Oversized or malformed input, 100 images: limits and clean errors.

## Regression loop

- One command runs the eval and prints a table. Run it after every prompt or rule change.
- Log scores per prompt version.
- User corrections in the app go to a log for review as new cases.

## Pitfalls

- Tuning on test inflates the score.
- Data that is too clean makes judges doubt it. Keep messy cases.
- Reporting only good numbers loses trust. Show weak spots and the fix plan. State sample sizes.

## Slide order

1. Dataset: size, sources, hard-case count
2. Reader accuracy
3. Rules vs rules + LLM
4. Risk: flag rate, miss rate, false alarms
5. Chat accuracy and injection test
6. Known limits

## Timeline

Day 1 schema and templates, day 2 generate about 100 images, day 3 label, day 5 eval script, day 9 full run. One teammate owns eval from day 1.
