# Lease Charges Imports API

> **New meter-reading-based utility billing** (in transition, coexists with the
> spreadsheet import below): see [utility-billing.md](./utility-billing.md).

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/charges`

## Endpoints

- `POST /charges/import-validate`
- `POST /charges/import-process`

## Charge rate for utility charges

When the property's utility distribution method is not `defined`, the user types in three
figures from the provider's bill and the UI derives the rates before validating:

- Inputs: **total bill amount including tax**, the **tax amount on the bill** (suggested
  from the utility VAT rate until overwritten) and the **total billed consumption**.
- **Rate with utility VAT** = total ÷ consumption.
- **Charge rate without utility VAT** = (total − tax) ÷ consumption.

Both rates are shown and both are editable; editing one moves the other by the bill's own
ratio. The charge rate without VAT is what is sent as `charge_rate`. The import tax is then
added back on every invoice line, so tenants are billed the provider's gross rate. A bill
with no tax has a tax amount of 0 and the two rates are equal.

When the distribution method is `defined`, `charge_rate` is entered by hand.

## Validate Import File

`POST /api/v1/app/{company}/property-management/lease-management/charges/import-validate`

Content type:

- `multipart/form-data`

Form-data fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `charge_id` | Yes | integer | Must exist in `lease_components.id` and be a charge component |
| `import_file` | Yes | file | `.xlsx`, `.xls`, or `.csv` |
| `tax_id` | Yes | integer | Tax applied to every generated invoice line; the preview `tax` and `total` are computed from it |
| `charge_rate` | No | number | Required for utility charges and must be `> 0` |

Supported file headers:

- Land fee style: `lease_id`, `amount`
- Utility style: `lease_id`, `previous_reading`, `current_reading`

Validation behavior:

- Invalid rows are returned with `has_error: true`.
- Rows with recoverable issues can return `has_warning: true`.
- Utility charge amount formula:
  - `(current_reading - previous_reading) * charge_rate`
- Each valid row carries `tax = amount × tax rate` and `total = amount + tax` for the
  selected `tax_id`. The preview is computed server-side, so the UI re-runs this call
  whenever the tax selection changes.
- `summary` echoes the inputs the preview was built with (`charge_rate`, `tax_id`,
  `tax_rate`) alongside the row counts and money totals of the valid rows.

Example response:

```json
{
  "message": "Import file validated.",
  "rows": [
    {
      "lease_id": 101,
      "user": { "id": 77, "name": "Jane Tenant" },
      "property": { "id": 3, "name": "Riverside Towers" },
      "currency": { "id": 1, "code": "KES", "name": "Kenyan Shilling" },
      "consumed": null,
      "amount": 1250,
      "tax": 200,
      "total": 1450,
      "has_warning": false,
      "warning": null,
      "has_error": false,
      "error": null
    },
    {
      "lease_id": 999999,
      "user": null,
      "property": null,
      "currency": null,
      "consumed": null,
      "amount": null,
      "tax": null,
      "total": null,
      "has_warning": false,
      "warning": null,
      "has_error": true,
      "error": "Lease ID 999999 does not exist."
    }
  ],
  "summary": {
    "total_rows": 2,
    "valid_rows": 1,
    "error_rows": 1,
    "amount": 1250,
    "tax": 200,
    "total": 1450,
    "charge_rate": null,
    "tax_id": 1,
    "tax_rate": 0.16
  }
}
```

## Process Import

`POST /api/v1/app/{company}/property-management/lease-management/charges/import-process`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `charge_id` | Yes | integer | Must exist in `lease_components.id` and be a charge component |
| `tax_id` | Yes | integer | Tax to apply to generated invoice items; use the same tax the rows were validated with |
| `rows` | Yes | array | Rows returned from `import-validate` |
| `charge_rate` | No | number | Required for utility charges |
| `notes` | No | string | Appended into invoice notes |

Each invoice line is recomputed from the row's readings or amount, `charge_rate` and the
`tax_id` tax, so the preview matches the invoices only when the same tax is used for both
calls.

Example request:

```json
{
  "charge_id": 15,
  "tax_id": 1,
  "charge_rate": 25,
  "notes": "February utility billing",
  "rows": [
    {
      "lease_id": 101,
      "previous": 120,
      "current": 150,
      "amount": 750,
      "has_error": false
    }
  ]
}
```

Example response:

```json
{
  "message": "Imported 3 row(s). Skipped 1 row(s).",
  "invoice_ids": [9001, 9002],
  "errors": [
    {
      "row": { "lease_id": 999999, "amount": 1200, "has_error": false },
      "error": "Lease ID 999999 does not exist."
    }
  ],
  "summary": {
    "imported_rows": 3,
    "skipped_rows": 1,
    "amount": 3750,
    "tax": 600,
    "total": 4350
  }
}
```

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
