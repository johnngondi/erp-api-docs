# Utility Management API

Domain: `Facility Management`

Base prefix:

`/api/v1/app/{company}/facility-management/utility-management`

## Utility Meters

Endpoints:

- `GET /meters`
- `POST /meters`
- `GET /meters/{meter}`
- `PUT/PATCH /meters/{meter}`
- `DELETE /meters/{meter}`
- `PATCH /meters/{meter}/activate`
- `PATCH /meters/{meter}/deactivate`

List query support:

- Filters:
  - `filter[meter_number]`, `filter[meter_serial_number]`, `filter[name]`
  - `filter[utility_id]`, `filter[reading_unit]`, `filter[status]`, `filter[created_at]`
- Sort:
  - `sort=meter_number,meter_serial_number,name,utility_id,reading_unit,status,created_at`
- Include: not documented
- Fields: not documented

Create/Update payload (`UtilityMeterData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `meter_number` | Yes | string | - |
| `meter_serial_number` | Yes | string | Unique |
| `name` | Yes | string | - |
| `utility_id` | Yes | integer | `utilities.id` |
| `facility_id` | Yes | integer | `facilities.id` |
| `reading_unit` | Yes | integer | `stock_keeping_units.id` |
| `initial_reading` | Yes | number | - |
| `description` | No | string | Optional |
| `status` | No | string | `active`, `inactive` |

## Utility Meter Readings

Endpoints:

- `GET /meters/{meter}/readings`
- `POST /meters/{meter}/readings`
- `GET /meters/{meter}/readings/{reading}`
- `PUT/PATCH /meters/{meter}/readings/{reading}`
- `DELETE /meters/{meter}/readings/{reading}`

List query support:

- Filters:
  - `filter[current_reading]`, `filter[utility_meter_id]`, `filter[created_at]`
- Sort:
  - `sort=current_reading,utility_meter_id,created_at`

Create/Update payload (`UtilityMeterReadingData`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `current_reading` | Yes | number | - |
| `reading_image_id` | No | integer | `uploads.id` |

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show `4xx` messages/validation details.
- Show generic fallback for `5xx`.
