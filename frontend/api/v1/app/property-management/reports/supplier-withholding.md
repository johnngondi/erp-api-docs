# Supplier Withholding Report API

Domain: `Property Management > Reports > Suppliers > Supplier Withholding`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Supplier Withholding report shows the **tax withheld from supplier bills**: totals per property
per withholding tax, then every withholding line, then the lines of each property. It covers **every
withholding tax** unless you narrow it to one. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/suppliers/supplier-withholding` — requires `view-supplier-withholding-report`
- `GET /reports/suppliers/supplier-withholding/export` — requires
  `export-supplier-withholding-report`. Takes a required `format` (`excel` | `pdf`) plus the same
  filters as the report; any other format, or none, returns `422` with a `format` error.

The view and export permissions are **independent**. There is exactly **one** report, one endpoint pair
and one permission pair.

## Withholding taxes are data, not nav entries

The report is not split into a nav entry per tax. Withholding taxes are rows in `withholding_taxes`, and
the report reads whatever taxes the withholding lines carry:

- **Without `withholding_tax_id`** it covers every tax — each tax with lines in the period gets its own
  column in `all-properties`, and its lines appear in `overview` and the property buckets.
- **With `withholding_tax_id`** every bucket narrows to that one tax.

To offer a per-tax view, list the taxes from the withholding taxes endpoint and pass the chosen `id` as
`withholding_tax_id`. **A withholding tax added later appears automatically** — nothing in the report,
the registry or the nav needs to change.

> The spec wrote the path as `…/supplier/supplier-withholding/supplier-withholding`. It is served at
> `…/suppliers/supplier-withholding`: party segments are plural, and the `supplier-withholding` nav
> entry is the report itself (a parent group and a report cannot share one key).

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

| Param | Type | Meaning when omitted |
|---|---|---|
| six global filters | see the shared contract | all / current month |
| `withholding_tax_id` | `withholding_taxes.id` | **All withholding taxes** |
| `supplier_id` | `users.id` (the bill's `vendor_id`) | All suppliers |

`header.filters` echoes all eight.

```
GET …/reports/suppliers/supplier-withholding?period_from=2026-03-01&period_to=2026-03-31
GET …/reports/suppliers/supplier-withholding?period_from=2026-03-01&period_to=2026-03-31&withholding_tax_id=2
```

## Header

- `withholding_tax` — `{ id, name, rate }` of the selected tax when `withholding_tax_id` is given
  (`rate` as a label, e.g. `"2%"`), otherwise `null`.
- `property` — `{ id, name }` when a `facility_id` is given, otherwise `null`.
- `currency` — `{ code, name }` when every listed bill is in one currency; `null` when they differ.

## Buckets

Always in this order:

| `bucket` | When | `header.label` | Rows |
|---|---|---|---|
| `all-properties` | always, first | `By Property` | one per property with lines, sorted by name, then a `Total` row |
| `overview` | always, second | `All Withholdings`, or `{tax name} Withheld` when one tax is selected | every withholding line, in posting date order, then a `Total` row |
| `property-{id}` | one per property **with lines**, sorted by name | the property's name | that property's lines only, like `overview`, then its own `Total` row |

A property bucket's `header` also carries `property { id, name }`; use it rather than parsing the
`bucket` id. A property with no withholding lines in the period appears in no bucket.

### `all-properties` — properties × withholding taxes

| Column | `key` | `format` |
|---|---|---|
| Property | `property` | string |
| *(one per withholding tax)* the tax's name | `tax_{id}` | money — amount withheld under that tax |
| Total | `total` | money, `grosstotal` — the row's tax cells added up |

- **Tax columns are generated at runtime**: one for each withholding tax that has lines in the period
  (after filters), ordered by name, placed between `property` and `total`. Each is `togglable: true`
  and carries `withholding_tax_id`. **Read the columns from `fields`**; never assume which taxes or how
  many.
- **Missing values are omitted, not zero.** A property with nothing withheld under a tax leaves that
  `tax_{id}` key out of its row — render an absent key as an empty cell. `total` is always present.
- The closing `Total` row (`type: subtotal`, `background_color: secondary`) sums every property per tax
  and overall, and equals `overview`'s total withheld.

Its `summary`: `total_withheld`, `property_count`, `tax_count`.

### `overview` and `property-{id}` — withholding lines

| Column | `key` | `format` | Source |
|---|---|---|---|
| Posted At | `invoice_posted_at` | string `Y-m-d` | the bill's `expense_posted_at` (or `invoice_date` for imported bills) |
| Invoice No. | `invoice_no` | string | `facility_bills.invoice_number` |
| Supplier | `supplier` | string | the supplier's name |
| Supplier PIN | `supplier_pin` | string | the supplier's `users.tax_pin` |
| Bill No. | `bill_no` | string | the bill's id |
| Gross Amount | `gross_amount` | money | the bill's `total` |
| Withholding Tax | `withholding_tax` | string | the line's tax: `withholding_taxes.name` |
| Rate | `rate` | string | the line's tax: `withholding_taxes.value` as a percentage, e.g. `"2%"` |
| Amount Withheld | `amount_withheld` | money, `grosstotal` | the line's `facility_bill_withholdings.amount` |
| Withholding Paid | `withholding_paid` | money | the line's `facility_bill_withholdings.paid`: how much has been remitted through a liability bill |

Rows can be of different taxes, so the tax name and rate are **per row**. The `Total` row spans the
first five columns (`col_span: 5`), then carries the sums of `gross_amount`, `amount_withheld` and
`withholding_paid`; `withholding_tax` and `rate` are omitted. It has `type: subtotal` and
`background_color: secondary`, and is the exact sum of the rows in that bucket.

A bill withheld under two taxes has two lines, so its gross appears on both — `gross_amount` totals
across taxes count such a bill once per line.

## Summary

`overview`, each property bucket and `data.summary` carry:

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

All taxes, March 2026:

```jsonc
{
  "data": {
    "header": {
      "property": null,
      "withholding_tax": null,
      "currency": { "code": "KES", "name": "Kenya Shilling" },
      "period": { "from": "2026-03-01", "to": "2026-03-31" },
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null, "properties_status": null,
                   "period_from": "2026-03-01", "period_to": "2026-03-31", "withholding_tax_id": null, "supplier_id": null },
      "generated_at": "2026-04-05T09:00:00+03:00"
    },
    "report": [
      {
        "bucket": "all-properties",
        "header": { "label": "By Property" },
        "fields": [
          { "label": "Property", "key": "property", "format": "string", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "left", "visible": true, "togglable": false },
          { "label": "Prof 5%", "key": "tax_2", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "withholding_tax_id": 2 },
          { "label": "VAT 2%", "key": "tax_1", "format": "money", "type": "normal", "weight": "font-normal", "background_color": "none", "alignment": "right", "visible": true, "togglable": true, "withholding_tax_id": 1 },
          { "label": "Total", "key": "total", "format": "money", "type": "grosstotal", "weight": "font-bold", "background_color": "none", "alignment": "right", "visible": true, "togglable": false }
        ],
        "items": [
          { "property": { "value": "ACK Gardens" }, "tax_2": { "value": 2500 }, "tax_1": { "value": 1160 }, "total": { "value": 3660 }, "type": "normal" },
          { "property": { "value": "Beta House" }, "tax_1": { "value": 400 }, "total": { "value": 400 }, "type": "normal" },
          { "property": { "value": "Total" }, "tax_2": { "value": 2500 }, "tax_1": { "value": 1560 }, "total": { "value": 4060 }, "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "total_withheld": 4060, "property_count": 2, "tax_count": 2 }
      },
      {
        "bucket": "overview",
        "header": { "label": "All Withholdings" },
        "fields": [ /* the ten line columns above, in order */ ],
        "items": [
          { "invoice_posted_at": { "value": "2026-03-10" }, "invoice_no": { "value": "GF-2026-03" },
            "supplier": { "value": "Guardforce" }, "supplier_pin": { "value": "P059876543Z" },
            "bill_no": { "value": "418" }, "gross_amount": { "value": 58000 },
            "withholding_tax": { "value": "VAT 2%" }, "rate": { "value": "2%" },
            "amount_withheld": { "value": 1160 }, "withholding_paid": { "value": 1160 },
            "type": "normal" },
          /* …the other lines… */
          { "invoice_posted_at": { "value": "Total", "col_span": 5 },
            "gross_amount": { "value": 128000 }, "amount_withheld": { "value": 4060 }, "withholding_paid": { "value": 1160 },
            "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "total_gross": 128000, "total_withheld": 4060, "total_paid": 1160, "line_count": 3 }
      },
      {
        "bucket": "property-3",
        "header": { "label": "ACK Gardens", "property": { "id": 3, "name": "ACK Gardens" } },
        "fields": [ /* same as overview */ ],
        "items": [ /* ACK Gardens' lines, then its Total row */ ],
        "summary": { "total_gross": 116000, "total_withheld": 3660, "total_paid": 1160, "line_count": 2 }
      }
      /* …then property-{id} for Beta House… */
    ],
    "summary": { "total_gross": 128000, "total_withheld": 4060, "total_paid": 1160, "line_count": 3 }
  }
}
```
