# Sensors API

Domain: `Facility Management`

Base prefix:

`/api/v1/app/{company}/facility-management/sensors`

## Sensors

Endpoints:

- `GET /sensors`
- `POST /sensors`
- `GET /sensors/{sensor}`
- `PUT/PATCH /sensors/{sensor}`
- `DELETE /sensors/{sensor}`
- `GET /sensors/available-devices`

List query support:

- Filters:
  - `filter[asset_id]`, `filter[facility_id]`, `filter[facility_space_id]`, `filter[type]`, `filter[current_status]`
- Sort: not documented
- Include: not documented
- Fields: not documented

Available devices endpoint:

- `GET /api/v1/app/{company}/facility-management/sensors/available-devices`
- Supports:
  - `filter[facility_id]`
- Returns unassigned sensor server devices.

Create/Update payload (`SensorData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | `facilities.id` |
| `sensor_unique_identifier` | Yes | string | Must exist in `sensor_server_devices.device_unique_identifier` and be unique in sensors |
| `type` | Yes | string | `asset`, `space` |
| `facility_space_id` | Conditional | integer | Required if `type=space` |
| `asset_id` | Conditional | integer | Required if `type=asset` |

Status values seen in responses:

- `ok`, `warning`, `critical`

## Sensor Actions

Endpoints:

- `GET /sensors/{sensor}/actions`
- `POST /sensors/{sensor}/actions`
- `GET /sensors/{sensor}/actions/{action}`
- `PUT/PATCH /sensors/{sensor}/actions/{action}`
- `DELETE /sensors/{sensor}/actions/{action}`

List query support:

- Filters:
  - `filter[action_trigger]`, `filter[action_to_take]`
- Sort: defaulted by backend (`action_trigger` desc)

Create/Update payload (`SensorActionData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `action_trigger` | Yes | string | `warning`, `critical` |
| `action_to_take` | Yes | string | `notify`, `ticket`, `request` |
| `notifiable` | Conditional | array | Required if `action_to_take=notify` |
| `priority` | Conditional | string | Required unless action is notify; `high`, `normal`, `low` |
| `issue_category_id` | Conditional | integer | Required if `action_to_take=ticket`; `ticket_categories.id` |
| `request_type` | Conditional | string | Required if `action_to_take=request`; `work`, `purchase` |
| `expense_type_id` | Conditional | integer | Required if `request_type=work`; `expense_types.id` |
| `expense_sub_type_id` | Conditional | integer | Required if `request_type=work`; `expense_sub_types.id` |
| `purchase_item_id` | Conditional | integer | Required if `request_type=purchase`; `inventories.id` |
| `purchase_item_quantity` | Conditional | integer | Required if `request_type=purchase` |

## Sensor Entries

Routes exist:

- `GET /sensors/{sensor}/entries`
- `POST /sensors/{sensor}/entries`
- `GET /sensors/{sensor}/entries/{entry}`
- `PUT/PATCH /sensors/{sensor}/entries/{entry}`
- `DELETE /sensors/{sensor}/entries/{entry}`

Current backend status:

- `SensorEntryController` is not implemented yet.
- Treat these endpoints as pending unless backend implementation is added.

Planned payload shape from `SensorEntryData`:

| Field | Required | Type |
|---|---|---|
| `current_reading` | No | string |
| `current_status` | Yes | string |
| `current_status_info` | No | string |

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show `4xx` messages/validation details.
- Show generic fallback for `5xx`.
