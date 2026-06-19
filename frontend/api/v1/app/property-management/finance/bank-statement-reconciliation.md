# Bank Statement Reconciliation (AI Extraction) API

Reads an uploaded bank statement with Claude and returns the account summary
and every transaction line item as structured data. The statement's account
number is reconciled against the bank account it is being attached to, so a
statement belonging to a different account is flagged up-front.

Supports **PDF** statements (read natively) and **spreadsheets** (`xlsx`, `xls`,
`csv`, `ods`), which are converted to text before extraction.

## Endpoint

### Extract & reconcile a statement against a bank account
`POST /api/v1/app/{company}/property-management/settings/finance/banks/accounts/{account}/reconciliations/extract`

Send the statement directly as `multipart/form-data`. The file is processed
in-memory for the reconciliation pre-check and is **not** persisted.

Request body (`multipart/form-data`):
- `file` (file, required) — the statement. Allowed types: `pdf`, `xlsx`, `xls`, `csv`, `ods`. Max size: 10 MB.

Response:
```json
{
  "data": {
    "message": "Bank statement extracted successfully",
    "reconciliation": {
      "matches_account": true,
      "expected_account_number": "0123456789",
      "statement_account_number": "0123456789"
    },
    "result": {
      "account_summary": {
        "account_number": "0123456789",
        "account_name": "JOHN DOE",
        "statement_period": { "from": "2025-05-01", "to": "2025-05-31" }
      },
      "transactions": [
        {
          "transaction_reference": "CHQ25145TT22",
          "value_date": "2025-05-10",
          "payment_date": "2025-05-08",
          "description": "Cheque deposit ref 004512",
          "money_in": 25000.00,
          "money_out": null
        }
      ],
      "is_legible": true,
      "feedback": null
    }
  }
}
```

Field reference (`result`):
- `account_summary.account_number` — account number as printed on the statement.
- `account_summary.account_name` — account holder name.
- `account_summary.statement_period.from` / `.to` — statement period (ISO-8601 `YYYY-MM-DD`).
- `transactions[]`:
  - `transaction_reference` — statement reference / cheque number.
  - `value_date` — date the money reflected in the account.
  - `payment_date` — date the payment was made (also called posting date).
  - `description` — transaction notes / details.
  - `money_in` — credit amount (`null` when the row is a debit).
  - `money_out` — debit amount (`null` when the row is a credit).
- `is_legible` — `false` when the document could not be read; see `feedback`.
- `feedback` — user-facing message (e.g. why a document was rejected); `null` when legible.

Reconciliation (`reconciliation`):
- `matches_account` — `true` when the statement's account number matches the
  target bank account (compared ignoring spaces and case).
- `expected_account_number` — the bank account's number.
- `statement_account_number` — the account number read from the statement (may be `null`).

### Not-legible response
When the statement cannot be read, `200 OK` is still returned with empty data
and an explanation for the user:
```json
{
  "data": {
    "message": "Bank statement extracted successfully",
    "reconciliation": {
      "matches_account": false,
      "expected_account_number": "0123456789",
      "statement_account_number": null
    },
    "result": {
      "account_summary": null,
      "transactions": [],
      "is_legible": false,
      "feedback": "The uploaded file is too blurry to read. Please upload a clearer PDF or the original spreadsheet."
    }
  }
}
```

## Notes
- Extraction is synchronous and may take several seconds for large statements.
- Always check `is_legible` before using `result`, and `reconciliation.matches_account`
  before attaching the statement to the account.

## Errors
- `422 Unprocessable Entity` — `file` missing, not a file, an unsupported type, or over 10 MB.
- `403 Forbidden` — the authenticated user cannot view the bank account.
- `500 Internal Server Error` — extraction failed (e.g. unsupported file type, or the AI provider errored).
