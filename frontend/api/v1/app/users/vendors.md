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

- `PUT/PATCH /vendors/{vendor}`
- `PATCH /vendors/{vendor}/suspend`
- `PATCH /vendors/{vendor}/activate`
- `PATCH /vendors/{vendor}/terminate`

> A supplier has four states — `active`, `suspended`, `terminated`, or deleted. See below.

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

Behavior, by what the email matches:

| The email | Result |
|---|---|
| belongs to nobody | A new user is created and made a supplier |
| belongs to someone who is **not** a supplier | That person is elevated, **and the supplier details sent with the request are applied to them** |
| belongs to someone who **is already a supplier** | `422` on `email`: *"A vendor with this email already exists."* |

**Read the returned `message`.** An elevation answers *"User with details already exists. They have
been elevated to a vendor"*, not *"Vendor created successfully"* — showing a generic success on an
elevation is what made a returning email look like a new supplier with blank fields.

On an elevation, identity is deliberately left alone: `name`, `phone` and `email` belong to the
person, who may already be a tenant or landlord under that name. Only supplier fields are written —
`tax_pin`, `address`, `business_registration_number`, `country_id`, `city_id`, `has_vat`,
`is_tax_merchant`, `is_statutory_vendor`, `is_withholding_exempt` — and only where the request
actually sent them.

## Suspend, Terminate and Activate

```
PATCH /api/v1/app/{company}/users/vendors/{vendor}/suspend
PATCH /api/v1/app/{company}/users/vendors/{vendor}/terminate
PATCH /api/v1/app/{company}/users/vendors/{vendor}/activate
```

No request body on any of them.

| Action | Permission | What it does |
|---|---|---|
| Suspend | `suspend-vendor` | `status` → `suspended`. Nothing else changes. |
| Terminate | `update-status-vendor` | `status` → `terminated`, **their contracts** → `terminated`, **their prequalified categories** switched off |
| Activate | `activate-vendor` | `status` → `active`, and prequalified categories restored |

**Terminate does not touch open LPOs or unpaid bills.** An unpaid bill is money owed, and ending a
relationship is not a decision to stop owing it.

**A terminated supplier can be given no new work.** New bills, liability bills, contracts and
direct LPOs naming them are refused with `422` on `vendor_id`:

> "This supplier has been terminated and cannot be given new work. Their existing bills and LPOs are unaffected."

Their **existing** bills and LPOs are untouched and run to finalisation as normal — an unpaid bill
is money owed. Filter supplier pickers with `filter[status]=active` so a terminated supplier is not
offered in the first place.

**A terminated supplier loses vendor portal access.** `ValidateUserIsActive` runs on every portal
and now turns them away the way it already turned away suspended accounts — they can still sign in
and get a token, but every portal call returns `403`:

> "Sorry! Your account has been terminated. Please contact the property manager."

Their existing bills and LPOs still run to finalisation; that paperwork is finished from the app
side rather than by the supplier.

Suspended suppliers are **not** blocked from new work. Suspension is a pause rather than an ending,
so it only affects how they are shown.

**Activating a terminated supplier does not reinstate their contracts.** Those stay `terminated`
and are brought back deliberately, one at a time. Only the prequalified categories the termination
itself switched off are restored — any that were already off beforehand stay off, so coming back
never re-approves something somebody had withdrawn.

`ContractStatus` gained `terminated` alongside `pending`, `active`, `suspended`, `expired` and
`inactive`. It renders `danger`.

### Which buttons to show

All four options are exposed and gated by permission alone — pick what to show from the supplier's
`status`, which is on the resource beside them:

```json
"permissions": {
  "view": true, "update": true,
  "suspend": true, "activate": true, "terminate": true,
  "delete": false
}
```

`delete` is the exception: it comes from the policy and goes **false** as soon as the supplier
holds anything at all — contracts, bills, LPOs, bids, bank accounts or prequalified categories —
because such a supplier would be terminated rather than deleted.

## Delete Vendor

`DELETE /api/v1/app/{company}/users/vendors/{vendor}`

**A delete request does not always delete.** The backend looks at what the supplier holds and
decides:

- **Holding nothing** — they are removed, `"Vendor deleted successfully"`.
- **Holding anything** — they are **terminated instead**, and told so:

```json
{
  "message": "This supplier was terminated rather than deleted, because they hold 4 contracts. Their contracts have been terminated and their prequalified categories switched off."
}
```

Both are `200`. Read the message rather than assuming the supplier is gone — and re-read the
returned `user`, whose `status` will be `terminated` in the second case.

This replaced a delete that destroyed records: `users` has 25 foreign keys that cascade, so
deleting a supplier took their financial history with it, or returned `500` where something further
down blocked the cascade.

> The endpoint's permission was also corrected. It gated on `delete-landlord`, so the flag the
> screen read and the permission the endpoint enforced belonged to different roles. It now checks
> `delete-vendor` when it will delete, and `update-status-vendor` when it will terminate.

`expense_sub_types` is **optional on update**. Omitting it leaves the supplier's prequalifications
untouched; sending `[]` clears them. It used to be required, so a supplier who had never been given
any could not be saved at all.

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
