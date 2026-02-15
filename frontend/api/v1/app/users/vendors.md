# Vendors API

Domain: `Users`

Base prefix:

`/api/v1/app/{company}/users/vendors`

## Endpoints

Currently implemented:

- `GET /vendors`
- `POST /vendors`
- `GET /vendors/{vendor}`
- `DELETE /vendors/{vendor}`

Route exists but is not implemented in controller:

- `PUT/PATCH /vendors/{vendor}`

## List Vendors

`GET /api/v1/app/{company}/users/vendors`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[name]`
  - `filter[email]`
  - `filter[phone]`
  - `filter[created_at]`
- Sort:
  - `sort=id,name,created_at`
- Include:
  - Not supported
- Select fields:
  - Not supported
- Pagination:
  - `per_page`, `page`

## Create/Elevate Vendor

`POST /api/v1/app/{company}/users/vendors`

Behavior:

- If `email` matches an existing user, backend elevates that user to vendor.
- If no user exists, backend creates a new user then elevates.

Request fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `email` | Yes | string (email) | Used to check existing user |
| `name` | Conditional | string | Required when creating a brand-new user |
| `phone` | Conditional | string | Required when creating a brand-new user |
| `portal` | Conditional | integer | Required for brand-new user; must exist in `user_groups.id` |
| `terms` | Conditional | boolean/accepted | Required for brand-new user |
| `policy` | Conditional | boolean/accepted | Required for brand-new user |
| `has_vat` | No | boolean | - |
| `tax_pin` | Conditional | string | Required if `has_vat=true`; unique |
| `is_prequalified_vendor` | No | boolean | Optional |
| `withholds` | No | array | Optional |

Example request:

```json
{
  "name": "Prime Mechanical Ltd",
  "email": "ops@primemechanical.co.ke",
  "phone": "+254733001122",
  "portal": 4,
  "terms": true,
  "policy": true,
  "is_prequalified_vendor": true
}
```

Example response:

```json
{
  "message": "Vendor created successfully.",
  "user": {
    "id": 512,
    "name": "Prime Mechanical Ltd",
    "email": "ops@primemechanical.co.ke",
    "phone": "+254733001122",
    "status": {
      "value": "active",
      "color": "success",
      "label": "Active"
    }
  }
}
```

## Status Enum (Returned in Resource)

Common values returned for `status`:

- `pending` (color: `secondary`)
- `active` (color: `success`)
- `suspended` (color: `warning`)
- `inactive` (color: `danger`)

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show backend `4xx` messages and validation errors to users.
- Show generic fallback for `5xx` responses.
