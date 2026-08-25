# Receipt Reallocation (Frontend)

Domain: `Property Management > Lease Management`

Lets an operator move a **confirmed** receipt's money between the tenant's invoices without
cancelling and re-keying it. The receipt keeps its amount, transaction number and cashbook
entry — only which invoices it settles changes.

Base route:

`/api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/reallocation`

Backend reference: `docs/backend/api/v1/app/receipt-reallocation.md`.

## Endpoints

- `GET /receipts/{receipt}/reallocation` — load the screen (current split + allocatable invoices)
- `PUT /receipts/{receipt}/reallocation` — commit the new split

Both require the `reallocate-facility-receipt` permission.

## Where the action lives

Add **Reallocate** to the existing row action split button in two places:

1. **Receipt list** — the per-row split button, alongside `View`, `Edit`, `Cancel`, `Dispute`.
2. **Receipt details / information page** — the header split button, same group.

Placement within the menu: after `Edit`, before `Cancel`. It is a non-destructive action, so
use the default item style (not the danger style used for `Cancel` / `Delete`).

### When to show it

Drive visibility off the record, never off a hardcoded role check:

```js
const canReallocate = receipt.permissions?.reallocate === true
```

`permissions.reallocate` is returned by `FacilityReceiptResource` on both the list and the
detail payloads. The backend already returns `false` when the receipt is not `confirmed` or
has been reversed, so no extra client-side status check is needed. Hide the item entirely
when it is `false` — do not render it disabled.

> The list endpoint (`GET /receipts`) returns `permissions` per row, so the split button can
> be built without an extra request.

## Loading the screen

`GET /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/reallocation`

```json
{
  "receipt": {
    "id": 12,
    "transaction_number": "RCPT-1005",
    "amount": "50000.00000",
    "allocated": "50000.00000",
    "balance": "0.00000",
    "currency": { "id": 1, "code": "KES", "name": "Kenyan Shilling" },
    "status": { "value": "confirmed", "color": "success" },
    "paying_user": { "id": 77, "name": "Jane Wanjiru" },
    "receipt_allocations": [ { "id": 40, "amount": "50000.00000", "invoice": { "id": 5 } } ],
    "permissions": { "reallocate": true, "cancel": true }
  },
  "invoices": [
    {
      "id": 5,
      "lease_id": 3,
      "notes": "Rent - March 2026",
      "status": { "value": "paid", "color": "success" },
      "due_at": {
        "raw": "2026-03-01T00:00:00.000000Z",
        "formatted": "01 Mar, 2026",
        "diff": "4 months ago"
      },
      "amount": "50000.00000",
      "tax": "0.00000",
      "total": "50000.00000",
      "paid": "50000.00000",
      "balance": "0.00000",
      "currency": { "id": 1, "code": "KES", "name": "Kenyan Shilling" },
      "allocated_from_this_receipt": 50000
    },
    {
      "id": 9,
      "lease_id": 3,
      "notes": "Rent - April 2026",
      "status": { "value": "unpaid", "color": "warning" },
      "due_at": { "raw": "2026-04-01T00:00:00.000000Z", "formatted": "01 Apr, 2026", "diff": "3 months ago" },
      "amount": "50000.00000",
      "tax": "0.00000",
      "total": "50000.00000",
      "paid": "0.00000",
      "balance": "50000.00000",
      "currency": { "id": 1, "code": "KES", "name": "Kenyan Shilling" },
      "allocated_from_this_receipt": 0
    }
  ],
  "collections_frozen": false
}
```

Notes for rendering:

- `invoices` is already filtered to what this receipt may be allocated to — same property,
  same tenant, still open, or already carrying this receipt's money. **Do not filter further.**
- An invoice with `allocated_from_this_receipt > 0` and `balance: "0.00000"` is one this
  receipt paid in full. It is intentionally in the list so money can be moved *off* it.
- `allocated_from_this_receipt` is a plain number and seeds the amount input for that row.
- Money fields on the invoice are decimal **strings** (`decimal(50,5)`). Parse with
  `Number(...)` before arithmetic; never string-concatenate them.
- Sorted by `due_at`, then id. Preserve that order.
- Empty `invoices` means the tenant has no open invoices on this property — show an empty
  state ("No open invoices to reallocate to") and disable Save.

## The reallocation screen

A modal or drawer titled `Reallocate Receipt RCPT-1005`.

```text
┌────────────────────────────────────────────────────────────────────┐
│ Reallocate Receipt RCPT-1005            Jane Wanjiru · KES 50,000   │
├────────────────────────────────────────────────────────────────────┤
│ Invoice          Due          Total      Balance     Allocate       │
│ ─────────────────────────────────────────────────────────────────── │
│ Rent - Mar 2026  01 Mar 2026  50,000     0           [        0  ]  │
│ Rent - Apr 2026  01 Apr 2026  50,000     50,000      [   50,000  ]  │
│ Service - Apr    15 Apr 2026  12,000     12,000      [        0  ]  │
├────────────────────────────────────────────────────────────────────┤
│ Receipt amount  50,000    Allocated  50,000    Remaining  0         │
│                                              [ Cancel ] [ Save ]    │
└────────────────────────────────────────────────────────────────────┘
```

Required behaviour:

- **Live totals footer.** `Allocated` = sum of the inputs, `Remaining` = receipt amount −
  allocated. Show `Remaining` in danger colour whenever it is non-zero.
- **Save is disabled until `Remaining` is exactly 0.** The receipt must be fully allocated —
  see below.
- Amounts are entered in the **receipt currency** (`receipt.currency`), not the invoice
  currency. Label the column with the receipt's currency code.
- Rows left at `0` are simply omitted from the request.
- Offer an **Auto-allocate** helper: walk the invoices in the given order, filling each
  `balance` until the receipt is used up, then put any leftover on the last row touched.
- Guard against navigating away with unsaved changes.

### Frozen collections banner

When `collections_frozen` is `true`, show an information banner above the table:

> This receipt has already been remitted. Reallocating updates the invoices and the tenant
> statement, but the landlord remittance figures stay as they were remitted.

The form behaves identically otherwise — do not block the action.

### Overpayment

The receipt must be allocated in full, so where the open invoices cannot absorb it the user
must put the excess on one invoice. Allow an amount **greater** than that invoice's balance;
the backend does not clamp it. When an input exceeds its row's balance, show an inline
warning rather than an error:

> KES 5,000 more than this invoice's balance. The excess will be carried onto the tenant's
> next invoice.

## Committing

`PUT /api/v1/app/{company}/property-management/lease-management/receipts/{receipt}/reallocation`

```json
{
  "allocations": [
    { "invoice_id": 9, "amount": 30000 },
    { "invoice_id": 11, "amount": 20000 }
  ]
}
```

| Field | Required | Type | Notes |
|---|---|---|---|
| `allocations` | Yes | array | At least one entry; must total the full receipt amount |
| `allocations[].invoice_id` | Yes | integer | Must be one of the ids returned by `GET` |
| `allocations[].amount` | Yes | number | In the **receipt** currency; must be `> 0` |

Success (`200`):

```json
{
  "message": "Receipt reallocated successfully",
  "receipt": {
    "id": 12,
    "amount": "50000.00000",
    "allocated": "50000.00000",
    "balance": "0.00000",
    "status": { "value": "confirmed", "color": "success" },
    "receipt_allocations": [ "..." ],
    "component_allocations": [ "..." ]
  }
}
```

After a successful save, invalidate cached data for: the receipt list and detail, the
affected invoices, the tenant statement, and lease collections.

## Client-side validation

Mirror these so the user is not round-tripped for avoidable errors:

| Rule | Message |
|---|---|
| At least one non-zero amount | Allocate the receipt to at least one invoice. |
| Every submitted amount `> 0` | Enter an amount greater than zero. |
| Amounts total exactly the receipt amount | Allocations must add up to KES 50,000. Remaining: KES 20,000. |

Compare totals with a tolerance of `0.00001` (the stored decimal precision), not `===`, to
avoid floating-point false negatives.

The same invoice cannot appear twice — the table is one row per invoice, so this cannot
happen through the UI, but guard it if you build the payload from anything else.

## Error handling (`422`)

Standard Laravel validation envelope. Keys are `receipt` or `allocations`; render them as
form-level errors above the table rather than against a single input.

```json
{
  "message": "The allocations must add up to the full receipt amount of 50,000.00.",
  "errors": {
    "allocations": ["The allocations must add up to the full receipt amount of 50,000.00."]
  }
}
```

| Key | Cause | Suggested handling |
|---|---|---|
| `receipt` | Not confirmed, reversed, or has allocations awaiting approval | Show the message and close the modal — the receipt is no longer reallocatable. Refresh the row. |
| `allocations` | Ineligible invoice, duplicate invoice, non-positive amount, or a total that does not match | Show the message above the table and keep the form open. |

A `403` means the permission was revoked between render and submit — refresh the receipt and
hide the action.

Apply the shared auth, tenancy and error rules in `docs/frontend/app/README.md`.

## Related

- `docs/frontend/app/property-management/lease-management/receipts.md`
- `docs/frontend/app/property-management/lease-management/invoices.md`
- `docs/frontend/app/property-management/lease-management/tenant-statements.md`
