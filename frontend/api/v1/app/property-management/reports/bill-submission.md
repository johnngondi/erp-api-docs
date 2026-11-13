# Bill Submission Report API

Domain: `Property Management > Reports > Suppliers > Expenditure Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Bill Submission report lists **every supplier bill submitted in the period**, one row per bill,
so you can see what suppliers have billed, when each bill was posted to expenses and which bills are
still waiting. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/suppliers/expenditure-reports/bill-submission` — requires `view-bill-submission-report`
- `GET /reports/suppliers/expenditure-reports/bill-submission/export` — requires
  `export-bill-submission-report`. Takes a required `format` (`excel` | `pdf`) plus the same filters as
  the report; any other format, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports) under the supplier
  `expenditure-reports` group.

The view and export permissions are **independent**, and neither is shared with
[Bill Payment](./bill-payment.md).

## What is listed

- **The period is the Bill Posted Date range**: a bill is listed when `facility_bills.expense_posted_at`
  falls in `period_from`–`period_to`, and only then. A bill **not yet posted** to expenses (typically
  `pending`) has no posted date and is **never listed**, however recently it was submitted. The invoice
  date is never read.
- **`created_from` / `created_to` narrow by Bill Created Date** (`facility_bills.created_at`, when the
  bill was captured). Either end may be sent alone; omit both for any submission date.
- Every bill type and every status is listed unless `bill_status` narrows it.
- **Status is `facility_bills.status` itself** — `pending`, `unpaid`, `partially-paid`, `paid` or
  `cancelled`. It is never derived from the approval workflow.
- Figures are **as stored on the bill, in the bill's own currency**. Nothing is converted.

## Filters

The [global filters](./README.md#global-filters) plus:

| Param | Type | Meaning when omitted |
|---|---|---|
| [global filters](./README.md#global-filters) — `landlord_ids`, `facility_ids`, `facility_type_id`, `properties_status` | see the shared contract | all |
| `period_from` / `period_to` | date `Y-m-d` — **Bill Posted Date**: the day the bill was posted to expenses (`facility_bills.expense_posted_at`); an unposted bill is never listed | Current month |
| `created_from` / `created_to` | date `Y-m-d` — **Bill Created Date**: the day the bill was captured (`facility_bills.created_at`). Either end may be sent alone. | Any submission date |
| `bill_status` | `pending` \| `unpaid` \| `partially-paid` \| `paid` \| `cancelled` \| `reversed` \| `payable` — `payable` is a shorthand for `unpaid` and `partially-paid` together (the bills still owed), never a stored status | Any status |
| `supplier_id` | `users.id` (the bill's `vendor_id`) | All suppliers |
| `buckets` | list of this report's bucket ids | Every bucket — see [Selecting buckets](./README.md#selecting-buckets--buckets) |

`header.filters` echoes every filter, `created_from` and `created_to` included. `header.period` is the
Bill Posted Date range.

```
GET …/reports/suppliers/expenditure-reports/bill-submission?period_from=2026-01-01&period_to=2026-06-30&created_from=2026-03-01&created_to=2026-03-31&bill_status=pending
```

## Header

- `property` — `{ id, name }` when exactly one property is selected, otherwise `null`. The full
  selection is in `properties` / `landlords` (see the [shared `header`](./README.md#header)).
- `currency` — `{ code, name }` when every listed bill is in one currency; `null` when they differ
  (the totals row then mixes currencies, so show it with care).

## Bucket

One bucket, `overview` (`header.label`: `Bills Submitted`): one row per bill, then a closing
`grosstotal` row.

By default the rows are ordered by supplier, then property, then expense posted date
(`header.sort.default` `["supplier", "property", "billed_date"]`). Ties keep the report's own order,
oldest submission first. Re-sort
with [`sort[overview]`](./README.md#sorting-rows--sort), for example `sort[overview]=billed_date` for
the earliest postings first. The closing row stays last. [`columns`](./README.md#choosing-columns--columns) picks the columns shown and exported.

## Columns

These nine columns are **shared with [Bill Payment](./bill-payment.md)**: same keys, same order, same
formats. Two of them read differently here — `expense_type` is labelled **Service**, and
`billed_date` is labelled **Expense Posted Date** and carries the date the bill was posted to expenses
rather than its invoice date. Keys are unchanged, so saved column and sort preferences carry over.

| Column | `key` | `format` | Source |
|---|---|---|---|
| Invoice No. | `invoice_no` | string | `facility_bills.invoice_number` |
| Service | `expense_type` | string | the bill's expense type name |
| Supplier | `supplier` | string | the bill's vendor name |
| Property | `property` | string | the bill's property name |
| Expense Posted Date | `billed_date` | string `Y-m-d` | `facility_bills.expense_posted_at` (always set, since only posted bills are listed) |
| Amount | `amount` | money | `facility_bills.amount` |
| Tax | `tax` | money | `facility_bills.tax` |
| Total | `total` | money | `facility_bills.total` |
| Status | `status` | string | `facility_bills.status`; the cell carries the status colour (`pending` warning, `unpaid` danger, `partially-paid` info, `paid` success, `cancelled` secondary) |

The closing row is a `Total` label with `col_span: 5` (covering the five text columns), then the
`amount` / `tax` / `total` sums; `status` is omitted. It has `type: grosstotal`,
`background_color: secondary`.

## Overdue tint

A row gets **`background_color: warning`** when **both** hold:

- more than **90 days** have passed since the bill's `created_at` (counted to today), and
- its status is still **`pending`**.

A bill in any other status is never tinted, however old it is. Since only posted bills are listed and
posting normally takes a bill out of `pending`, the tint applies only to a bill that still reads
`pending` despite carrying a posted date.

## Summary

| Key | Meaning |
|---|---|
| `total_amount` | Sum of `amount` |
| `total_tax` | Sum of `tax` |
| `total` | Sum of `total` |
| `bill_count` | Bills listed |
| `overdue_pending_count` | Rows tinted warning |

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "currency": { "code": "KES", "name": "Kenya Shilling" },
      "period": { "from": "2026-01-01", "to": "2026-06-30" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null, "properties_status": null,
                   "period_from": "2026-01-01", "period_to": "2026-06-30", "bill_status": null, "supplier_id": null,
                   "created_from": null, "created_to": null },
      "generated_at": "2026-06-30T12:00:00+03:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "Bills Submitted" },
        "fields": [ /* the nine columns above, in order; expense_type labelled "Service", billed_date "Expense Posted Date" */ ],
        "items": [
          { "invoice_no": { "value": "INV-0142" }, "expense_type": { "value": "Plumbing" }, "supplier": { "value": "AquaFix Ltd" },
            "property": { "value": "ACK Gardens" }, "billed_date": { "value": "2026-01-28" },
            "amount": { "value": 1000 }, "tax": { "value": 160 }, "total": { "value": 1160 },
            "status": { "value": "partially-paid", "color": "info" },
            "type": "normal" },
          { "invoice_no": { "value": "INV-0187" }, "expense_type": { "value": "Security" }, "supplier": { "value": "Guardforce" },
            "property": { "value": "ACK Gardens" }, "billed_date": { "value": "2026-06-02" },
            "amount": { "value": 5000 }, "tax": { "value": 800 }, "total": { "value": 5800 },
            "status": { "value": "unpaid", "color": "danger" },
            "type": "normal" },
          { "invoice_no": { "value": "Total", "col_span": 5 }, "amount": { "value": 6000 }, "tax": { "value": 960 }, "total": { "value": 6960 },
            "type": "grosstotal", "background_color": "secondary" }
        ],
        "summary": { "total_amount": 6000, "total_tax": 960, "total": 6960, "bill_count": 2, "overdue_pending_count": 0 }
      }
    ],
    "summary": { "total_amount": 6000, "total_tax": 960, "total": 6960, "bill_count": 2, "overdue_pending_count": 0 }
  }
}
```
