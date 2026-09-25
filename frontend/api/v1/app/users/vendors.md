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
  - `filter[search]` (Scout search across name, email, phone and tax pin; supports CSV user ids e.g. `12,45`)
  - `filter[id]`
  - `filter[name]`
  - `filter[email]`
  - `filter[phone]`
  - `filter[created_at]`
  - `filter[country_id]` — exact
  - `filter[city_id]` — exact match on the supplier's region
  - `filter[business_registration_number]` — partial match
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
| `address` | No | string (max 255) | The supplier's physical or postal address |
| `business_registration_number` | No | string (max 255) | Company registration number. **Not unique** — legacy data has never been deduplicated |
| `country_id` | No | integer | Must exist in `countries.id` |
| `city_id` | No | integer | The supplier's **region**. Must exist in `cities.id`, and **must sit in `country_id`** when one is given |
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

## Supplier details

`address`, `business_registration_number`, `country_id` and `city_id` are accepted on **create and
update**, and all four are **optional**.

They are optional on purpose, not by oversight. Suppliers are `users` rows, and the same action that
creates them also creates public registrations, staff, fleet drivers and a vendor's own sub-users —
none of which carry supplier details. The EPMAS importer creates suppliers from legacy records that
may hold none of them. Making any of the three required at this layer would break all of that; a
supplier-specific rule belongs on this endpoint alone.

`city_id` is the **region**, and `country_id` is stored beside it rather than derived from it — the
form needs a country to narrow the city list, and a supplier can have a country before a town is
chosen. The resource returns them as siblings, so a country is readable without a city:

```jsonc
"country": { "id": 1, "name": "Kenya" },
"city":    { "id": 42, "name": "Nakuru" }
```

> **A city must sit in the country given.** `cities.country_id` already names the country, so the
> two are a second copy of the same fact and free to disagree — a supplier in Nakuru stamped
> Tanzania, with nothing to say which is right. Sending a mismatched pair answers `422` on
> `city_id`. Either may be sent alone.

`business_registration_number` carries **no unique index**. The column is being backfilled from
legacy data that has never been deduplicated, so uniqueness can only be added once that data is
known to be clean.

> **Updating one of these means re-sending `expense_sub_types`.** It is `required` on the update
> endpoint and Laravel's `required` rejects an empty array, so a partial update that changes only an
> address answers `422`. Send the supplier's existing prequalifications alongside the change.

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
