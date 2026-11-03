# Document dates: `requested_at` and `created_at`

Applies to **receipts, invoices, credit notes and LPOs**.

Each of these carries two dates:

| Field | Meaning | Changes? |
|---|---|---|
| `requested_at` | When the document was first entered. | Never. |
| `created_at` | When the document was **processed**: the day of its final approval, or the moment it was entered if it needs no approval. | Once, on approval. Receipts and invoices can also be moved with Change Date (backdate). |

A receipt captured on 30 Oct and approved on 2 Nov reads `requested_at` 30 Oct and `created_at`
2 Nov, and falls in November wherever the list or a report goes by `created_at`. Its statement
lines, collections and cashbook entry are written on approval and carry the same day.

Both come in the usual shape, on the list and the detail of all four:

```json
"requested_at": { "raw": "2026-10-30T07:15:00.000000Z", "formatted": "30 Oct, 2026", "diff": "3 days ago" },
"created_at":   { "raw": "2026-11-02T11:40:00.000000Z", "formatted": "02 Nov, 2026", "diff": "8 hours ago" }
```

The LPO resource names its processing date `created`, as before.

## Filter

`filter[requested_between]=2026-10-01,2026-10-31`, the same format as `filter[created_between]`:
two `Y-m-d` dates in ascending order, both days included.

| List | Endpoint |
|---|---|
| Receipts | `GET /property-management/lease-management/receipts` |
| Invoices | `GET /property-management/lease-management/invoices` |
| Credit notes | `GET /property-management/lease-management/credit-notes` |
| LPOs | `GET /property-management/procurement/lpos` |

## Older records

Records entered before 2026-10-27 have no stored `requested_at`. They return their `created_at`
there, and `requested_between` matches them by it. That is their entry date, because their
`created_at` was never moved.

One of them still pending on that day is approved like any other: its `created_at` moves to the
approval day, and its entry date is stored in `requested_at` at that moment, so the two differ as
they do on a newer record. Change Date on an older receipt stores it the same way.

## What moves with `created_at`

- **Receipts:** the receipts list, sorting and Change Date. The payment date (`transaction_date`)
  and the cashbook's `transaction_at` stay on the day the tenant paid.
- **Invoices and credit notes:** the VAT output report period, the ageing report, and the "issued
  at" date on the document and its ETR signature. An invoice raised 30 Oct and approved 2 Nov is in
  November's VAT and ages from 2 Nov. An invoice or credit note with its own `transaction_at` (a
  backdated one) keeps its `created_at` on approval.
- **LPOs:** the "issued at" date on the LPO document and the dashboard counts. An LPO is placed on
  the day it is approved.

Running an already approved receipt again, for example to allocate more of it, does not move its
`created_at`.
