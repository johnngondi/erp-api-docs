# Assets API

Domain: `Facility Management`

Base prefix:

`/api/v1/app/{company}/facility-management/assets`

## Asset Endpoints

- `GET /assets`
- `POST /assets`
- `GET /assets/{asset}`
- `PUT/PATCH /assets/{asset}`
- `DELETE /assets/{asset}`

## List Assets

`GET /api/v1/app/{company}/facility-management/assets`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[name]`
  - `filter[facility_id]`
  - `filter[asset_type_id]`
  - `filter[created_at]`
  - `filter[status]`
  - `filter[warranty_status]`
- Sort:
  - `sort=id,created_at,purchase_price,warranty_status,warranty_expiration_at`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Examples:

- `GET /api/v1/app/12/facility-management/assets?filter[facility_id]=14&sort=-created_at`
- `GET /api/v1/app/12/facility-management/assets?filter[warranty_status]=active&per_page=20`

## Create/Update Asset

`POST /api/v1/app/{company}/facility-management/assets`  
`PUT/PATCH /api/v1/app/{company}/facility-management/assets/{asset}`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `name` | Yes | string | - |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `facility_space_id` | Yes | integer | Must exist in `facility_spaces.id` |
| `asset_type_id` | Yes | integer | Must exist in `asset_types.id` |
| `purchased_at` | Yes | date (`YYYY-MM-DD`) | - |
| `currency` | Yes | string | Must exist in `currencies.code` (example: `KES`, `USD`) |
| `purchase_price` | Yes | integer | Numeric |
| `value_change_type` | Yes | string | `appreciate`, `depreciate` |
| `value_change_rate` | Yes | number | Numeric |
| `value_change_duration` | Yes | string | `biennial`, `annually`, `biannual`, `quarterly`, `monthly`, `weekly`, `daily`, `hourly` |
| `last_maintenance_at` | Yes | date (`YYYY-MM-DD`) | - |
| `current_value` | No | number | Defaults to purchase price when omitted |
| `last_value_change_at` | Conditional | date (`YYYY-MM-DD`) | Required when `current_value` is not `0` |
| `warranty_status` | No | string | `active`, `inactive` |
| `warranty_period_in_days` | No | number | Defaults to `365` |
| `has_inventory` | No | boolean | Default `false` |
| `description` | No | string | Optional |
| `photo` | No | string | Optional |

Example create request:

```json
{
  "name": "Main Generator",
  "facility_id": 14,
  "facility_space_id": 223,
  "asset_type_id": 6,
  "purchased_at": "2025-04-10",
  "currency": "KES",
  "purchase_price": 1800000,
  "value_change_type": "depreciate",
  "value_change_rate": 8,
  "value_change_duration": "annually",
  "last_maintenance_at": "2026-01-15",
  "warranty_status": "active",
  "warranty_period_in_days": 730,
  "description": "Backup diesel generator"
}
```

Example response:

```json
{
  "message": "Asset created successfully.",
  "asset": {
    "id": 401,
    "name": "Main Generator",
    "purchase_price": 1800000,
    "warranty_status": { "value": "active", "color": "success" },
    "value_change_type": { "value": "depreciate", "color": "danger" }
  }
}
```

## Insurance Submodule

Base prefix:

`/api/v1/app/{company}/facility-management/assets/insurance`

### Insurance Providers

Endpoints:

- `GET /insurance/providers`
- `POST /insurance/providers`
- `GET /insurance/providers/{provider}`
- `PUT/PATCH /insurance/providers/{provider}`
- `DELETE /insurance/providers/{provider}`

List query support:

- Filters:
  - `filter[name]`, `filter[insurance_company_name]`, `filter[contact_person]`
- Include:
  - `include=insurancePolicies`
- Sort: not documented
- Fields: not documented

Create/Update payload (`InsuranceProviderData`):

| Field | Required | Type |
|---|---|---|
| `name` | Yes | string |
| `contact_person` | Yes | string |
| `contact_phone` | Yes | string |
| `insurance_company_name` | Yes | string |
| `claim_email` | Yes | string |
| `claim_phone` | Yes | string |
| `email` | No | string |
| `phone` | No | string |
| `contact_email` | No | string |

### Insurance Policies

Endpoints:

- `GET /insurance/policies`
- `POST /insurance/policies`
- `GET /insurance/policies/{policy}`
- `PUT/PATCH /insurance/policies/{policy}`
- `DELETE /insurance/policies/{policy}`

List query support:

- Filters:
  - `filter[id]`, `filter[policy_number]`, `filter[start_date]`, `filter[end_date]`, `filter[owner_id]`
- Sort:
  - `sort=start_date,end_date,created_at,premium_pa,coverage_amount`
- Include:
  - `include=assets`
- Fields: not documented

Create/Update payload (`InsurancePolicyData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `policy_number` | Yes | string | - |
| `provider_id` | Yes | integer | Must exist in `insurance_providers.id` |
| `owner_id` | Yes | integer | Must exist in `users.id` |
| `negotiated_by` | Yes | string | - |
| `currency` | Yes | string | Currency code/value used by backend |
| `premium_pa` | Yes | number | Annual premium |
| `coverage_amount` | Yes | number | - |
| `start_date` | Yes | date (`YYYY-MM-DD`) | - |
| `duration_in_days` | Yes | integer | Used to compute `end_date` |
| `signed_by` | No | integer | Must exist in `users.id` |
| `upload_id` | No | integer | Must exist in `uploads.id` |
| `covered_assets` | No | array | Asset IDs covered by policy |

## Frontend Error Handling

Use shared behavior in `docs/frontend/app/README.md`:

- Show `4xx` validation/permission errors to users.
- Show a generic fallback message for `5xx`.
