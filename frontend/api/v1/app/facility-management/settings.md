# Facility Management Settings API

Domain: `Facility Management`

Base prefix:

`/api/v1/app/{company}/facility-management/settings`

## Utilities

Settings endpoints:

- `GET /utility-management/utilities`
- `POST /utility-management/utilities`
- `GET /utility-management/utilities/{utility}`
- `PUT/PATCH /utility-management/utilities/{utility}`
- `DELETE /utility-management/utilities/{utility}`

List query support:

- Filters:
  - `filter[id]`, `filter[name]`, `filter[description]`, `filter[utility_type]`
  - `filter[stock_keeping_unit_id]`, `filter[is_regulated]`
  - `filter[default_price_per_unit]`, `filter[created_at]`
- Sort:
  - `sort=id,name,description,utility_type,stock_keeping_unit_id,is_regulated,default_price_per_unit,created_at`

Create/Update payload (`UtilityData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `name` | Yes | string | - |
| `utility_type` | Yes | string | `electricity`, `water`, `gas`, `other` |
| `stock_keeping_unit_id` | Yes | integer | `stock_keeping_units.id` |
| `default_price_per_unit` | Yes | number | - |
| `description` | No | string | Optional |
| `is_regulated` | No | boolean | Default `false` |

## Utility Prices

Settings endpoints:

- `GET /utility-management/utilities/{utility}/prices`
- `POST /utility-management/utilities/{utility}/prices`
- `GET /utility-management/utilities/{utility}/prices/{price}`
- `PUT/PATCH /utility-management/utilities/{utility}/prices/{price}`
- `DELETE /utility-management/utilities/{utility}/prices/{price}`

List query support:

- Filters:
  - `filter[currency_id]`, `filter[price_per_unit]`, `filter[facility_id]`, `filter[created_at]`
- Sort:
  - `sort=currency_id,price_per_unit,facility_id,created_at`

Create/Update payload (`UtilityPriceData`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_id` | No | integer | `facilities.id` |
| `currency_id` | No | integer | `currencies.id` |
| `price_per_unit` | No | integer | - |
| `utility_id` | No | integer | `utilities.id` |

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show `4xx` messages/validation details.
- Show generic fallback for `5xx`.
