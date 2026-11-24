# Data Sources (real shapes seen so far)

All samples below come from real screens and emails provided by the team. Never put real names, phone numbers, account numbers, customer codes or addresses into the demo, video, slides or this repo. Use fake data in the same shape.

## 1. MoMo screenshot: bank transfer out

Title pattern: `CHUYỂN ĐẾN <NAME> (<BANK>)`, amount shown as negative.

Fields: receiver name (uppercase, no accents), bank, amount, time, transaction ID (Mã giao dịch), wallet used (Tài khoản/thẻ), fee, status, category (MoMo may show "Ẩn khỏi báo cáo"), spend toggle, repeat frequency, note field (usually empty).

Missing: receiver account number. Payee key is name + bank.

Notes: the screen can be cut off at the bottom when scrolled. MoMo hides these transfers from its own spending report, which is the gap this product fills.

## 2. MoMo screenshot: money in from a wallet

Title pattern: `NHẬN TIỀN TỪ <NAME>`, amount shown as positive.

Fields: sender name, wallet name, nickname slot (Tên gợi nhớ), phone number, amount, time, transaction ID, category (for example "Trợ cấp"), income toggle.

Payee key: phone number. A set nickname means the user knows this person. Contains a phone number, so treat as sensitive.

## 3. MoMo screenshot: bill payment

Fields: provider (Nhà cung cấp), customer code, customer name, address, billing period (Kỳ thanh toán, for example 11/2026), amount, spend toggle, repeat frequency, note field.

Known biller means business, category Bills and utilities, no LLM needed. Period enables month-over-month change detection. **Drop customer code, name and address at parse time.**

## 4. Bank email: Vietcombank payment receipt ("Biên lai chuyển tiền qua tài khoản")

Format: bank-sent email with bilingual labels, plain text.

Fields: transaction date and time, order number (Số lệnh giao dịch), debit account, remitter name, **credit account (receiver account number)**, beneficiary name, beneficiary bank, amount, fee type and amount, note (Nội dung chuyển tiền, usually "<NAME> chuyen tien").

Best payee key. Parseable with rules and regex, so no vision model needed. One adapter per bank. Check the sender domain when headers are available. Input path: paste text or upload PDF or .eml.

## Cross-source facts

- Notes are default or preset in the large majority of cases. Do not rely on notes for intent.
- Transaction ID or order number is the dedupe key.
- Payee key priority: account number + bank, phone, normalized name + bank. Merge across sources when keys match.
- Screenshot text is untrusted input (prompt injection surface).

## Input paths summary

| Path | Status |
|---|---|
| Demo dataset button | Must have |
| MoMo screenshot upload (5 to 30 images) | Main path |
| Paste text (bank email, SMS) | MVP, small |
| Bank email adapter (Vietcombank) | **[Decision]** |
| Gmail connect, forwarding address | Out of scope |
| CSV | Test data only |
