# Tenant Dashboard API

Base route:

`/api/v1/tenant`

## Endpoint

- `GET /api/v1/tenant`

Every figure is for one of the signed-in tenant's leases. A tenant holding several leases switches
between them with `lease_id`; the money figures are never added across leases, because each lease
has its own currency and its own statement.

## Query Parameters

| Param | Type | Required | Description |
|---|---|---|---|
| `lease_id` | integer | No | The lease to show. Defaults to the tenant's newest active lease, or their newest lease when none is active. A lease that is not the tenant's is a `404`. |

## Response

Example response:

```json
{
  "data": {
    "user": { "id": 88, "name": "Jane Wanjiku" },
    "leases": [
      {
        "id": 12,
        "label": "Westlands Plaza - Shop A1, Shop A2",
        "status": { "value": "active", "color": "success" }
      }
    ],
    "lease": {
      "id": 12,
      "facility": { "id": 3, "name": "Westlands Plaza" },
      "spaces": ["Shop A1", "Shop A2"],
      "status": { "value": "active", "color": "success" },
      "start_at": "2024-01-01",
      "end_at": "2026-12-31",
      "billing_cycle": "monthly",
      "next_due_at": "2026-11-01",
      "currency": { "id": 1, "code": "KES" },
      "monthly_rent": 45000,
      "monthly_charges": 52000,
      "deposits_held": 90000
    },
    "account": {
      "period": { "from": "2026-10-01", "to": "2026-10-31" },
      "billed": 52000,
      "paid": 20000,
      "credited": 0,
      "balance": 32000,
      "unpaid_invoices": 2,
      "next_due": { "date": "2026-10-05", "days": -1, "overdue": true },
      "trend": [
        { "label": "May 2026", "billed": 52000, "paid": 52000, "credited": 0, "balance": 0 },
        { "label": "Jun 2026", "billed": 52000, "paid": 52000, "credited": 0, "balance": 0 },
        { "label": "Jul 2026", "billed": 52000, "paid": 40000, "credited": 0, "balance": 12000 },
        { "label": "Aug 2026", "billed": 52000, "paid": 64000, "credited": 0, "balance": 0 },
        { "label": "Sep 2026", "billed": 52000, "paid": 52000, "credited": 0, "balance": 0 },
        { "label": "Oct 2026", "billed": 52000, "paid": 20000, "credited": 0, "balance": 32000 }
      ]
    },
    "utilities": {
      "period": { "from": "2026-09-01", "to": "2026-09-30" },
      "total": 4700,
      "items": [
        { "name": "Electricity", "amount": 3200 },
        { "name": "Water", "amount": 1500 }
      ]
    },
    "stats": {
      "open_tickets": 1,
      "documents": 3
    },
    "recent_payments": [
      {
        "id": 905,
        "transaction_number": "QBK7X2M9P4",
        "transaction_date": "2026-10-03",
        "amount": 20000,
        "currency": { "id": 1, "code": "KES" },
        "payment_method": { "id": 2, "name": "M-Pesa" },
        "status": { "value": "confirmed", "color": "success" }
      }
    ],
    "permissions": ["view-facility-invoice"]
  }
}
```

A tenant with no lease gets `leases: []`, and `lease`, `account` and `utilities` are `null`;
`recent_payments` is empty and `stats.documents` is `0`.

## Field Rules

| Field | Meaning |
|---|---|
| `leases` | All the tenant's leases, newest first, for the lease switcher. `label` is the property name and the leased spaces. |
| `lease.monthly_rent` | Sum of `cost_per_month` on the lease's rent components (`is_rent`). Before tax. |
| `lease.monthly_charges` | Sum of `cost_per_month` on every component except deposits (`is_deposit`, `is_legal_fees_deposit`). Before tax. |
| `lease.deposits_held` | Sum of the lease's deposits not yet refunded. |
| `account` | Read from the tenant statement, so it agrees with the statement page. |
| `account.period` | The current calendar month. |
| `account.billed` | Invoices dated in the period, net of any cancelled in the period. |
| `account.paid` | Receipts dated in the period, net of any cancelled in the period. |
| `account.credited` | Credit notes dated in the period, net of any cancelled in the period. |
| `account.balance` | The lease's statement balance (every line on the statement): what the tenant owes, or a negative amount held in credit. |
| `account.unpaid_invoices` | Invoices in `unpaid` or `partially paid` with a balance left. |
| `account.next_due` | The due date of the oldest unpaid invoice. `days` counts from today and is negative once the date has passed; `overdue` is `true` then. `null` when no invoice is unpaid. |
| `account.trend` | The last six calendar months, the current one included: `billed`, `paid` and `credited` as above for each month, and `balance` as the statement balance at the month's end. |
| `utilities` | Utility lines on the lease's invoices in the most recent month that had any, by component. `period` is `null` and `items` empty when the lease has never been billed for utilities. Cancelled invoices are left out. |
| `stats.open_tickets` | Tickets the tenant raised that are still `open`. Covers all their leases. |
| `stats.documents` | Files on the lease's Documents tab, counted the same way as `GET /api/v1/tenant/leases/{lease}/documents`. |
| `recent_payments` | The five newest receipts allocated to the lease's invoices. `amount` is the whole receipt. |
| `permissions` | The tenant's dashboard permissions, cached by the frontend. |

Money is in currency units, in the lease currency.
