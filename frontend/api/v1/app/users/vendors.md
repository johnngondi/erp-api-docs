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
| `is_statutory_vendor` | No | boolean | Optional |
| `is_withholding_exempt` | No | boolean | Optional; when `true`, no withholding is applied to the vendor's bills even if withholding ids are supplied |
| `is_tax_merchant` | No | boolean | Optional; flags the vendor as a tax merchant |
| `withholds` | No | array | Optional |
| `bank_account` | No | object | Optional. When present, a bank account is created and owned by the vendor. See the table below. |

`bank_account` object fields (validated against `BankAccountData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `bank_branch_id` | Yes | integer | Must exist in `bank_branches.id` |
| `account_name` | Yes | string | Name on the account |
| `account_number` | Yes | string | Must be unique in `bank_accounts.account_number` |
| `account_number_confirmation` | Yes | string | Must match `account_number` (`confirmed` rule) |

The backend automatically sets `type = user`, `user_id` (the created vendor) and `user_group_id` (the vendor
group) — do not send these. If the bank details fail validation, the entire vendor creation is rolled back.

Example request:

```json
{
  "name": "Prime Mechanical Ltd",
  "email": "ops@primemechanical.co.ke",
  "phone": "+254733001122",
  "portal": 4,
  "terms": true,
  "policy": true,
  "is_statutory_vendor": true,
  "bank_account": {
    "bank_branch_id": 12,
    "account_name": "Prime Mechanical Ltd",
    "account_number": "0123456789",
    "account_number_confirmation": "0123456789"
  }
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
