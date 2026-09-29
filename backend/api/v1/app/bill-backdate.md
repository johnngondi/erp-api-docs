# Bill Backdate (Change Date)

This moves a posted bill to a different posting date: the date its expense is recorded under.
It changes nothing else. The amount, the invoice details, `invoice_date` (the date the invoice was
generated) and `posted_at` (when the bill reached the ledger) all stay as they were.

Base prefix: `/api/v1/app/{company}`

| Method | URI | Name |
|---|---|---|
| `PATCH` | `.../property-management/finance/bills/{bill}/backdate` | `app.property-management.finance.bills.backdate` |

Permission: `backdate-facility-bill` (`FacilityBillPolicy::backdate`). The ability is also exposed
on each record as `permissions.backdate` on `FacilityBillResource`.

## Request

```json
{
  "new_bill_posted_date": "2026-08-15"
}
```

| Field | Required | Type | Rules |
|---|---|---|---|
| `new_bill_posted_date` | Yes | date (`Y-m-d`) | Must be a valid date on or before today. It may be before `invoice_date`. |

The original time of day from `expense_posted_at` is kept.

## Rules

| Rule | Where | Result |
|---|---|---|
| Bill is `pending` or `cancelled`, was never posted (`expense_posted_at` null), or has no expense | Policy | `403` |
| The bill's expense is attached to any remittance (`facility_expense_remittance`) | Policy, then again in the action as a `422` on `bill` | `403` |
| A **non-advance** remittance for the bill's property (not `cancelled`/`rejected`) has a period (`period_from` to `period_to`) that overlaps the target month | Action | `422` on `new_bill_posted_date`: the user must pick a different month |
| Only advance remittances cover the month | — | Allowed |

Credit notes are bills with their own (negative) expense, and they follow the same rules.

## What moves

| Artifact | Field | Effect |
|---|---|---|
| `FacilityBill` | `expense_posted_at` | Set to the new date. |
| `FacilityBill` | `invoice_date`, `posted_at` | **Not** changed. |
| `FacilityExpense` (the bill's expense) | `transaction_at`, `created_at` | Set to the new date. `transaction_at` is what remittances select expenses by. |

Editing `invoice_date` through the bill update endpoint no longer moves `expense_posted_at`. This
endpoint is the only way to change the posting date.

## Response

`200`

```json
{
  "data": {
    "message": "Bill date changed successfully",
    "bill": { "id": 7, "expense_posted_at": { "raw": "2026-08-15T09:12:44Z", "...": "..." }, "...": "..." }
  }
}
```

`422` (month already remitted):

```json
{
  "message": "This month has already been remitted. Choose a date in a different month.",
  "errors": {
    "new_bill_posted_date": ["This month has already been remitted. Choose a date in a different month."]
  }
}
```

## Implementation

| Concern | Class |
|---|---|
| Controller | `App\Http\Controllers\Api\V1\App\PropertyManagement\Finance\Bills\BackdateBillController` |
| Request data | `App\Data\BackdateBillData` |
| Action | `App\Actions\PropertyManagement\Finance\Bill\BackdateBillAction` |
| Policy | `App\Policies\FacilityBillPolicy::backdate` |
