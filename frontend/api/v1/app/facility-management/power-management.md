# Power Management API

Domain: `Facility Management`

Base prefix:

`/api/v1/app/{company}/facility-management/power-management`

## Power Sources

Endpoints:

- `GET /power-sources`
- `POST /power-sources`
- `GET /power-sources/{powerSource}`
- `PUT/PATCH /power-sources/{powerSource}`
- `DELETE /power-sources/{powerSource}`
- `PATCH /power-sources/{powerSource}/suspend`
- `PATCH /power-sources/{powerSource}/decomission` (route uses this spelling)

List query support:

- Filters:
  - `filter[power_source_type_id]`, `filter[name]`, `filter[co2_per_kwh]`
  - `filter[max_generation_capacity]`, `filter[commissioned_at]`, `filter[asset_id]`
  - `filter[has_maintenance_contract]`, `filter[maintenance_contract_id]`
  - `filter[status]`, `filter[activity_status]`, `filter[created_at]`
- Sort:
  - `sort=created_at,co2_per_kwh,max_generation_capacity,activity_status`
- Include: not documented
- Fields: not documented

Create/Update payload (`PowerSourceData`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `name` | Yes | string | Required for create |
| `power_source_type_id` | No | integer | `power_source_types.id` |
| `facility_id` | No | integer | `facilities.id` |
| `co2_per_kwh` | No | number | Numeric |
| `max_generation_capacity` | No | number | Numeric |
| `commissioned_at` | No | date | - |
| `description` | No | string | Optional |
| `is_asset` | No | boolean | Default `false` |
| `asset_id` | No | integer | `assets.id` |
| `has_maintenance_contract` | No | boolean | Default `false` |
| `maintenance_contract_id` | No | integer | `contracts.id` |
| `status` | No | string | Common values: `in-use`, `suspended`, `decommissioned` |
| `activity_status` | No | string | Common values: `running`, `off` |
| `suspension_reason` | No | string | Optional |
| `decommission_reason` | No | string | Optional |

Notes:

- Suspend/decommission actions enrich payload with actor and timestamp server-side.

## Power Source Activity Logs

Endpoints:

- `GET /power-sources/{powerSource}/activity-logs`
- `POST /power-sources/{powerSource}/activity-logs`
- `GET /power-sources/activity-logs/{powerSourceActivityLog}`
- `PUT/PATCH /power-sources/activity-logs/{powerSourceActivityLog}`
- `DELETE /power-sources/activity-logs/{powerSourceActivityLog}`

List query support:

- Filters:
  - `filter[id]`, `filter[power_source_id]`, `filter[start_at]`, `filter[stop_at]`, `filter[power_generated]`
- Sort:
  - `sort=id,power_source_id,start_at,stop_at,power_generated`
- Fields:
  - `fields[power_source_activity_logs]=id,power_source_id,start_at,stop_at,power_generated`

Create/Update payload (`PowerSourceActivityLogData`):

| Field | Required | Type |
|---|---|---|
| `start_at` | Yes | date |
| `stop_at` | Yes | date |
| `power_generated` | Yes | number |
| `power_source_id` | No | integer (`power_sources.id`) |

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show `4xx` messages/validation details.
- Show generic fallback for `5xx`.
