# Invoice Backdate (Change Date)

This moves an issued invoice to a different date: the invoice's own `created_at` and
`transaction_at`, the date its statement line is recorded under, and the date of its lease
billings. The amount, the items and `due_at` stay as they were.

Base prefix: `/api/v1/app/{company}`

| Method | URI | Name |
|---|---|---|
| `PATCH` | `.../property-management/lease-management/invoices/{invoice}/backdate` | `app.property-management.lease-management.invoices.backdate` |

Permission: `backdate-facility-invoice` (`FacilityInvoicePolicy::backdate`). The ability is also
exposed on each record as `permissions.backdate` on `FacilityInvoiceResource`.

## Request

```json
{
  "new_invoice_date": "2026-08-15"
}
```

| Field | Required | Type | Rules |
|---|---|---|---|
| `new_invoice_date` | Yes | date (`Y-m-d`) | Must be a valid date on or before today. |

The invoice and its statement line keep their original time of day.

## Rules

| Rule | Where | Result |
|---|---|---|
| Invoice is `pending`, `cancelled` or `rejected` | Policy | `403` |
| Invoice has been reversed by a credit note (`reversal_credit_note_id` set) | Policy | `403` |

Invoices are not part of landlord remittances (remittances select collections and expenses), so
unlike receipts and bills no remittance blocks the move.

## What moves

| Artifact | Field | Effect |
|---|---|---|
| `TenantStatement` (the invoice's lines) | `transaction_at`, `created_at` | Set to the new date, keeping the line's time of day. |
| `LeaseBilling` (rows with this `invoice_id`) | `transaction_date` | Set to the new date at midnight. Reports select billings with `whereBetween` on day boundaries, so a time of day would drop the last day of a period. |
| `LeaseBilling` | `created_at` | Set to the new date, keeping the statement's time of day. |
| `FacilityInvoice` | `created_at`, `transaction_at` | Set to the new date, keeping the time of day. `updated_at` is left alone. The invoice list, the view page and the PDF show the new date. |
| `FacilityInvoice` | `due_at` | **Not** changed. |

## Response

`200`

```json
{
  "data": {
    "message": "Invoice date changed successfully",
    "invoice": { "id": 42, "...": "..." }
  }
}
```

## Implementation

| Concern | Class |
|---|---|
| Controller | `App\Http\Controllers\Api\V1\App\PropertyManagement\LeaseManagement\Billing\BackdateInvoiceController` |
| Request data | `App\Data\BackdateInvoiceData` |
| Action | `App\Actions\PropertyManagement\LeaseManagement\Invoice\BackdateInvoiceAction` |
| Policy | `App\Policies\FacilityInvoicePolicy::backdate` |

## Backdating opening balances in bulk (console)

This command moves every opening balance invoice **and** opening balance credit note on the given
properties to one date. There is no endpoint.

```bash
php artisan lease-opening-balances:backdate 125,5662                     # report only, nothing changes
php artisan lease-opening-balances:backdate 125,5662 --apply             # move them to 2025-09-30
php artisan lease-opening-balances:backdate 125,5662 --date=2025-06-30 --apply
```

| Input | Default | Notes |
|---|---|---|
| `properties` | — | Comma-separated property (`facilities.id`) list. Required. |
| `--date` | `2025-09-30` | `Y-m-d`, today or earlier. |
| `--apply` | off | Without it the command prints what it would move and writes nothing. |

A document is an opening balance when a `lease_opening_balances` row points at it (`invoice_id` or
`credit_note_id`). The notes text is not used: it also matches reversal and "Credit for INV#…" credit
notes raised against an opening balance invoice, and those must stay where they are. Every status
is moved. A pending document has no ledger rows yet, so moving its `transaction_at` makes processing
post it on the new date.

What moves, all in one transaction:

| Artifact | Field | Effect |
|---|---|---|
| Opening balance invoice | as [What moves](#what-moves) | `BackdateInvoiceAction`, the same action as the endpoint. |
| `FacilityCreditNote` | `created_at`, `transaction_at` | Set to the new date, keeping the time of day. `updated_at` is left alone. |
| `TenantStatement` (the credit note's line) | `transaction_at`, `created_at` | Set to the new date, keeping the line's time of day. |
| `LeaseBilling` (rows with this `credit_note_id`) | `transaction_date` | The new date at midnight. |
| `LeaseBilling` | `created_at` | The new date, keeping the time of day. |
| `LeaseOpeningBalance` | `opening_balance_at` | The new date, so the opening balance and its documents agree. |

Not changed: `due_at`, the notes text (EPMAS titles read "Opening balance as at 30 Sep 2026 – …"),
and credit note applications (`facility_credit_note_applications`).

| Concern | Class |
|---|---|
| Command | `App\Console\Commands\BackdateOpeningBalancesCommand` |
| Credit note action | `App\Actions\PropertyManagement\LeaseManagement\CreditNote\BackdateCreditNoteAction` |
