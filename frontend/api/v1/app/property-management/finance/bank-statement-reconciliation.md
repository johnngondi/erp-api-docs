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
   - if it is a bank-side entry with nothing behind it (bank charge, deposit
     interest), add it straight to the cashbook as a reconciled line.
4. **Report** — once the matches are worked through, generate the bank
   reconciliation statement: cashbook balances → known adjustments → balance per
   the bank → variance.

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
            "add_to_cashbook": false
          },
          {
            "statement": { "transaction_reference": null, "value_date": "2025-05-31", "payment_date": "2025-05-31", "description": "Interest credited", "money_in": 120.5, "money_out": null, "is_transaction_fee": false, "is_interest_earning": true },
            "candidates": [],
            "add_to_cashbook": true
          }
        ],
        "money_outs": [
          {
            "statement": { "transaction_reference": null, "value_date": "2025-05-31", "payment_date": "2025-05-31", "description": "Ledger fees", "money_in": null, "money_out": 350.0, "is_transaction_fee": true, "is_interest_earning": false },
            "candidates": [],
            "add_to_cashbook": true
          }
        ]
      },
      "outstanding": {
        "money_ins": [
          { "id": 77, "refference_number": "DEP100", "transaction_at": "2025-05-02", "debit": "8000.00000", "credit": "0.00000", "notes": "Receipt#9 - …", "payment_method": "Cash", "transaction": { "type": "FacilityReceipt", "id": 9 } }
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
  (truly not in our books). `add_to_cashbook` is `true` for bank-side lines
  (charges / interest) that can be booked directly — see step 3.3.
- **`outstanding`** — cashbook rows with no statement counterpart (in our books,
  not yet on the statement); no children.

`summary` carries the total count per bucket across both sides, plus
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

### 3.3 Add to cashbook (bank charges / interest)
`POST .../{account}/reconciliations/cashbook-entries`

For an `unrecorded` line flagged `add_to_cashbook: true` (bank charges, deposit
interest — entries that have no receipt or voucher behind them). This creates a
cashbook line directly from the statement line and marks it reconciled, so the
cashbook is cleaned up in one click. Mapping: `money_in → debit`,
`money_out → credit`; `payment_date` (else `value_date`, else now) → `transaction_at`.

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

Generate the period's bank reconciliation statement — the classic "reco report".
Run it once the matches have been worked through; it is a **read-only** snapshot
that walks from the cashbook balance, through the known timing/recording
differences, to the balance per the bank statement, and shows the variance.

The period comes from the statement's `statement_period`. Cashbook balances are
computed over `transaction_at` (= the statement `payment_date`), so they tie out
with the account statement shown elsewhere.

Request body (`application/json`):
- `statement_token` (string, required) — the token from the extract step.

```json
{ "statement_token": "9f1c2b6e-…" }
```

### What it computes
| Report line | Source |
| --- | --- |
| **CashBook balance b/f** (`cashbook_opening_balance`) | net `debit − credit` of cashbook rows before the period |
| **Receipts** / **Payments** for the period | sum of `debit` / `credit` within the period |
| **CashBook balance c/f** (`cashbook_closing_balance`) | b/f + receipts − payments |
| **ADD: Unpresented cheques** | pending cashbook **credits** (payments booked, not yet on the statement) |
| **LESS: Uncredited bankings** | pending cashbook **debits** (receipts booked, not yet on the statement) |
| **ADD: Unknown credits on statement** | statement `money_in` lines not recorded in the cashbook |
| **LESS: Unknown debits on statement** | statement `money_out` lines not recorded |
| **LESS: Bank charges** | unrecorded statement lines flagged `is_transaction_fee` |
| **Bank reconciliation balance** | c/f + unpresented − uncredited + unknown credits − unknown debits − bank charges |
| **Bank balance per statement** | the extracted `closing_balance` |
| **Variance** | reconciliation balance − bank balance (`0` when fully reconciled) |

A statement line counts as *recorded* when it has been matched to a cashbook row
(stored on that row's `bank_entry`); the rest fall into the "unknown" lines above.

### Response
```json
{
  "data": {
    "message": "Bank reconciliation report generated successfully",
    "result": {
      "period_from": "2025-04-01",
      "period_to": "2025-04-30",
      "cashbook_opening_balance": 209412.10,
      "receipts": 1327525.00,
      "payments": 1480088.27,
      "cashbook_closing_balance": 56848.83,
      "unpresented_cheques": 0.00,
      "uncredited_bankings": 0.00,
      "unknown_credits": 0.00,
      "unknown_debits": 0.00,
      "bank_charges": 0.00,
      "reconciliation_balance": 56848.83,
      "bank_statement_balance": 56849.85,
      "variance": -1.02,
      "details": {
        "unpresented_cheques": [],
        "uncredited_bankings": [],
        "unknown_credits": [],
        "unknown_debits": [],
        "bank_charges": []
      }
    }
  }
}
```

- All amounts are rounded to 2dp.
- `bank_statement_balance` and `variance` are `null` when the statement's closing
  balance could not be read from the document.
- `details` carries the supporting line items behind each adjustment (cashbook
  rows for `unpresented_cheques` / `uncredited_bankings`, statement lines for the
  rest) for the report's lower "Date / Detail / Amount" section.

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
  `statement_token`, or `match`), or the token expired / belongs to another account.
- `403 Forbidden` — the user cannot view the bank account.
- `404 Not Found` — the cashbook entry, receipt, or voucher does not belong to this account.
- `500 Internal Server Error` — extraction failed. (Matching does **not** 500 on
  an AI outage — it degrades; see `summary.degraded`.)
