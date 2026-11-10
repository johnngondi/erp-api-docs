# Expenses API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/expenses`

## Dates

`transaction_at` come back with `raw` as a **plain date**, `"2026-10-01"` (row 142). They used to be UTC
timestamps - `"2026-09-30T21:00:00.000000Z"` for 1 Oct in Nairobi - and a form that sent `raw` back
saved the day before. The server now reads any date it is sent as the day it means in
Nairobi, but send `YYYY-MM-DD`.

## Endpoints

- `GET /expenses`
- `POST /expenses`
- `GET /expenses/{expense}`
- `PUT/PATCH /expenses/{expense}`
- `DELETE /expenses/{expense}`
- `PATCH /expenses/{expense}/mark-paid`
- `PATCH /expenses/{expense}/cancel`

## Cancelling: cancelled or reversed

`PATCH /expenses/{expense}/cancel` does one of two things (since 2026-11-04):

| The expense | Result |
|---|---|
| Unpaid, dated in the current month | Status `cancelled`. It drops out of every report and remittance. |
| Paid, or dated in an earlier month | Status `reversed`, plus a negative **reversal entry** dated today (`notes`: `Reversal of expense #{id}`, `reverses_expense_id`: the original's id). |

A `reversed` expense is **not** dropped: it keeps counting in the month it was posted, and its
reversal offsets it in the month the reversal is dated. Reports for a past month therefore do not
change when something in it is cancelled later. It used to be marked `cancelled`, which took it
out of its own month while the reversal still counted in the current one, so the cancellation
was counted twice.

Cancelling a `cancelled` or `reversed` expense is refused (`422`).

In remittances:

- A `reversed` expense is never newly remitted. If it was not remitted before the reversal, its
  reversal entry is not remitted either, because the two cancel out.
- If the original was remitted (it sits on a live remittance), its reversal entry is remitted in
  a later period as a negative expense, which credits the landlord back.

The response field `reverses_expense_id` is `null` on every expense except a reversal entry.

## List Expenses

`GET /api/v1/app/{company}/property-management/finance/expenses`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and invoice numbers)
  - `filter[facility_id]`, `filter[vendor_id]`, `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[status]`

**Ids, `status` and `type` match exactly.** They were partial until 2026-09-29, so
`filter[status]=paid` also returned `unpaid` and `partially-paid`, and `filter[facility_id]=6`
returned rows for facility 60. Send the whole value. Date filters are still partial, so
`filter[created_at]=2026-09` means that month.
Every date filter also takes a range, `filter[created_between]=2026-09-01,2026-09-30`; see
[List filters](../../../list-filters.md) for each one's name.

  - `filter[created_at]`, `filter[transaction_at]`
- Sort:
  - `sort=id,invoice_number,amount,tax,total,paid,balance,transaction_at,created_at,updated_at`
- Include:
  - `include=taxType,invoiceUpload,creatingUser,bill`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[status]`: `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled`, `reversed` (from `ExpenseStatus` enum)

Sample list response (`FacilityExpenseResource`):

```json
{
  "data": [
    {
      "id": 501,
      "notes": "Electrical repairs",
      "invoice_number": "EXP-2026-001",
      "tax_invoice_number": "TEXP-2026-001",
      "amount": "800.00",
      "tax_amount": "128.00",
      "total": "928.00",
      "paid": "100.00",
      "balance": "828.00",
      "transaction_at": {
        "raw": "2026-05-04T00:00:00.000000Z",
        "formatted": "04 May, 2026",
        "diff": "5 days ago"
      },
      "vendor": { "id": 7, "name": "Acme Vendor" },
      "facility": { "id": 22, "name": "Riverside Plaza" },
      "status": { "value": "partially-paid", "color": "primary" },
      "created": {
        "raw": "2026-05-09T10:12:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      },
      "updated": {
        "raw": "2026-05-09T10:12:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      }
    }
  ]
}
```

## Create payload (`ExpenseData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `transaction_at` | Yes | date | - |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `expense_category_id` | No | integer | Must exist in `expense_categories.id`; when omitted, backend derives from `expense_type_id` |
| `invoice_number` | Yes | string | - |
| `amount` | Yes | number | - |
| `status` | No | string | `pending`, `unpaid`, `partially-paid`, `paid`, `cancelled` |
| `creating_user_id` | No | integer | Must exist in `users.id` |
| `notes` | No | string | Optional |
| `bill_id` | No | integer | Must exist in `bills.id` |
| `invoice_upload_id` | No | integer | Must exist in `uploads.id` |
| `tax_invoice_number` | No | string | Optional |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `paid` | No | number | Default `0` |

## Update payload (`PUT/PATCH /expenses/{expense}`)

Only these six can be changed after an expense is raised. Everything else is settled at creation
and is ignored if sent.

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `notes` | No | string \| null | - |
| `invoice_number` | No | string | Cannot be set to empty when sent |
| `invoice_upload_id` | No | integer \| null | Must exist in `uploads.id` |
| `tax_invoice_number` | No | string \| null | - |
| `transaction_at` | No | date | Must not be in the future, and must fall in the current month |
| `expense_category_id` | No | integer \| null | Must exist in `expense_categories.id` |

Notes:

- **Send only what you are changing.** An omitted field keeps its stored value. Editing one field
  no longer requires resending the other five - doing so previously returned a 500.
- The `transaction_at` restriction applies only when you are **moving** the date. An expense raised
  in an earlier month can still have its other fields corrected; its stored date is left untouched
  and is not re-checked.

