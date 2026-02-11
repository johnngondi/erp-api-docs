# Lease Charges Imports API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/charges`

## Endpoints

- `POST /charges/import-validate`
- `POST /charges/import-process`

## Validate Import File

`POST /api/v1/app/{company}/property-management/lease-management/charges/import-validate`

Content type:

- `multipart/form-data`

Form-data fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `charge_id` | Yes | integer | Must exist in `lease_components.id` and be a charge component |
| `import_file` | Yes | file | `.xlsx`, `.xls`, or `.csv` |
| `charge_rate` | No | number | Required for utility charges and must be `> 0` |

Supported file headers:

- Land fee style: `lease_id`, `amount`
- Utility style: `lease_id`, `previous_reading`, `current_reading`

Validation behavior:

- Invalid rows are returned with `has_error: true`.
- Rows with recoverable issues can return `has_warning: true`.
- Utility charge amount formula:
  - `(current_reading - previous_reading) * charge_rate`

Example response:

```json
{
  "message": "Import file validated.",
  "rows": [
    {
      "lease_id": 101,
      "user": { "id": 77, "name": "Jane Tenant" },
      "amount": 1250,
      "has_warning": false,
      "warning": null,
      "has_error": false,
      "error": null
    },
    {
      "lease_id": 999999,
      "user": null,
      "amount": null,
      "has_warning": false,
      "warning": null,
      "has_error": true,
      "error": "Lease ID 999999 does not exist."
    }
  ]
}
```

## Process Import

`POST /api/v1/app/{company}/property-management/lease-management/charges/import-process`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `charge_id` | Yes | integer | Must exist in `lease_components.id` and be a charge component |
| `tax_id` | Yes | integer | Tax to apply to generated invoice items |
| `rows` | Yes | array | Rows returned from `import-validate` |
| `charge_rate` | No | number | Required for utility charges |
| `notes` | No | string | Appended into invoice notes |

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
  ]
}
```

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
