# Spending Assistant: Risk Rules Spec

Status: **DRAFT**. Items marked **[Decision]** are open for the team. Defaults are my proposal and will be updated once the team finalizes.

Scope: how the app decides whether a payment deserves a warning. The app only reads and analyzes. It never moves money.

---

## 1. Core principles

1. **Type is not risk.** Paying an individual is normal (rent, family, splitting a meal). Risk comes from a combination of signals, not from "individual" alone.
2. **The transfer note carries almost no intent.** Most notes are the bank default ("NAME chuyen khoan") or are preset by the receiver. Preset notes can be written by a scammer too, so they are never proof of safety. The note is a weak bonus signal, not a gate.
3. **Deterministic code decides the flag.** An explicit rule table produces the flag. The LLM only helps with ambiguous payee names and with wording the explanation. It never assigns a risk score.
4. **Advisory tone.** Warnings say "worth double-checking", never "this is fraud". Flags are signals, not verdicts.
5. **Abstain when unsure.** Ambiguous payee type returns `uncertain`. Missing history returns `unknown`, not `new`.

---

## 2. What data we actually have

Real samples show the app will mostly work from screenshots and bank emails, because MoMo has no history export and no API for us.

| Source | Fields available | Notes |
|---|---|---|
| MoMo screenshot: bank transfer out | Receiver name (caps, no accents), bank, amount, time, transaction ID, fee | No account number. Note usually empty. MoMo hides these from its own report ("Ẩn khỏi báo cáo"). |
| MoMo screenshot: money in from wallet | Sender name, phone, amount, time, transaction ID, nickname slot | Phone is a stable ID. A set nickname means the user knows the person. |
| MoMo screenshot: bill payment | Provider, period, amount | Known biller means business. Contains customer code and address (drop them). |
| Bank email (e.g. Vietcombank receipt) | Time, order number, sender, receiver name, **receiver account number**, receiver bank, amount, fee, note | Plain text, so rules and regex can parse it. No vision model needed. |

### Payee key (best to worst)

1. Account number + bank (bank email)
2. Phone number (MoMo wallet)
3. Normalized name + bank (MoMo bank-out screenshot)

Merge payees across sources when keys match.

### Signals we use

| Signal | Source | Strength |
|---|---|---|
| Payee name pattern (business keywords vs a 2 to 4 token personal name) | All | Strong |
| Known merchant or biller list | All | Strong |
| Payee history (repeat count) | Needs several uploads | Strong when available |
| Amount vs the user's own range | Needs several uploads | Medium |
| Nickname set on a wallet contact | MoMo wallet screen | Medium |
| Time of day (late night + new + big) | All | Weak |
| Preset note with order or invoice code | Notes | Weak, can be faked |
| QR payload merchant fields | Only if raw QR is available | Stretch, verify on real samples |
| **User-supplied intent** (pre-payment check or one-tap label) | User | Strongest |

---

## 3. Inputs to the rule table

| Input | Values | How it is derived |
|---|---|---|
| `direction` | in / out | Screen title or email type |
| `payee_type` | individual / business / uncertain (+ confidence) | Name rules first, LLM only in the uncertain band |
| `newness` | new / known / unknown | Payee history. `unknown` when we have too little history to say. |
| `size` | big / normal / small | Amount vs floor and vs user norm |
| `user_label` | purchase / rent / family / friend / trusted / none | One-tap answer or per-user rule |

**[Decision] Floor amount:** 200k VND or 500k VND? (Default: 200k)
**[Decision] "Big" threshold:** 2x the user's median for individual payees, or a fixed amount such as 2M VND? (Default: whichever is larger of 2x median and 2M, once there are 5+ transactions. Fixed 2M before that.)
**[Decision] Newness handling:** use the three-state version above, or rely only on the one-tap ask? (Default: three states plus one-tap ask on flagged rows.)

---

## 4. Rule table (first match wins)

| # | If | Flag | Reason code |
|---|---|---|---|
| 1 | Direction is in | none | `INCOMING` |
| 2 | User label is rent / family / friend / trusted | none | `USER_LABELED_SAFE` |
| 3 | Amount below floor | none | `SMALL_AMOUNT` |
| 4 | Business and known merchant or biller | none | `KNOWN_MERCHANT` |
| 5 | Individual and known and repeat payee | none | `RECURRING` |
| 6 | Individual, new or unknown, big, **user label is purchase** | caution | `NEW_PERSON_BIG_PURCHASE` |
| 7 | Individual, new or unknown, big, no label | review, trigger one-tap ask | `NEW_PERSON_BIG_UNLABELED` |
| 8 | Uncertain type, new or unknown, big | review | `UNSURE_NEW_BIG` |
| 9 | Otherwise | none | `DEFAULT` |

Design notes:

- Safe rules come first to cut noise early.
- Only rule 6 uses the strongest warning, and it needs user-supplied intent. Because the note is usually empty, the app does **not** guess intent. It says "new person, large amount, worth checking" and asks one question.
- Target: roughly 5 to 10 percent of transactions flagged. If the rate is higher, users will learn to ignore the app. Tune thresholds against the eval set.
- Bumps (each raises the flag one level, max `caution`): late-night time for a new payee, same account number seen under two different names, seller-chat red flags in the pre-payment check (see section 6).

**[Decision] UI wording:** show the internal names `review` / `caution`, or softer Vietnamese labels such as "nên kiểm tra" and "cần cân nhắc"? (Default: Vietnamese labels.)

---

## 5. Output per transaction

```json
{
  "risk_flag": "none | review | caution",
  "reason_code": "NEW_PERSON_BIG_UNLABELED",
  "signals": {
    "payee_type": "individual",
    "payee_confidence": 0.82,
    "newness": "unknown",
    "amount_ratio": 3.1,
    "late_night": true
  },
  "needs_user_label": true,
  "explanation": "Người nhận mới, số tiền lớn hơn thường lệ. Nên kiểm tra lại trước khi tin tưởng."
}
```

- The explanation comes from a template keyed by `reason_code`. The LLM may polish the wording but may not add facts.
- Every flag shows the exact rule that fired, so the user and the judges can see why.

---

## 6. One-tap ask and per-user rules

- On rows that trigger rule 7, ask: "Cho việc gì?" with options: mua hàng / tiền nhà / gia đình / bạn bè / khác.
- The answer is saved as a per-user rule for that payee key. The same payee is not asked again.
- Corrections also go into the evaluation log.

### Pre-payment check (P1)

The user enters payee name, bank, amount, and what they are buying (optionally pasting the seller's chat). This is the **only place real intent exists**, and it needs no history.

- Reuses the same classifier and the same rule table, with `user_label = purchase`.
- Seller-chat signals that bump the flag: asks to transfer first, rushes the buyer, price far below market, asks to move to another app, refuses a meeting or COD.

---

## 7. Known limits (say these in the slides)

- One screenshot has no history, so "new payee" is often `unknown`. Mitigation: ask users to upload 1 to 3 months, and use the one-tap ask.
- Many person-to-person transfers stay "Transfers to people, unlabeled" until the user labels them. The category chart will show a large unlabeled slice. This is honest, not a bug.
- A shop operating under a personal name with a preset note is the hardest case. The app flags it as `review`, not as fraud.
- The app reads a receipt. It cannot prove the receipt is genuine.

---

## 8. Security and abuse

- **Prompt injection.** Notes, names, and text inside images are untrusted. Keep them in a delimited data field, keep every tool read-only, and never let that text choose tool arguments.
- **Fake or edited screenshots and emails.** Validate fields in code, check the sender domain for bank emails when headers exist, and show the parsed table for user confirmation before analysis.
- **Misread numbers.** Show every parsed row for confirmation. Highlight low-confidence cells. Use the transaction ID or order number as the dedupe key.
- **Rate limits.** Per-session limit, global spend cap, cap of about 30 images per import, input size limits, and graceful errors.

## 9. Privacy

- Parse, keep only needed fields, and drop the raw image or email text after parsing.
- Drop customer codes and addresses from bill screens. Do not store phone numbers beyond the payee key.
- Session-scoped data with a delete button and a TTL. No raw financial data in logs or third-party analytics.
- All demo data, video, and slides use fake data. **Never** show real names, phone numbers, or account numbers.

---

## 10. Evaluation cases per rule

Generate fake screenshots and emails in the real layouts. Make 70 to 80 percent of notes default or preset to match real life. Split into dev and test, and do not tune on test.

| Rule | Test cases |
|---|---|
| 1 `INCOMING` | Wallet receipt, bank credit. Must never flag. |
| 2 `USER_LABELED_SAFE` | Rent to a new landlord after the user taps "tiền nhà". Next month must not flag. |
| 3 `SMALL_AMOUNT` | 25k transfer to a new person. No flag. |
| 4 `KNOWN_MERCHANT` | Electricity, water, Grab, Shopee. No flag. |
| 5 `RECURRING` | Same person monthly (rent, family). No flag. |
| 6 `NEW_PERSON_BIG_PURCHASE` | Pre-payment check: 3M to a new individual to buy a phone. `caution`. |
| 7 `NEW_PERSON_BIG_UNLABELED` | 2.5M to a new person, no label. `review` plus one-tap ask. |
| 8 `UNSURE_NEW_BIG` | Ambiguous name (for example a brand-like personal name). `review`. |
| Hard | Shop on a personal name with a preset note, name with no accents, late-night transfer, scroll-cut screenshot, dark-mode screenshot, injection text in a note. |

Metrics: flag rate, miss rate on hand-labeled risky cases, false-alarm rate on safe cases, how often the one-tap ask fires, and field-level parse accuracy (amount and date near 100 percent).

---

## 11. Open decisions summary

1. **Input scope:** MoMo screenshot only, or plus the Vietcombank email adapter? (My pick: MoMo first, then one bank adapter.)
2. Floor amount.
3. "Big" threshold.
4. Newness handling.
5. UI wording.
6. Vision model choice (test 20 real screens on 2 models before choosing).
