# Lease Charges Import API

Base route:
`/api/v1/crm/lease-management/charges`

## Endpoints

### Validate import file
`POST /api/v1/crm/lease-management/charges/import-validate`

Consumes:
`multipart/form-data`

Form-data fields:
- `charge_id` (int, required)  
  Must exist in `lease_components.id` and have `is_a_charge = true`.
- `import_file` (file, required)  
  Excel/CSV file (`.xlsx`, `.xls`, `.csv`).
- `charge_rate` (number, optional)  
  Required for utility charges and must be `> 0`.  
  For land fee charges, used as the default amount when the row amount is missing.

Excel columns (header row):
- **Land fee charge**: `lease_id`, `amount`
- **Utility charge**: `lease_id`, `previous_reading`, `current_reading`

Validations performed:
- **All charges**
  - `lease_id` must exist in `leases.id`.
- **Land fee charges**
  - `amount` must be greater than `0`.
  - If `amount` is missing in the file, the request `charge_rate` is used and a warning is returned.
- **Utility charges**
  - `current_reading` cannot be less than `previous_reading`.
  - `charge_rate` is required and must be greater than `0`.

Response (success):
```json
{
  "message": "Import file validated.",
  "rows": [
    {
      "lease_id": 101,
      "user": {
        "id": 7,
        "name": "Jane Tenant",
        "email": "jane@example.com",
        "phone": "+123456789",
        "profile_photo_url": "https://example.com/avatar.jpg"
      },
      "amount": 1500,
      "has_warning": false,
      "warning": null,
      "has_error": false,
      "error": null
    }
  ]
}
```

Response (utility charge row):
```json
{
  "message": "Import file validated.",
  "rows": [
    {
      "lease_id": 101,
      "user": {
        "id": 7,
        "name": "Jane Tenant",
        "email": "jane@example.com",
        "phone": "+123456789",
        "profile_photo_url": "https://example.com/avatar.jpg"
      },
      "previous": 120,
      "current": 150,
      "amount": 300,
      "has_warning": false,
      "warning": null,
      "has_error": false,
      "error": null
    }
  ]
}
```

Response (row warnings/errors example):
```json
{
  "message": "Import file validated.",
  "rows": [
    {
      "lease_id": 999999,
      "user": null,
      "amount": null,
      "has_warning": false,
      "warning": null,
      "has_error": true,
      "error": "Lease ID 999999 does not exist."
    },
    {
      "lease_id": 101,
      "user": {
        "id": 7,
        "name": "Jane Tenant",
        "email": "jane@example.com",
        "phone": "+123456789",
        "profile_photo_url": "https://example.com/avatar.jpg"
      },
      "amount": 250,
      "has_warning": true,
      "warning": "Amount not provided in file; default charge_rate applied.",
      "has_error": false,
      "error": null
    }
  ]
}
```

Notes:
- The calculation for utility charge amount is:  
  `(current_reading - previous_reading) * charge_rate`.

### Process import
`POST /api/v1/crm/lease-management/charges/import-process`

Request body (JSON):
- `charge_id` (int, required)  
  Must exist in `lease_components.id` and have `is_a_charge = true`.
- `tax_id` (int, required)  
  Tax to apply on invoice items.
- `charge_rate` (number, optional)  
  Required for utility charges and must be `> 0`.
- `notes` (string, optional)  
  Added to invoice notes together with the charge name.
- `rows` (array, required)  
  Rows returned from `import-validate`. Any row with `has_error = true` is skipped.

Processing rules:
- Finds (or creates) a `lease_item_component` for the given `charge_id` on the lease.
  - If no lease items exist for the lease, the row is skipped and added to errors.
- Utility charges:
  - `consumed = current - previous`
  - Invoice item `quantity = consumed`
  - Invoice item `amount = consumed * charge_rate`
  - Item notes include previous/current/consumption
- Land fee charges:
  - Uses `row.amount`, or `charge_rate` if amount is missing
  - Item notes are the charge name
- Invoice notes: `{notes} - {charge name}` (or just `{charge name}` if no notes)

Response:
```json
{
  "message": "Imported 3 row(s). Skipped 1 row(s).",
  "invoice_ids": [9001, 9002],
  "errors": [
    {
      "row": {
        "lease_id": 999999,
        "amount": 1200,
        "has_error": false
      },
      "error": "Lease ID 999999 does not exist."
    }
  ]
}
```
