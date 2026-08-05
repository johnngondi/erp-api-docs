# Receipt Reallocation

Moves a confirmed receipt's money between a tenant's invoices without cancelling and
re-keying the receipt. The receipt keeps its amount, transaction number and cashbook
entry; what moves is which invoices it settles.

Base prefix: `/api/v1/app/{company}`

| Method | URI | Name |
|---|---|---|
| `GET` | `.../property-management/lease-management/receipts/{receipt}/reallocation` | `app.property-management.lease-management.receipts.reallocation.index` |
| `PUT` | `.../property-management/lease-management/receipts/{receipt}/reallocation` | `app.property-management.lease-management.receipts.reallocation.store` |

Permission: `reallocate-facility-receipt` (`FacilityReceiptPolicy::reallocate`). The ability
is also exposed per-record under `permissions.reallocate` on `FacilityReceiptResource`, so
the UI can hide the action.

Frontend spec (screen, split-button placement, client-side validation):
`docs/frontend/api/v1/app/property-management/lease-management/receipt-reallocation.md`.

## GET — allocatable invoices

Returns the receipt, the invoices its money may be spread across, and how it is spread today.

```json
{
  "receipt": { "id": 12, "amount": "50000.00000", "allocated": "50000.00000", "balance": "0.00000", "...": "..." },
  "invoices": [
    {
      "id": 5,
      "lease_id": 3,
      "notes": "Rent - March 2026",
      "status": { "value": "paid", "...": "..." },
      "due_at": { "raw": "2026-03-01T00:00:00Z", "formatted": "01 Mar, 2026", "diff": "4 months ago" },
      "amount": "50000.00000",
      "tax": "0.00000",
      "total": "50000.00000",
      "paid": "50000.00000",
      "balance": "0.00000",
      "currency": { "id": 1, "code": "KES", "...": "..." },
      "allocated_from_this_receipt": 50000
    }
  ],
  "collections_frozen": false
}
```

An invoice is offered when **all** of the following hold:

- its lease is on the **same property** as the receipt (`lease.facility_id`);
- its lease belongs to the **same tenant** as the receipt (`lease.user_id == receipt.paying_user_id`);
- its status is not `pending` and not `cancelled`;
- it still has a balance (`balance > 0`) — **or** this receipt is currently allocated to it.

That last clause is what lets money be moved *off* an invoice this receipt has already paid
in full. An invoice settled by some *other* receipt has no balance and is not this receipt's,
so it never appears.

`allocated_from_this_receipt` is in the allocation (lease) currency and is `0` for invoices
the receipt has not been allocated to.

`collections_frozen` is `true` when the receipt has been remitted — see below.

## PUT — commit a new split

```json
{
  "allocations": [
    { "invoice_id": 9, "amount": 30000 },
    { "invoice_id": 11, "amount": 20000 }
  ]
}
```

`amount` is in the **receipt (payment) currency**, the same convention as receipt creation.

Returns `{ "message": "Receipt reallocated successfully", "receipt": { ... } }`.

### Validation (`422`)

| Condition | Message |
|---|---|
| Receipt is not `confirmed` | Only confirmed receipts can be reallocated. |
| Receipt has been reversed | A reversed receipt cannot be reallocated. |
| Receipt has allocations awaiting approval | This receipt has allocations awaiting approval and cannot be reallocated yet. |
| Invoice fails the eligibility rules above | Invoice #N cannot be allocated to this receipt… |
| Same invoice listed twice | Invoice #N is listed more than once. |
| An amount is `<= 0` | Allocation amount for invoice #N must be greater than zero. |
| The split does not equal the receipt amount | The allocations must add up to the full receipt amount of X. |

### Full allocation and overpayment

The split must consume the **whole** receipt (tolerance `0.00001`, one unit in the last
stored decimal place). A receipt is never left partly parked.

Where the tenant's open invoices cannot absorb the whole receipt, allocate the excess to one
invoice: the allocation is **not** clamped to the invoice balance, so that invoice goes to a
negative balance (an overpayment). `ApplyLeaseOverpaymentAction`, which already runs on every
invoice generation path via `ProcessInvoiceAction`, then carries that overpayment onto the
tenant's next invoice automatically.

## What moves, and what does not

| Artifact | Effect |
|---|---|
| Cashbook (`BankAccountTransaction`) | **Never touched.** The money received into the bank account has not changed. |
| Receipt (`amount`, `transaction_number`, status) | Unchanged. `allocated` becomes the full amount, `balance` becomes `0`. |
| `FacilityReceiptAllocation` | Old rows removed, new rows created from the submitted split. |
| `FacilityInvoice` `paid` / `balance` / `status` | Reversed on the old invoices, applied to the new ones. |
| `FacilityInvoiceItem` `paid` / `balance` | Same, split by the facility's `lease_component_allocation_priority_method`. |
| `LeaseCollection` | Deleted and rebuilt from the new split — unless the receipt is remitted. |
| `TenantStatement` | The receipt's rows are deleted and re-posted, one confirmed credit per new allocation. |

## Remitted receipts — collections are frozen

When the receipt is already attached to a `FacilityRemittance`, its lease collections are
**left exactly as they are** and `collections_frozen` is `true`.

The reason is structural: `ComputeRemittableAmountService` excludes collections with
`whereDoesntHave('receiptAllocation.receipt.remittances')`, which is a *receipt-level*
filter. Once a receipt is in `facility_receipt_remittance`, every collection beneath it —
past, present or future — is permanently invisible to remittance, so reversing collections
posted there would never net out. The money has also already been paid out to the landlord
against those specific components.

In this mode:

- the old allocation rows are **kept** and set to `cancelled` (they cannot be deleted:
  `lease_collections.receipt_allocation_id` has no cascade, so removing them would orphan
  the collections);
- the new allocation rows are created with **no** collections and **no** component-allocation
  breakdown;
- invoice and invoice-item balances still move, and the tenant statement is still rebuilt.

Net effect: the tenant sees the reallocation and the receipt amount is unchanged, while
remittance economics stay exactly as they were remitted.

## Implementation

| Concern | Class |
|---|---|
| Endpoints | `App\Http\Controllers\Api\V1\App\PropertyManagement\LeaseManagement\Receipting\ReceiptReallocationController` |
| Orchestration + rules | `App\Services\PropertyManagement\Billing\Receipt\ReceiptReallocationService` |
| Applying one allocation (shared with the approval path) | `App\Actions\PropertyManagement\LeaseManagement\Receipt\ApplyReceiptAllocationAction` |
| Unwinding normally | `App\Actions\PropertyManagement\LeaseManagement\Receipt\DeleteReceiptAllocationAction` |
| Unwinding a remitted receipt | `App\Actions\PropertyManagement\LeaseManagement\Receipt\ReverseReceiptAllocationBalancesAction` |
| Request payloads | `App\Data\ReceiptReallocationData`, `App\Data\ReceiptReallocationItemData` |
| Tests | `tests/Feature/ReceiptReallocationTest.php` |

`ApplyReceiptAllocationAction` is the single place the per-allocation money math lives;
`ProcessReceiptAction` (the approval path) delegates to it too, so both paths convert
currencies and split components identically.
