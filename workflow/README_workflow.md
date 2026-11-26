# How to Read the Two Workflow Files

Status: draft, Nov 26, 2026. A short guide for teammates.

## Read in this order (about 15 minutes)

1. **`workflow_report.md` first (3 min).** A short story of how we reached the design, with the problem, the suggestion and the choice for each topic. After it you will know *why* the system looks the way it does.
2. **`Workflow_draft.md` second (10-12 min).** The detailed steps. You need this one to build or draw anything.

## Inside `Workflow_draft.md`

| Section | What it is | Read it if you... |
|---|---|---|
| 0. Decisions | The 8 decisions everything is built on | want the rules in one table |
| 1. Product on one screen | The screen layout and the 2 flows | want the big picture |
| 2. Flow 1 | Labeling one transaction, from JSON in to final label | work on backend, the AI step or the pipeline |
| 3. Flow 2 | Spend table, advice and vouchers | work on the new income, advice or voucher feature |
| 4. Data stored | The tables and what each is used for | design the database |
| 5. Run modes | `live` vs `replay` | prepare the zip and the demo |
| 6. Open items | Questions still waiting for a decision | need to know what is not final |

## Tips for reading a flow

- Start with the **diagram**, then read the **step table** below it. The step numbers match.
- The **"Done by"** column shows what is code and what is AI. Only a few steps are AI, and code checks every AI answer.
- The **example JSON** blocks show the exact shape of the data at that step. All numbers are made up.
- **[Confirm]** marks a suggestion that has not been approved yet. Treat it as a proposal, not a rule.

## Seeing the diagrams

The diagrams are written in Mermaid. They render automatically on GitHub, in VS Code with a Markdown preview, or by pasting the code into mermaid.live.

## The one idea to remember

**The AI proposes, and code decides and checks.**
