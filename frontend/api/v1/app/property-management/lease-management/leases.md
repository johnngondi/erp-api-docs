# Leases API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases`

## Main Endpoints

- `GET /leases` (list)
- `POST /leases` (create)
- `GET /leases/{lease}` (show)
- `PUT/PATCH /leases/{lease}` (update)
- `DELETE /leases/{lease}` (delete)
- `PATCH /leases/{lease}/activate`
- `PATCH /leases/{lease}/suspend`
- `PATCH /leases/{lease}/terminate`

## List Leases

`GET /api/v1/app/{company}/property-management/lease-management/leases`

Supported query params:

- Filters:
  - `filter[user_id]`
  - `filter[status]`
  - `filter[created_at]`
  - `filter[start_at]`
  - `filter[end_at]`
  - `filter[next_due_at]`
  - `filter[billing_cycle]`
- Sort:
  - `sort=user_id,status,created_at,start_at,end_at,next_due_at,billing_cycle`
  - Prefix with `-` for descending (example: `sort=-start_at`)
- Include: not supported
- Select fields: not supported
- Pagination:
  - `per_page` (optional)
  - `page` (optional)

Examples:

- `GET /api/v1/app/12/property-management/lease-management/leases?filter[status]=active&sort=-start_at`
- `GET /api/v1/app/12/property-management/lease-management/leases?filter[user_id]=77&per_page=10&page=2`

## Create Lease

`POST /api/v1/app/{company}/property-management/lease-management/leases`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `user_id` | Yes | integer | Must exist in `users.id` |
| `start_at` | Yes | date (`YYYY-MM-DD`) | - |
| `period_in_years` | Yes | integer | - |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `period_in_months` | Yes | integer | - |
| `billing_cycle` | Yes | string | `biennial`, `annually`, `biannual`, `quarterly`, `monthly`, `weekly`, `daily`, `hourly` |
| `next_due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `status` | No | string | `active`, `suspended`, `terminated` |
| `currency_id` | No | integer | Must exist in `currencies.id` (defaults to `1` when omitted) |

Example request:

```json
{
  "user_id": 77,
  "start_at": "2026-01-01",
  "period_in_years": 1,
  "facility_id": 14,
  "period_in_months": 0,
  "billing_cycle": "monthly",
  "next_due_at": "2026-03-01",
  "status": "active",
  "currency_id": 1
}
```

Example response:

```json
{
  "message": "Lease created successfully",
  "lease": {
    "id": 101,
    "user": { "id": 77, "name": "Jane Tenant" },
    "facility": { "id": 14, "name": "Parkview Residency" },
    "billing_cycle": "monthly",
    "status": { "value": "active", "color": "success" },
    "start_at": { "raw": "2026-01-01", "formatted": "01 Jan, 2026" },
    "next_due_at": { "raw": "2026-03-01", "formatted": "01 Mar, 2026" }
  }
}
```

## Update Lease

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}`

Use the same payload shape as create.

## Lease Status Actions

No request body required:

- Activate: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/activate`
- Suspend: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/suspend`
- Terminate: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/terminate`

Response shape:

```json
{
  "message": "Lease suspended successfully",
  "lease": {
    "id": 101,
    "status": { "value": "suspended", "color": "secondary" }
  }
}
```

## Lease Items and Components

### Lease items endpoints

- `GET /leases/{lease}/items`
- `POST /leases/{lease}/items`
- `GET /leases/{lease}/items/{leaseItem}`
- `PUT/PATCH /leases/{lease}/items/{leaseItem}` (currently disabled by backend and returns validation error)
- `DELETE /leases/{lease}/items/{leaseItem}`

Create lease item payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_space_id` | Yes | integer | Must exist in `facility_spaces.id`, must belong to lease facility, and must not be occupied |
| `components` | Yes | array | Array of lease item component objects |

Component object inside `components`:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_component_id` | Yes | integer | Must exist in `lease_components.id` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |
| `cost_per_sqft` | Yes | integer | - |
| `hs_code` | No | string | Optional |

### Lease item components endpoints

- `GET /leases/{lease}/items/{leaseItem}/components`
- `POST /leases/{lease}/items/{leaseItem}/components`
- `GET /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`
- `PUT/PATCH /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`
- `DELETE /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`

List component query params:

- Filters: `filter[lease_item_id]`, `filter[lease_component_id]`, `filter[cost_per_sqft]`, `filter[cost_per_month]`, `filter[tax_id]`, `filter[hs_code]`
- Sort: `sort=lease_item_id,lease_component_id,cost_per_sqft,cost_per_month,tax_id,hs_code`
- Include: not supported
- Select fields: supported
  - `fields[lease_item_components]=id,lease_item_id,lease_component_id,cost_per_sqft,cost_per_month,tax_id,hs_code`

Example list with field selection:

`GET /api/v1/app/12/property-management/lease-management/leases/101/items/55/components?fields[lease_item_components]=id,lease_component_id,cost_per_month&sort=-cost_per_month`

## Related Docs

- `docs/frontend/app/property-management/lease-management/lease-deposits.md`
- `docs/frontend/app/property-management/lease-management/lease-escalations.md`
- `docs/frontend/app/property-management/lease-management/lease-opening-balances.md`

## Frontend Error Handling

Apply the shared rules in `docs/frontend/app/README.md`:

- Show backend `message` and field `errors` for `4xx`.
- Show generic fallback message for `5xx`.
