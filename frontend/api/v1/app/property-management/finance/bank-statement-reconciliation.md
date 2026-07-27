# Bank Statement Reconciliation API

Reconciling a bank statement is a four-step flow:

1. **Extract** — upload the statement; Claude reads it into structured lines and
   the account number is checked against the target account. The extraction is
   cached and a `statement_token` is returned.
2. **Match** — send the token back with the sides you want to reconcile. The
   server matches the statement against our cashbook, auto-records confident
   matches, and returns everything bucketed (matched / possible matches /
   unrecorded / outstanding).
3. **Resolve** — clear an `unrecorded` bank line:
   - if it corresponds to a pending receipt / payment voucher, approve the
     receipt or release the voucher (posts it to the cashbook and matches it); or
   - add it straight to the cashbook as a reconciled line — available for **any**
     unrecorded line, typically a bank-side entry with nothing behind it (bank
     charge, deposit interest) but also a genuine receipt/payment captured only at
     the bank; or
   - **classify** it into the reconciliation taxonomy (statement classes C1–C10 /
     D1–D14). Each item already carries a **`default_classification`** the server
     inferred; the UI pre-selects it and lets the reconciler change it. The class's
     *resolution type* (timing / omission / error / bank issue / suspense) decides
     where it lands in the report.
4. **Report** — once the matches are worked through, generate the bank
   reconciliation statement. `full_account` proves the **adjusted cashbook balance**
   two ways — from our books (± omissions & errors) and from the bank (± timing,
   bank issues, suspense) — and they must agree.

The classification vocabulary is fetched once from the
[classifications endpoint](#classification-taxonomy) so the classify pickers stay
in sync with the engine and show each class's description.

All endpoints are under:
`/api/v1/app/{company}/property-management/settings/finance/banks/accounts/{account}`

Money mapping (fixed): statement **money_in ↔ our debit**, statement
**money_out ↔ our credit**. A line is never both. `is_transaction_fee` and
`is_interest_earning` flag bank-side charges and interest.

---

## 1. Extract a statement

`POST .../{account}/reconciliations/extract`

Send the statement as `multipart/form-data`. The file is processed in-memory and
is **not** persisted; only the structured result is cached (~1 hour) under the
returned token.

Supports **PDF** (read natively) and **spreadsheets** (`xlsx`, `xls`, `csv`,
`ods`, converted to text first).

Request body (`multipart/form-data`):
- `file` (file, required) — `pdf`, `xlsx`, `xls`, `csv`, `ods`. Max 10 MB.

Response:
```json
{
  "data": {
    "message": "Bank statement extracted successfully",
    "statement_token": "9f1c2b6e-1a2b-4c3d-9e8f-0a1b2c3d4e5f",
    "reconciliation": {
      "matches_account": true,
      "expected_account_number": "0123456789",
      "statement_account_number": "0123456789"
    },
    "result": {
      "account_summary": {
        "account_number": "0123456789",
        "account_name": "JOHN DOE",
        "statement_period": { "from": "2025-05-01", "to": "2025-05-31" },
        "opening_balance": 209412.10,
        "closing_balance": 56849.85
      },
      "transactions": [
        {
          "transaction_reference": "CHQ25145TT22",
          "value_date": "2025-05-10",
          "payment_date": "2025-05-08",
          "description": "Cheque deposit ref 004512",
          "money_in": 25000.00,
          "money_out": null,
          "is_transaction_fee": false,
          "is_interest_earning": false
        }
      ],
      "is_legible": true,
      "feedback": null
    }
  }
}
```

Field reference (`result`):
- `account_summary.account_number` / `account_name` — as printed on the statement.
- `account_summary.statement_period.from` / `.to` — ISO-8601 `YYYY-MM-DD`.
- `account_summary.opening_balance` / `closing_balance` — the brought-forward and
  carried-forward balances per the bank (`null` if not printed); the closing
  balance feeds the **report** step. 
- `transactions[]`:
  - `transaction_reference` — statement reference / cheque number.
  - `value_date` — date the money reflected in the account.
  - `payment_date` — date the payment was made (posting date).
  - `description` — transaction notes / details.
  - `money_in` — credit amount (`null` for a debit row).
  - `money_out` — debit amount (`null` for a credit row).
  - `is_transaction_fee` — `true` for bank-levied charges (ledger fees, commission, excise duty).
  - `is_interest_earning` — `true` for interest / deposit interest credited by the bank.
- `is_legible` — `false` when the document could not be read; see `feedback`.
- `feedback` — user-facing message; `null` when legible.

Top-level:
- `statement_token` — pass this to the **match** step (valid ~1 hour, bound to this account).
- `reconciliation.matches_account` — `true` when the statement's account number
  matches the target account (compared ignoring spaces and case).
- `reconciliation.expected_account_number` / `statement_account_number` — the two numbers compared.

### 1a. Async extraction (start + poll)

Large statements can exceed the proxy's ~100s request timeout when extracted
synchronously (the call above). To avoid that, extract in the background: start a
job, then poll for the result. The final `done.data` payload is **identical** to
the synchronous response above, so the follow-up **match** / **report** flow is
unchanged.

**Start** — `POST .../{account}/reconciliations/extract/start`

Same `multipart/form-data` body and validation as the synchronous extract
(`file`: `pdf`, `xlsx`, `xls`, `csv`, `ods`, max 10 MB). The upload is written to
temporary storage for the worker and deleted once the job finishes.

`202 Accepted`:
```json
{ "job_id": "3b2a1c0d-9e8f-4a5b-8c7d-6e5f4a3b2c1d" }
```

**Poll** — `GET .../{account}/reconciliations/extract/jobs/{jobId}`

`200 OK` with the current status envelope:
```json
{ "status": "pending" }
```
- `status` — `pending` | `processing` | `done` | `failed`.
- `data` — present only when `status === "done"`; identical to the synchronous
  extract response body (`message`, `statement_token`, `reconciliation`, `result`).
- `error` — present only when `status === "failed"`.

The job status is scoped to the account, so a job id polled against a different
account returns `404 Not Found`. The status entry lives ~1 hour (≥ the
`statement_token` TTL), so a `done` job stays pollable while its token is valid.
Recommended client cadence: poll every ~3s, give up after ~8 minutes.

### Not-legible response
`200 OK` with empty data and an explanation:
```json
{
  "data": {
    "message": "Bank statement extracted successfully",
    "statement_token": "…",
    "reconciliation": { "matches_account": false, "expected_account_number": "0123456789", "statement_account_number": null },
    "result": { "account_summary": null, "transactions": [], "is_legible": false, "feedback": "The uploaded file is too blurry to read. Please upload a clearer PDF or the original spreadsheet." }
  }
}
```

---

## 2. Match against the cashbook

`POST .../{account}/reconciliations/match`

Request body (`application/json`):
- `statement_token` (string, required) — the token from the extract step.
- `match` (string[], required) — which sides to reconcile. Any of:
  - `money_in` — ordinary credits (vs our debit).
  - `money_out` — ordinary debits (vs our credit).
  - `transaction_fees` — bank charges (only matched when included).
  - `interest_earnings` — bank interest (only matched when included).

Only the requested sides are processed; fees and interest are **ignored unless
listed**. Zero-amount lines are skipped.

```json
{ "statement_token": "9f1c2b6e-…", "match": ["money_in", "money_out"] }
```

### What happens
For each requested statement line the server looks for a cashbook row where **all
three** hold:
1. **Reference** — statement `transaction_reference` equals our `refference_number`
   (EFT/RTGS), or our reference appears inside the statement `description`
   (cheque / M-Pesa).
2. **Date** — our `transaction_at` equals the statement `payment_date` or `value_date`.
3. **Amount** — statement `money_in`/`money_out` equals our `debit`/`credit`.

Exact matches are **recorded immediately** (the cashbook row's
`reconciliation_status` becomes `matched`, with `reconciled_at` and the statement
line stored on `bank_entry`). The leftovers are then sent to AI, which proposes:
- **possible matches** against other unmatched cashbook rows, and
- candidates against **pending receipts** (money in) / **pending vouchers**
  (money out) that have not been posted to the cashbook yet.

AI suggestions are **not** auto-recorded — they need user action.

### Response
```json
{
  "data": {
    "message": "Bank statement reconciled successfully",
    "result": {
      "matched": {
        "money_ins": [
          {
            "cashbook": { "id": 81, "refference_number": "CHQ25145TT22", "transaction_at": "2025-05-08", "debit": "25000.00000", "credit": "0.00000", "notes": "Receipt#12 - …", "payment_method": "Cheque", "transaction": { "type": "FacilityReceipt", "id": 12 } },
            "statement": { "transaction_reference": "CHQ25145TT22", "value_date": "2025-05-10", "payment_date": "2025-05-08", "description": "Cheque deposit ref 004512", "money_in": 25000.0, "money_out": null, "is_transaction_fee": false, "is_interest_earning": false }
          }
        ],
        "money_outs": []
      },
      "possible_matches": {
        "money_ins": [
          {
            "cashbook": { "id": 90, "refference_number": "RTG889", "transaction_at": "2025-05-12", "debit": "10000.00000", "credit": "0.00000", "notes": "Receipt#15 - Jane Doe", "payment_method": "Bank Transfer", "transaction": { "type": "FacilityReceipt", "id": 15 } },
            "candidates": [
              { "statement": { "transaction_reference": "RTG0889", "payment_date": "2025-05-12", "description": "RTGS Jane D", "money_in": 10000.0, "money_out": null }, "confidence": 0.72, "reason": "Same amount and date; reference differs by a leading zero", "color": "warning" }
            ]
          }
        ],
        "money_outs": []
      },
      "unrecorded": {
        "money_ins": [
          {
            "statement": { "transaction_reference": null, "value_date": "2025-05-20", "payment_date": "2025-05-20", "description": "MPESA QGH7… ACME LTD", "money_in": 4500.0, "money_out": null, "is_transaction_fee": false, "is_interest_earning": false },
            "candidates": [
              { "source": { "id": 33, "type": "receipt", "reference": "RCT-0033", "amount": "4500.00000", "date": "2025-05-20", "name": "Acme Ltd", "facility": "Westlands Plaza", "payment_method": "M-Pesa" }, "action": "approve_receipt", "confidence": 0.86, "reason": "Amount and payer name match a pending receipt", "color": "success" }
            ],
            "add_to_cashbook": true,
            "is_bank_side": false,
            "can_classify": true,
            "default_classification": "unrecognised_credit",
            "default_classification_info": { "value": "unrecognised_credit", "code": "C10", "label": "Unrecognised Credits on Bank Statement", "description": "Default holding class: a credit appears on the statement, is absent from the cashbook, and has not yet been investigated.", "side": "credit", "resolution_type": "suspense", "landing_place": "suspense_queue" }
          },
          {
            "statement": { "transaction_reference": null, "value_date": "2025-05-31", "payment_date": "2025-05-31", "description": "Interest credited", "money_in": 120.5, "money_out": null, "is_transaction_fee": false, "is_interest_earning": true },
            "candidates": [],
            "add_to_cashbook": true,
            "is_bank_side": true,
            "can_classify": true,
            "default_classification": "interest_received",
            "default_classification_info": { "value": "interest_received", "code": "C2", "label": "Interest Received", "description": "Credit interest earned on the account balance.", "side": "credit", "resolution_type": "omission", "landing_place": "adjusted_cashbook" }
          }
        ],
        "money_outs": [
          {
            "statement": { "transaction_reference": null, "value_date": "2025-05-31", "payment_date": "2025-05-31", "description": "Ledger fees", "money_in": null, "money_out": 350.0, "is_transaction_fee": true, "is_interest_earning": false },
            "candidates": [],
            "add_to_cashbook": true,
            "is_bank_side": true,
            "can_classify": true,
            "default_classification": "bank_charge",
            "default_classification_info": { "value": "bank_charge", "code": "D1", "label": "Bank Charges & Ledger Fees", "description": "Account maintenance, transaction, RTGS, cheque book and service fees levied directly by the bank.", "side": "debit", "resolution_type": "omission", "landing_place": "adjusted_cashbook" }
          }
        ]
      },
      "outstanding": {
        "money_ins": [
          { "id": 77, "refference_number": "DEP100", "transaction_at": "2025-05-02", "debit": "8000.00000", "credit": "0.00000", "notes": "Receipt#9 - …", "payment_method": "Cash", "transaction": { "type": "FacilityReceipt", "id": 9 }, "default_classification": "uncredited_receipts", "default_classification_info": { "value": "uncredited_receipts", "code": "R1", "label": "Uncredited Receipts (Deposits in Transit)", "description": "Cash, cheques or transfers banked at or near period end that the bank had not credited by the statement date. Fully valid - just late.", "side": "receipt", "resolution_type": "timing", "landing_place": "reconciliation_statement" } }
        ],
        "money_outs": []
      },
      "summary": { "matched": 1, "possible_matches": 1, "unrecorded": 3, "outstanding": 1, "degraded": false }
    }
  }
}
```

### Buckets (each grouped by side: `money_ins` / `money_outs`)
- **`matched`** — already recorded. `{ cashbook, statement }`.
- **`possible_matches`** — parent is the **cashbook** row; `candidates[]` are the
  AI-suggested statement lines, each with `confidence` (0–1), `reason`, and
  `color` (`success` ≥ 0.8, else `warning`). The user confirms one via
  [Confirm a possible match](#confirm-a-possible-match) to record it.
- **`unrecorded`** — parent is the **statement** line; `candidates[]` are pending
  receipts / vouchers it might be. Each candidate has a `source`, an `action`
  (`approve_receipt` or `release_voucher`), `confidence`, `reason`, `color`. A
  statement line with no candidate still appears here with `candidates: []`
  (truly not in our books). Each line also carries:
    - `add_to_cashbook` — always `true`: **any** unrecorded line can be booked
      straight into the cashbook (see step 3.3), not just bank-side charges/interest.
      Use this for a genuine receipt/payment that was only ever captured at the bank.
    - `is_bank_side` — `true` for charges / interest (`is_transaction_fee` or
      `is_interest_earning`); a UI hint that booking to the cashbook is the *typical*
      resolution for this line.
    - `can_classify` — always `true`: any line may be classified into the statement
      taxonomy (C1–C10 / D1–D14).
    - `default_classification` — the statement class the server inferred for this
      line (e.g. a fee → `bank_charge`, interest → `interest_received`, otherwise
      `unrecognised_credit` / `unrecognised_debit`). **Pre-select it** in the
      classify picker; the reconciler can change it.
    - `default_classification_info` — the full class payload for that default
      (`value`, `code`, `label`, `description`, `side`, `resolution_type`,
      `landing_place`) so the picker can render it without a separate lookup.
- **`outstanding`** — cashbook rows with no statement counterpart (in our books,
  not yet on the statement); no children. Each also carries `default_classification`
  + `default_classification_info` (a receipt banked before the period start defaults
  to `receipts_from_prior_periods` (R11), otherwise `uncredited_receipts` (R1); a
  payment defaults to `unpresented_payments` (P1)).
- **`classified`** — lines the reconciler resolved during the workings by
  classification (see [Classify lines](#5-classify-lines-during-the-workings)),
  grouped `cashbook` / `statement` (**not** by side). Classified items are dropped
  from `outstanding` / `unrecorded`. `classification` is the class **value**;
  `classification_info` is the full class payload. Shape:
  ```json
  "classified": {
    "cashbook":  [ { "cashbook": { "id": 77, "debit": "8000.00000", "credit": "0.00000", "notes": "Receipt#9 …" }, "classification": "erroneous_receipt", "classification_info": { "value": "erroneous_receipt", "code": "R4", "label": "Erroneous Receipts", "description": "A receipt posted that never happened at all - keyed in error, wrong customer, or against the wrong instrument.", "side": "receipt", "resolution_type": "error", "landing_place": "adjusted_cashbook" }, "notes": "Double receipted" } ],
    "statement": [ { "statement": { "value_date": "2025-05-15", "description": "Rent - Westlands Plaza", "money_in": 60000.0, "money_out": null }, "classification": "external_transaction", "classification_info": { "value": "external_transaction", "code": "C8", "label": "External Transactions - Credits", "description": "Third-party funds landing in the account with no relationship to our operations (wrong beneficiary, misdirected transfer).", "side": "credit", "resolution_type": "bank_issue", "landing_place": "reconciliation_statement" }, "notes": null } ]
  }
  ```

`summary` carries the total count per bucket across both sides (including
`classified`), plus
`degraded` — `true` when the AI suggestion step was unavailable (e.g. the
provider was overloaded). Deterministic matches are still complete and recorded;
only the fuzzy `possible_matches` were skipped, so leftovers fall into
`unrecorded` / `outstanding`. Offer the user a re-run of **match** to populate
suggestions.

### Confirm a possible match
`POST .../{account}/reconciliations/{cashbookEntry}/match`

When the user accepts one of a cashbook row's `candidates` (from `possible_matches`),
mark that cashbook row reconciled against the statement line they picked.
`{cashbookEntry}` is the `cashbook.id` from the bucket.

Request body (`application/json`):
- `bank_entry` (object, required) — the selected `statement` line (the candidate's `statement`).

```json
{ "bank_entry": { "transaction_reference": "RTG0889", "payment_date": "2025-05-12", "description": "RTGS Jane D", "money_in": 10000.0, "money_out": null } }
```

Response:
```json
{ "data": { "message": "Cashbook entry matched successfully", "result": { "bank_account_transaction_id": 90, "reconciliation_status": "matched" } } }
```

The cashbook row's `reconciliation_status` becomes `matched`, with `reconciled_at`
set and the chosen statement line stored on `bank_entry`. A `404` is returned if
the cashbook entry does not belong to this account.

---

## 3. Resolve an unrecorded line

When the user accepts an `unrecorded` candidate, call the action it carries. Both
post the pending record to the cashbook and immediately match it to the statement
line. Pass the `statement` object from the bucket as `bank_entry`.

### Approve a receipt (money in)
`POST .../{account}/reconciliations/receipts/{receipt}/approve`

Request body (`application/json`):
- `bank_entry` (object, required) — the statement line to attach (the `statement` from the bucket).

```json
{ "bank_entry": { "transaction_reference": null, "payment_date": "2025-05-20", "description": "MPESA QGH7… ACME LTD", "money_in": 4500.0, "money_out": null } }
```

Response:
```json
{ "data": { "message": "Receipt approved and matched successfully", "result": { "receipt_id": 33, "bank_account_transaction_id": 142 } } }
```

### Release a voucher (money out)
`POST .../{account}/reconciliations/vouchers/{voucher}/release`

Request body — same shape (`bank_entry` required).

Response:
```json
{ "data": { "message": "Payment voucher released and matched successfully", "result": { "voucher_id": 21, "bank_account_transaction_id": 143 } } }
```

Both are idempotent on the posting side: a receipt/voucher that has already posted
its cashbook line is not double-posted; the existing line is simply (re)matched.

### 3.3 Add to cashbook
`POST .../{account}/reconciliations/cashbook-entries`

For any `unrecorded` line (all now carry `add_to_cashbook: true`). Typically a
bank-side entry with nothing behind it — bank charge or deposit interest, flagged
`is_bank_side: true` — but equally a genuine receipt/payment that only ever landed
at the bank. This creates a cashbook line directly from the statement line and
marks it reconciled, so the cashbook is cleaned up in one click. Mapping:
`money_in → debit`, `money_out → credit`; `payment_date` (else `value_date`, else
now) → `transaction_at`.

Request body (`application/json`):
- `bank_entry` (object, required) — the `statement` object from the `unrecorded` bucket.

```json
{ "bank_entry": { "transaction_reference": null, "payment_date": "2025-05-31", "description": "Ledger fees", "money_in": null, "money_out": 350.0, "is_transaction_fee": true, "is_interest_earning": false } }
```

Response:
```json
{ "data": { "message": "Statement entry added to cashbook and reconciled", "result": { "bank_account_transaction_id": 144 } } }
```

The new cashbook row has no `transaction` morph (it is not a receipt/voucher);
its `reconciliation_status` is set to `matched` with the statement line on `bank_entry`.

---

## 4. Reconciliation report

`POST .../{account}/reconciliations/report`

Generate the period's bank reconciliation report — the classic "reco report".
Run it once the matches have been worked through; it is a **read-only** snapshot
emitted as a flat, ordered `rows[]` envelope that drives the frontend table and
the Excel/PDF export row-for-row.

**Two report types** (select via `report_type`):
- `full_account` (default) — accounts where receipts **and** payments happen. Proves
  the **adjusted cashbook balance** two ways and shows they agree: *from our books*
  (opening + receipts − payments ± omissions & errors) and *from the bank* (closing
  balance + uncredited receipts − unpresented payments ± bank issues / suspense).
- `credits_only` — collections accounts (receipting only); proves cashbook
  collections equal bank statement collections after timing/error adjustments.

Each unmatched item is first resolved to a [class](#classification-taxonomy); the
class's resolution type decides which proof it lands in and its sign.

**Period basis:** the reconciling period comes from the statement's
`statement_period`, applied to the cashbook row **`created_at`** (the date a row
was recorded in our books). `transaction_at` (the day a payment was presented to
the bank) is used only by the matching pass.

Request body (`application/json`):
- `statement_token` (string, required) — the token from the extract step.
- `report_type` (string, optional) — `full_account` (default) or `credits_only`.

```json
{ "statement_token": "9f1c2b6e-…", "report_type": "full_account" }
```

### Output model
One object. `rows` is the flat, ordered list; each row is
`{ label, description, amount, op }` where `op` ∈
`OPENING | ADD | LESS | SUBTOTAL | COMPARISON | RESULT`. Computing the report is a
single pass applying `op_sign * amount` (op_sign `+1` for ADD, `-1` for LESS):

- `OPENING` resets the running total to `amount` (each proof starts with one);
  `ADD`/`LESS` adjust it; `SUBTOTAL` snapshots it; `RESULT` = the difference between
  the **last two subtotals** (the two adjusted-cashbook balances that must agree).
  `COMPARISON` is legacy (external figure that does not change the running total)
  and is no longer emitted by either layout.
- Signs are **literal** — a negative `amount` on a `LESS` row adds back. Never
  auto-correct signs.
- `SUBTOTAL`/`RESULT` `amount`s are recomputed server-side, not trusted from input.
- Render `-` for blank/NA; treat as `0` in maths. 2 dp throughout.

`reconciling_difference.status` is `balanced` (0.00), `variance` (a genuine
unresolved figure — surface it), or `incomplete` (the statement closing balance
was unreadable, so the bank-side proof `OPENING` is `0`). `detail_schedule`
itemises each reconciling category; each group's `subtotal` ties back to its
matching summary row.

### Full response — `full_account`
Two blocks, each starting with an `OPENING` and closing with a `SUBTOTAL` — the two
**adjusted cashbook balances** that must agree; `RESULT` is their difference. Only
classes that actually occur produce a row (plus the always-present opening / period
movement / `Receipts from Prior Periods`), and rows are ordered by class code within
each block. This sample is fully **balanced** (`0.00`): both subtotals are
`12,184,960.01`.

```json
{
  "data": {
    "message": "Bank reconciliation report generated successfully",
    "result": {
      "report_type": "full_account",
      "account": { "name": "NW REALITE LTD", "number": "2051188554" },
      "period": { "start": "2026-07-01", "end": "2026-07-09" },
      "title": "BANK RECONCILIATION AS AT 9TH JULY 2026",
      "rows": [
        { "label": "Balance per Cashbook b/f as at 01/07/2026", "description": "", "amount": 11427317.51, "op": "OPENING" },
        { "label": "Receipts for the Month", "description": "", "amount": 2345000.00, "op": "ADD" },
        { "label": "Payments for the Month", "description": "", "amount": 1585000.00, "op": "LESS" },
        { "label": "Receipts from Prior Periods", "description": "A receipt recorded in the cashbook this period but banked in a prior period (receipted now, banked prior).", "amount": 0.00, "op": "LESS" },
        { "label": "Bank Charges & Ledger Fees", "description": "Account maintenance, transaction, RTGS, cheque book and service fees levied directly by the bank.", "amount": 2357.50, "op": "LESS" },
        { "label": "Adjusted Cashbook Balance", "description": "", "amount": 12184960.01, "op": "SUBTOTAL" },
        { "label": "Balance per Bank Statement as at 09/07/2026", "description": "", "amount": 11924960.01, "op": "OPENING" },
        { "label": "Unrecognised Credits on Bank Statement", "description": "Default holding class: a credit appears on the statement, is absent from the cashbook, and has not yet been investigated.", "amount": 590000.00, "op": "LESS" },
        { "label": "Unrecognised Debits on Bank Statement", "description": "Default holding class: a debit appears on the statement, is absent from the cashbook, and has not yet been investigated.", "amount": 1135000.00, "op": "ADD" },
        { "label": "Unpresented Payments (Cheques / EFTs Issued)", "description": "Payments correctly written and recorded, but the payee has not yet presented them, or the EFT had not settled by the statement date.", "amount": 1160000.00, "op": "LESS" },
        { "label": "Uncredited Receipts (Deposits in Transit)", "description": "Cash, cheques or transfers banked at or near period end that the bank had not credited by the statement date. Fully valid - just late.", "amount": 875000.00, "op": "ADD" },
        { "label": "Adjusted Cashbook Balance (per Bank Statement)", "description": "", "amount": 12184960.01, "op": "SUBTOTAL" },
        { "label": "Reconciling Difference", "description": "The two adjusted cashbook balances must agree", "amount": 0.00, "op": "RESULT" }
      ],
      "reconciling_difference": { "amount": 0.00, "status": "balanced" },
      "detail_schedule": [
        { "group": "Receipts from Prior Periods", "op": "LESS", "items": [], "subtotal": 0.00 },
        {
          "group": "Unrecognised Credits on Bank Statement",
          "op": "LESS",
          "items": [
            { "date": "2026-07-02", "particulars": "EFT CR SERVICE CHARGE - GRACE WANJIKU", "ref": "248312026070200002", "amount": 120000.00, "remarks": "" },
            { "date": "2026-07-09", "particulars": "EFT CR SERVICE CHARGE - SAMUEL KIPTOO", "ref": "248312026070900009", "amount": 95000.00, "remarks": "" },
            { "date": "2026-07-06", "particulars": "EFT CR RENT PAYMENT - PETER OTIENO", "ref": "248312026070600005", "amount": 375000.00, "remarks": "" }
          ],
          "subtotal": 590000.00
        },
        {
          "group": "Bank Charges & Ledger Fees",
          "op": "LESS",
          "items": [
            { "date": "2026-07-31", "particulars": "Ledger fees", "ref": null, "amount": 2357.50, "remarks": "" }
          ],
          "subtotal": 2357.50
        },
        {
          "group": "Unrecognised Debits on Bank Statement",
          "op": "ADD",
          "items": [
            { "date": "2026-07-06", "particulars": "RTGS OUT; MAINTENANCE WORKS - JANET WAMBUI", "ref": "853207060001000006", "amount": 320000.00, "remarks": "" },
            { "date": "2026-07-09", "particulars": "RTGS OUT; UTILITY PAYMENT - FELISTER KARIUKI", "ref": "853207090001000010", "amount": 65000.00, "remarks": "" },
            { "date": "2026-07-08", "particulars": "RTGS OUT; LANDLORD REMITTANCE - BRIAN WATHOME", "ref": "853207080001000008", "amount": 750000.00, "remarks": "" }
          ],
          "subtotal": 1135000.00
        },
        {
          "group": "Unpresented Payments (Cheques / EFTs Issued)",
          "op": "LESS",
          "items": [
            { "date": "2026-07-07", "particulars": "PV#3 - landlord - Brian Wathome", "ref": "853207080001000008", "amount": 750000.00, "remarks": "" },
            { "date": "2026-07-05", "particulars": "PV#4 - vendor - Omar Hassan", "ref": "PV-OUT-001", "amount": 410000.00, "remarks": "" }
          ],
          "subtotal": 1160000.00
        },
        {
          "group": "Uncredited Receipts (Deposits in Transit)",
          "op": "ADD",
          "items": [
            { "date": "2026-07-05", "particulars": "Receipt#3 - Peter Otieno", "ref": "248312026070600005", "amount": 375000.00, "remarks": "" },
            { "date": "2026-07-05", "particulars": "Receipt#4 - Lucy Wairimu", "ref": "RCPT-OUT-001", "amount": 500000.00, "remarks": "" }
          ],
          "subtotal": 875000.00
        }
      ]
    }
  }
}
```

Notes for this layout:
- **Two `OPENING`s, two `SUBTOTAL`s.** Block A is the adjusted cashbook *from our
  books*: opening b/f + receipts − payments − `Receipts from Prior Periods`, then
  omission/error adjustments (here, `Bank Charges & Ledger Fees` — bank-recorded,
  not yet in our books → `LESS`). Block B is the adjusted cashbook *from the bank*:
  the statement closing balance, then timing (`Uncredited Receipts` `ADD`,
  `Unpresented Payments` `LESS`), bank-issue and suspense items. `RESULT` is Block A
  `SUBTOTAL` − Block B `SUBTOTAL`; `0.00` means they agree.
- **Signs follow the class resolution type**, not a fixed rule: an omission/error
  adjusts our books (Block A); a timing/bank-issue/suspense item adjusts the bank
  (Block B). A bank-side inflow (uncredited receipt) `ADD`s to the bank; a bank-side
  outflow (unpresented payment) `LESS`es. `Unrecognised Credits`/`Debits` are the
  default holding classes (C10 / D14) for statement lines still under investigation.
- `detail_schedule` groups mirror the adjustment rows (one per class, ordered by
  code, with `Receipts from Prior Periods` first); each group's `subtotal`
  **equals** its matching summary row.

### Full response — `credits_only`
Two `OPENING`s (cashbook, then bank); the second resets the running total, and
`RESULT` = first `SUBTOTAL` − second `SUBTOTAL`. This sample is the KCB KPA
Pension, 31 May 2026 worked example (fully reconciled — `0.00`), and shows the
negative-amount add-back on a `LESS` row (`-100,000` → `+100,000`).

```json
{
  "data": {
    "message": "Bank reconciliation report generated successfully",
    "result": {
      "report_type": "credits_only",
      "account": { "name": "KCB KPA PENSION", "number": "1122334455" },
      "period": { "start": "2026-05-01", "end": "2026-05-31" },
      "title": "BANK RECONCILIATION AS AT 31ST MAY 2026",
      "rows": [
        { "label": "Collections as per Cash book", "description": "Receipts For The Month", "amount": 5505070.00, "op": "OPENING" },
        { "label": "Unreceipted Funds", "description": "In bank statement not in cashbook", "amount": 3581975.05, "op": "ADD" },
        { "label": "External Credits on Bank Statement", "description": "Bank credits classified as external", "amount": 0.00, "op": "ADD" },
        { "label": "External Debits on Bank Statement", "description": "Bank debits classified as external", "amount": 0.00, "op": "LESS" },
        { "label": "Payments Receipted in May but Banked in prior periods", "description": "", "amount": 2205331.00, "op": "LESS" },
        { "label": "Uncleared Cheques", "description": "", "amount": 0.00, "op": "LESS" },
        { "label": "Unbounced in May", "description": "Bounced in bank statement but not in cashbook", "amount": 0.00, "op": "LESS" },
        { "label": "Bounced in May", "description": "Bounced in bank statement and in cashbook now", "amount": 0.00, "op": "ADD" },
        { "label": "Banked in other Banks", "description": "In cashbook but not in bank statement", "amount": 0.00, "op": "LESS" },
        { "label": "Erroneous receipts", "description": "Double receipting or receipted to wrong person or cancelled and repaid", "amount": 0.00, "op": "LESS" },
        { "label": "Unpaid cheques from prior periods", "description": "Bounced in bank statement earlier but bounced in cashbook now", "amount": -100000.00, "op": "LESS" },
        { "label": "Adjusted Cashbook Collections", "description": "", "amount": 6981714.05, "op": "SUBTOTAL" },
        { "label": "Collections as per Bank Statement", "description": "", "amount": 6981714.05, "op": "OPENING" },
        { "label": "Unpaid Cheques", "description": "All bounced cheques in bank", "amount": 0.00, "op": "LESS" },
        { "label": "Inter account Transfers", "description": "", "amount": 0.00, "op": "LESS" },
        { "label": "Adjusted Bank Collections", "description": "", "amount": 6981714.05, "op": "SUBTOTAL" },
        { "label": "Reconciling Difference", "description": "Variance Cashbook Vs Bank", "amount": 0.00, "op": "RESULT" }
      ],
      "reconciling_difference": { "amount": 0.00, "status": "balanced" },
      "detail_schedule": [
        { "group": "Erroneous receipts", "op": "LESS", "items": [], "subtotal": 0.00 },
        { "group": "Banked in other Banks", "op": "LESS", "items": [], "subtotal": 0.00 },
        { "group": "External Credits on Bank Statement", "op": "ADD", "items": [], "subtotal": 0.00 },
        { "group": "External Debits on Bank Statement", "op": "LESS", "items": [], "subtotal": 0.00 }
      ]
    }
  }
}
```

The `detail_schedule` groups are populated from
[classifications](#5-classify-lines-during-the-workings). External classifications
are shown **gross**, split by side: bank credits (`money_in`) go to the `ADD:
External Credits on Bank Statement` group, bank debits (`money_out`) to the `LESS:
External Debits on Bank Statement` group — e.g. after classifying one credit and
one debit line:

```json
[
  {
    "group": "External Credits on Bank Statement",
    "op": "ADD",
    "items": [
      { "date": "2026-05-15", "particulars": "Rent - Westlands Plaza", "ref": null, "amount": 60000.00, "remarks": "Owner's direct collection" }
    ],
    "subtotal": 60000.00
  },
  {
    "group": "External Debits on Bank Statement",
    "op": "LESS",
    "items": [
      { "date": "2026-05-22", "particulars": "Service charge remittance", "ref": "REF-88", "amount": 12000.00, "remarks": "Remitted to owner" }
    ],
    "subtotal": 12000.00
  }
]
```
(each group's `subtotal` ties back to its summary row; the `ADD` credits row reads
`60000.00` and the `LESS` debits row `12000.00`, a net effect of `+48000.00`.)

### Field reference (report envelope)
- `report_type` — `full_account` or `credits_only` (echoes the request).
- `account.name` / `account.number` — from the bank account record (never a system name).
- `period.start` / `period.end` — ISO `YYYY-MM-DD`, the reconciling window.
- `title` — pre-formatted header, long-form date (`30TH JUNE 2026`).
- `rows[]` — ordered; each `{ label, description, amount, op }`:
  - `label` — short row name. For `ADD` / `LESS` rows, prefix it with `ADD: ` /
    `LESS: ` when rendering.
  - `description` — explanatory clause (may be `""`).
  - `amount` — 2 dp; **may be negative**. Render `0.00` as `-` if you prefer,
    but it is a real `0` in the maths.
  - `op` — `OPENING` / `ADD` / `LESS` / `SUBTOTAL` / `RESULT` (`COMPARISON` is
    legacy and no longer emitted). Bold `OPENING` / `SUBTOTAL` / `RESULT`; highlight
    `SUBTOTAL` with a top rule; colour the `RESULT` cell green when `balanced`,
    red / amber otherwise.
- `reconciling_difference.amount` — the `RESULT` figure (target `0.00`).
- `reconciling_difference.status` — `balanced` (0.00) / `variance` (non-zero,
  surface it) / `incomplete` (statement closing balance unreadable; the bank-side
  `OPENING` is `0` and the difference is not meaningful).
- `detail_schedule[]` — supporting itemisation; each
  `{ group, op, items[], subtotal }`, `items[]` =
  `{ date, particulars, ref, amount, remarks }`. Each `subtotal` equals its
  matching summary row.

- All amounts 2 dp; no intermediate rounding.
- The report is deterministic and read-only — regenerate freely after each resolve
  to watch the difference close.

---

## Classification taxonomy

`GET .../{account}/reconciliations/classifications`

Read-only, account-independent. Returns every reconciliation class so the UI can
build the classify pickers (with descriptions) and resolve an item's
`default_classification`. Fetch once and cache for the session.

Response:
```json
{
  "data": {
    "message": "Reconciliation classifications retrieved successfully",
    "result": {
      "cashbook": [
        { "value": "uncredited_receipts", "code": "R1", "label": "Uncredited Receipts (Deposits in Transit)", "description": "Cash, cheques or transfers banked at or near period end that the bank had not credited by the statement date. Fully valid - just late.", "side": "receipt", "resolution_type": "timing", "landing_place": "reconciliation_statement" }
      ],
      "statement": [
        { "value": "bank_charge", "code": "D1", "label": "Bank Charges & Ledger Fees", "description": "Account maintenance, transaction, RTGS, cheque book and service fees levied directly by the bank.", "side": "debit", "resolution_type": "omission", "landing_place": "adjusted_cashbook" }
      ]
    }
  }
}
```

- **`cashbook[]`** — 22 classes: receipts **R1–R11**, payments **P1–P11**. Use these
  to classify an outstanding **cashbook row** (§5.1).
- **`statement[]`** — 24 classes: credits **C1–C10**, debits **D1–D14**. Use these
  to classify an unrecorded **statement line** (§5.2).
- Per class: `value` (the stored/POSTed value), `code` (R1…, C1…), `label`,
  `description` (render under the label), `side`
  (`receipt`|`payment`|`credit`|`debit`), `resolution_type`
  (`timing`|`omission`|`error`|`bank_issue`|`suspense`), and `landing_place`
  (`adjusted_cashbook`|`reconciliation_statement`|`suspense_queue`).

Grouping the picker by `resolution_type` mirrors how the report treats each class:
timing & bank-issue items stay on the reconciliation statement; omissions & errors
are journalised into the adjusted cashbook; suspense items are aged.

---

## 5. Classify lines during the workings

Let the reconciler *classify* leftover lines the flow can't auto-derive; the
report reads those classifications into its adjustment lines. Both endpoints are
**idempotent**, re-upload-safe, and authorize `view` on the account. On the next
**match** run, classified items move to the `classified` bucket (out of
`outstanding` / `unrecorded`).

### 5.1 Classify an outstanding cashbook row
`POST .../{account}/reconciliations/{cashbookEntry}/classify`

`{cashbookEntry}` = the `cashbook.id` from the `outstanding` bucket. A classified
row is resolved **without** a statement match (its `reconciliation_status` stays
`pending`).

Request body (`application/json`):
- `classification` (present, nullable) — any **cashbook** class `value` (R1–R11 /
  P1–P11) from the [classifications endpoint](#classification-taxonomy) — e.g.
  `erroneous_receipt`, `banked_in_other_bank`, `receipts_from_prior_periods` — or
  `null` to clear.
- `notes` (optional) — reconciler note shown in the report detail schedule.

```json
{ "classification": "erroneous_receipt", "notes": "Double receipted — RCT-0031 & RCT-0032" }
```

Response:
```json
{ "data": { "message": "Cashbook entry classified", "result": { "cashbook_id": 77, "classification": "erroneous_receipt" } } }
```
`404` if the entry is not on this account. `null` clears and the row returns to
`outstanding`. A `422` is returned for a value that is not a cashbook class.

### 5.2 Classify an unrecorded statement line
`POST .../{account}/reconciliations/statement-lines/classify`

Unrecorded lines have no cashbook row, so the classification is persisted keyed by
the statement line's identity (reference + dates + amounts) so it survives
re-extraction. Any unrecorded line carries `can_classify: true`. The class you send
determines how the line is treated in the report via its resolution type — e.g. a
bank charge (`bank_charge`, D1, omission) is journalised into the adjusted cashbook,
while an external transaction (`external_transaction`, C8 / `external_transaction_debit`,
D13, bank issue) stays on the reconciliation statement.

Request body (`application/json`):
- `statement_token` (required) — token from extract.
- `classification` (present, nullable) — any **statement** class `value` (C1–C10 /
  D1–D14) from the [classifications endpoint](#classification-taxonomy), or `null`
  to clear.
- `bank_entry` (required) — the `statement` object from the `unrecorded` bucket (money_in **or** money_out).
- `notes` (optional).

```json
{ "statement_token": "9f1c2b6e-…", "classification": "external_transaction", "bank_entry": { "transaction_reference": null, "value_date": "2025-05-15", "payment_date": "2025-05-15", "description": "Rent - Westlands Plaza", "money_in": 60000.0, "money_out": null } }
```

Response:
```json
{ "data": { "message": "Statement line classified", "result": { "classification": "external_transaction" } } }
```
`422` if the token expired / belongs to another account, or the classification is
not a statement class. In the `credits_only` report, external classifications are
shown gross — credits into the **ADD: External Credits on Bank Statement** row and
debits into the **LESS: External Debits on Bank Statement** row.

---

## Notes
- Extraction and matching are synchronous and may take several seconds (AI calls).
  Matching only calls AI for sides that still have unmatched lines.
- Always check `is_legible` before using `result`, and `matches_account` before
  reconciling.
- `statement_token` expires (~1 hour). Re-upload to get a fresh one.
- Re-running **match** never re-touches already-matched cashbook rows (they are
  excluded once `reconciliation_status` is `matched`).
- **Re-uploading the same statement** is safe: any line already recorded on a
  previous run (a prior match, or a charge / interest added via step 3.3) is
  recognised and returned under `matched` — it is **not** offered for matching or
  "add to cashbook" again. Recognition is by the statement line's reference +
  dates + amounts (description is used only when there is no reference), so it is
  stable across re-extractions. Adding the same line twice is also a no-op on the
  server (it returns the existing cashbook row).
- The **report** is deterministic (no AI) and read-only, so it can be regenerated
  freely — it always reflects the current cashbook state. Re-run it after each
  resolve to watch the variance close.

## Errors
- `422 Unprocessable Entity` — validation failed (missing/invalid `file`,
  `statement_token`, `match`, `report_type`, or `classification`), or the token
  expired / belongs to another account.
- `403 Forbidden` — the user cannot view the bank account.
- `404 Not Found` — the cashbook entry, receipt, or voucher does not belong to this account.
- `500 Internal Server Error` — extraction failed. (Matching does **not** 500 on
  an AI outage — it degrades; see `summary.degraded`.)
