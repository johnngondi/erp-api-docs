# Receipt Backdate (Change Date)

Moves a confirmed receipt to an earlier date, for example when it was captured late. The
amount, allocations and the receipt's own `transaction_date` stay unchanged. What moves is
the date the receipt, its ledger lines and its collections are recorded under.

Base prefix: `/api/v1/app/{company}`

| Method | URI | Name |
|---|---|---|
| `PATCH` | `.../property-management/lease-management/receipts/{receipt}/backdate` | `app.property-management.lease-management.receipts.backdate` |

Permission: `backdate-facility-receipt` (`FacilityReceiptPolicy::backdate`). The ability is also
exposed per record as `permissions.backdate` on `FacilityReceiptResource`, so the UI can hide the
action.

## Request

```json
{
  "new_receipt_date": "2026-08-15"
}
```

| Field | Required | Type | Rules |
|---|---|---|---|
| `new_receipt_date` | Yes | date (`Y-m-d`) | Must be a valid date on or before today. |

The receipt's original time of day is kept, so receipts entered on the same day stay in order.

## Rules

| Rule | Where | Result |
|---|---|---|
| Receipt is not `confirmed`, or is reversed | Policy | `403` |
| Receipt is attached to any remittance (`facility_receipt_remittance`) | Policy (and again in the action as `422` on `receipt`) | `403` |
| A **non-advance** remittance (`is_advance = false`) for the receipt's property, not `cancelled`/`rejected`, has a period (`period_from`–`period_to`) that overlaps the target month | Action | `422` on `new_receipt_date`: pick a different month |
| Only advance remittances cover the month | — | Allowed |

## What moves

| Artifact | Field | Effect |
|---|---|---|
| `FacilityReceipt` | `created_at` | Set to the new date. `transaction_date` is **not** changed. |
| `TenantStatement` (the receipt's rows) | `transaction_at` | Set to the new date. |
| `BankAccountTransaction` (the receipt's cashbook row) | `created_at` | Set to the new date. `transaction_at` is **not** changed. |
| `LeaseCollection` (under the receipt's allocations) | `transaction_date` | Set to the new date, so the money counts in the backdated month when that month's remittance is computed. |

## Response

`200`

```json
{
  "data": {
    "message": "Receipt date changed successfully",
    "receipt": { "id": 12, "created_at": { "raw": "2026-08-15T09:12:44Z", "...": "..." }, "...": "..." }
  }
}
```

`422` (month already remitted):

```json
{
  "message": "This month has already been remitted. Choose a date in a different month.",
  "errors": {
    "new_receipt_date": ["This month has already been remitted. Choose a date in a different month."]
  }
}
```

## Implementation

| Concern | Class |
|---|---|
| Controller | `App\Http\Controllers\Api\V1\App\PropertyManagement\LeaseManagement\Receipting\BackdateReceiptController` |
| Request data | `App\Data\BackdateReceiptData` |
| Action | `App\Actions\PropertyManagement\LeaseManagement\Receipt\BackdateReceiptAction` |
| Policy | `App\Policies\FacilityReceiptPolicy::backdate` |
