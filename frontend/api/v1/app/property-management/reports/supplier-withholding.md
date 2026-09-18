# Supplier Withholding Report API

Domain: `Property Management > Reports > Suppliers > Supplier Withholding`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Supplier Withholding report lists the **tax withheld from supplier bills for one withholding tax**,
one row per withholding line, so the tax can be reconciled and remitted. It is **read-only** and
computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/suppliers/supplier-withholding` — requires `view-supplier-withholding-report`
- `GET /reports/suppliers/supplier-withholding/export` — requires
  `export-supplier-withholding-report`. Takes a required `format` (`excel` | `pdf`) plus the same
  filters as the report (so `withholding_tax_id` is required here too); any other format, or none,
  returns `422` with a `format` error.

The view and export permissions are **independent**. There is exactly **one** report, one endpoint
pair and one permission pair — the per-tax entries in the nav are presets on it, not separate reports.

## One report, three presets

In [`GET /reports`](./README.md#discovering-reports--get-reports) `supplier-withholding` is a **report
node** under the supplier party, and it holds three **preset-filter children**. A preset is nav only:
it opens this same report with its `withholding_tax_id` pre-applied, is visible exactly when the user
holds `view-supplier-withholding-report`, and has no route or permission of its own.

| Preset key | Label | Opens |
|---|---|---|
| `supplier-withholding-vat` | Withholding VAT (2%) | `…/supplier-withholding?withholding_tax_id=1` |
| `supplier-withholding-professional-fees` | WHT Professional Fees (5%) | `…/supplier-withholding?withholding_tax_id=2` |
| `supplier-withholding-contractual` | WHT Contractual/Labour (3%) | `…/supplier-withholding?withholding_tax_id=3` |

Always read the ids from each preset's `preset_filters` in the discovery response rather than
hardcoding them. The user may change the tax (or any other filter) after opening a preset; the preset
is only a starting point. The **rate shown in the report comes from the tax itself**
(`withholding_taxes.value`), not from the preset label.

> The spec wrote the path as `…/supplier/supplier-withholding/supplier-withholding`. It is served at
> `…/suppliers/supplier-withholding`: party segments are plural, and the withholding parent is the
> report itself (a parent group and a report cannot share one key), as the
> [contract](./README.md#preset-filter-children) describes.

## Which lines count — the accrual rule

The same rule as [Input VAT Analysis](./vat-analysis-input.md): a withholding line counts once its
**bill is posted**, whether or not it has been paid. By the bill's `facility_bills.status`:

| Bill status | Included? |
|---|---|
| `pending` | **No** — not posted yet |
| `unpaid` | Yes |
| `partially-paid` | Yes |
| `paid` | Yes |
| `cancelled` | **No** |

A line falls in the period by its **bill's posting date** (`facility_bills.expense_posted_at`, or the
invoice date for bills imported without one).

## Filters

The six [global filters](./README.md#global-filters) plus:

| Param | Type | Required | Meaning when omitted |
|---|---|---|---|
| six global filters | see the shared contract | no | all / current month |
| `withholding_tax_id` | `withholding_taxes.id` | **yes** | — omitting it returns `422` with a `withholding_tax_id` error |
| `supplier_id` | `users.id` (the bill's `vendor_id`) | no | All suppliers |

`header.filters` echoes all eight.

```
GET …/reports/suppliers/supplier-withholding?withholding_tax_id=2&period_from=2026-03-01&period_to=2026-03-31
```

## Header

- `withholding_tax` — `{ id, name, rate }` of the selected tax (`rate` as a label, e.g. `"2%"`).
- `property` — `{ id, name }` when a `facility_id` is given, otherwise `null`.
- `currency` — `{ code, name }` when every listed bill is in one currency; `null` when they differ.

## Bucket and columns

One bucket, `overview` (`header.label`: `"{tax name} Withheld"`): one row per withholding line in
posting date order, then a `Total` subtotal row.

| Column | `key` | `format` | Source |
|---|---|---|---|
| Posted At | `invoice_posted_at` | string `Y-m-d` | the bill's `expense_posted_at` (or `invoice_date` for imported bills) |
| Invoice No. | `invoice_no` | string | `facility_bills.invoice_number` |
| Supplier | `supplier` | string | the supplier's name |
| Supplier PIN | `supplier_pin` | string | the supplier's `users.tax_pin` |
| Bill No. | `bill_no` | string | the bill's id |
| Gross Amount | `gross_amount` | money | the bill's `total` |
| Withholding Tax | `withholding_tax` | string | `withholding_taxes.name` |
| Rate | `rate` | string | `withholding_taxes.value` as a percentage, e.g. `"2%"` |
| Amount Withheld | `amount_withheld` | money, `grosstotal` | the line's `facility_bill_withholdings.amount` |
| Withholding Paid | `withholding_paid` | money | the line's `facility_bill_withholdings.paid`: how much has been remitted through a liability bill |

The `Total` row spans the first five columns (`col_span: 5`), then carries the sums of `gross_amount`,
`amount_withheld` and `withholding_paid`; `withholding_tax` and `rate` are omitted. It has
`type: subtotal` and `background_color: secondary`, and is the exact sum of the rows.

## Summary

| Key | Meaning |
|---|---|
| `total_gross` | Sum of `gross_amount` |
| `total_withheld` | Sum of `amount_withheld` |
| `total_paid` | Sum of `withholding_paid` |
| `line_count` | Withholding lines listed |

## No certificates

This report has **no withholding certificate functionality**. It does not generate, store, expose or
link to certificates: no field in the response refers to one, and there is no certificate download or
generation endpoint.

## Example response

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "withholding_tax": { "id": 1, "name": "VAT 2%", "rate": "2%" },
      "currency": { "code": "KES", "name": "Kenya Shilling" },
      "period": { "from": "2026-03-01", "to": "2026-03-31" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null, "properties_status": null,
                   "period_from": "2026-03-01", "period_to": "2026-03-31", "withholding_tax_id": 1, "supplier_id": null },
      "generated_at": "2026-04-05T09:00:00+03:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "VAT 2% Withheld" },
        "fields": [ /* the ten columns above, in order */ ],
        "items": [
          { "invoice_posted_at": { "value": "2026-03-10" }, "invoice_no": { "value": "GF-2026-03" },
            "supplier": { "value": "Guardforce" }, "supplier_pin": { "value": "P059876543Z" },
            "bill_no": { "value": "418" }, "gross_amount": { "value": 58000 },
            "withholding_tax": { "value": "VAT 2%" }, "rate": { "value": "2%" },
            "amount_withheld": { "value": 1160 }, "withholding_paid": { "value": 1160 },
            "type": "normal" },
          { "invoice_posted_at": { "value": "Total", "col_span": 5 },
            "gross_amount": { "value": 58000 }, "amount_withheld": { "value": 1160 }, "withholding_paid": { "value": 1160 },
            "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "total_gross": 58000, "total_withheld": 1160, "total_paid": 1160, "line_count": 1 }
      }
    ],
    "summary": { "total_gross": 58000, "total_withheld": 1160, "total_paid": 1160, "line_count": 1 }
  }
}
```
