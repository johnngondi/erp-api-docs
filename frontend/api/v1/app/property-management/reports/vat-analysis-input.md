# Input VAT Analysis Report API

Domain: `Property Management > Reports > Suppliers`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Input VAT Analysis report lists the **input VAT on supplier bills invoiced in the period**, one row
per bill, in the layout used to prepare the VAT return. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/suppliers/vat-analysis-input` — requires `view-vat-analysis-input-report`
- `GET /reports/suppliers/vat-analysis-input/export` — requires `export-vat-analysis-input-report`.
  Takes a required `format` (`excel` | `pdf`) plus the same filters as the report; any other format,
  or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports) as a top-level supplier
  report.

The view and export permissions are **independent**.

## Which bills count — the accrual rule

Input VAT is claimed when a bill is **posted**, whether or not it has been paid. A bill's status is
`facility_bills.status`:

| Status | Included? | Why |
|---|---|---|
| `pending` | **No** | Not posted yet (still awaiting approval) |
| `unpaid` | Yes | Posted |
| `partially-paid` | Yes | Posted |
| `paid` | Yes | Posted |
| `cancelled` | **No** | Never a liability |

A bill **moved from `pending` to `unpaid` appears** in the report from then on, in the period of its
invoice date.

- **Period = invoice date.** A bill falls in `period_from`–`period_to` by `facility_bills.invoice_date`,
  not by when it was captured or paid.
- **Bills only.** Input VAT is read from Facility Bills and their lines. There is no path for expenses
  recorded without a bill, so they never appear here.
- Every bill type is included; a credit note carries negative figures and reduces the totals.

## Filters

The six [global filters](./README.md#global-filters) plus:

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `tax_id` | `taxes.id` | All taxes |
| `supplier_id` | `users.id` (the bill's `vendor_id`) | All suppliers |

- **`tax_id`** keeps bills with **at least one line** (`facility_bill_items.tax_id`) on that tax. The
  row still shows the **whole bill's** figures, so a bill with lines on two taxes appears under either
  one with its full amounts.

`header.filters` echoes all eight.

```
GET …/reports/suppliers/vat-analysis-input?period_from=2026-03-01&period_to=2026-03-31
GET …/reports/suppliers/vat-analysis-input?period_from=2026-03-01&period_to=2026-03-31&tax_id=1&supplier_id=42
```

## Header

- `property` — `{ id, name }` when a `facility_id` is given, otherwise `null`.
- `currency` — `{ code, name }` when every listed bill is in one currency; `null` when they differ.
  Nothing is converted.

## Bucket and columns

One bucket, `overview` (`header.label`: `Input VAT`): one row per bill in invoice date order, then a
`Total` subtotal row.

| Column | `key` | `format` | Source |
|---|---|---|---|
| Supplier PIN | `supplier_pin` | string | the supplier's `users.tax_pin` |
| Supplier | `supplier` | string | the supplier's name |
| Invoice Date | `invoice_billed_at` | string `Y-m-d` | `facility_bills.invoice_date` |
| Invoice No. | `invoice_no` | string | `facility_bills.invoice_number` |
| CU Invoice No. | `cu_invoice_no` | string | `facility_bills.tax_invoice_number` (the control unit invoice number) |
| Description | `description` | string | the bill's line titles (`facility_bill_items.title`), comma-separated; the bill's `notes` when no line has a title. `togglable` |
| Taxable Value (Nett) | `tax_value_nett` | money | `facility_bills.amount` (net of VAT) |
| VAT | `vat` | money | `facility_bills.tax` |
| Amount | `amount` | money, `grosstotal` | `facility_bills.total` |

The closing row is a `Total` label with `col_span: 6` (covering the six text columns), then the sums of
`tax_value_nett`, `vat` and `amount`. It has `type: subtotal` and `background_color: secondary`. **It is
the exact sum of the rows above it.**

## Summary

The same object on the bucket and on `data.summary`:

| Key | Meaning |
|---|---|
| `total_nett` | Sum of `tax_value_nett` |
| `total_vat` | Sum of `vat`: the input VAT to claim |
| `total_amount` | Sum of `amount` |
| `bill_count` | Bills listed |

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "currency": { "code": "KES", "name": "Kenya Shilling" },
      "period": { "from": "2026-03-01", "to": "2026-03-31" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null, "properties_status": null,
                   "period_from": "2026-03-01", "period_to": "2026-03-31", "tax_id": null, "supplier_id": null },
      "generated_at": "2026-04-05T09:00:00+03:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "Input VAT" },
        "fields": [ /* the nine columns above, in order */ ],
        "items": [
          { "supplier_pin": { "value": "P051234567X" }, "supplier": { "value": "AquaFix Ltd" },
            "invoice_billed_at": { "value": "2026-03-10" }, "invoice_no": { "value": "INV-77" },
            "cu_invoice_no": { "value": "KRA-CU-9981" }, "description": { "value": "Pipe repair, Valve" },
            "tax_value_nett": { "value": 1000 }, "vat": { "value": 160 }, "amount": { "value": 1160 },
            "type": "normal" },
          { "supplier_pin": { "value": "P059876543Z" }, "supplier": { "value": "Guardforce" },
            "invoice_billed_at": { "value": "2026-03-31" }, "invoice_no": { "value": "GF-2026-03" },
            "cu_invoice_no": { "value": "KRA-CU-1204" }, "description": { "value": "Security services March" },
            "tax_value_nett": { "value": 50000 }, "vat": { "value": 8000 }, "amount": { "value": 58000 },
            "type": "normal" },
          { "supplier_pin": { "value": "Total", "col_span": 6 },
            "tax_value_nett": { "value": 51000 }, "vat": { "value": 8160 }, "amount": { "value": 59160 },
            "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "total_nett": 51000, "total_vat": 8160, "total_amount": 59160, "bill_count": 2 }
      }
    ],
    "summary": { "total_nett": 51000, "total_vat": 8160, "total_amount": 59160, "bill_count": 2 }
  }
}
```
