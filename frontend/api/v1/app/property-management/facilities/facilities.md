# Facilities API

Domain: `Property Management`

Base prefix:

`/api/v1/app/{company}/property-management`

## Facility Types

- `GET /facility-types`

Full endpoint:

`GET /api/v1/app/{company}/property-management/facility-types`

Notes:

- No request body.
- Returns all facility type options.

## Facilities (Main Resource)

Endpoints:

- `GET /facilities`
- `POST /facilities`
- `GET /facilities/{facility}`
- `PUT/PATCH /facilities/{facility}`
- `DELETE /facilities/{facility}`
- `PATCH /facilities/{facility}/activate`
- `PATCH /facilities/{facility}/deactivate`
- `GET /facilities/{facility}/admins`
- `GET /facilities/{facility}/spaces`

### List facilities

`GET /api/v1/app/{company}/property-management/facilities`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[name]`
  - `filter[landlord_id]`
  - `filter[landlord.name]`
- Sort:
  - `sort=id,name,created_at`
- Include (supported):
  - `include=landlord,facilityType,country,city`
- Select fields (supported):
  - `fields[facilities]=id,name,facility_type_id,status,created_at`
- Pagination:
  - `per_page`, `page`

Examples:

- `GET /api/v1/app/12/property-management/facilities?filter[name]=Parkview&sort=-created_at`
- `GET /api/v1/app/12/property-management/facilities?include=landlord,facilityType&fields[facilities]=id,name,status`

### Create/Update facility payload

Fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_type_id` | Yes | integer | Must exist in `facility_types.id` |
| `space_unit_id` | Yes | integer | Must exist in `stock_keeping_units.id` |
| `name` | Yes | string | - |
| `leasable_space` | Yes | number | - |
| `common_space` | Yes | number | - |
| `landlord_space` | Yes | number | - |
| `physical_address` | Yes | string | - |
| `landlord_id` | No | integer | Defaults to current user if omitted |
| `description` | No | string | Optional |
| `construction_date` | No | date (`YYYY-MM-DD`) | Optional |
| `has_blocks` | No | boolean | Optional |
| `has_wings` | No | boolean | Optional |
| `country_id` | No | integer | Must exist in `countries.id` |
| `city_id` | No | integer | Must exist in `cities.id` |
| `coordinates` | No | string/object | Optional |

Example create request:

```json
{
  "facility_type_id": 2,
  "space_unit_id": 1,
  "name": "Parkview Residency",
  "description": "Mixed-use commercial and residential property",
  "construction_date": "2021-06-15",
  "has_blocks": true,
  "has_wings": true,
  "leasable_space": 18000,
  "common_space": 4200,
  "landlord_space": 800,
  "country_id": 110,
  "city_id": 901,
  "physical_address": "Westlands, Nairobi",
  "coordinates": "-1.2641,36.8106"
}
```

### Facility status actions

- `PATCH /api/v1/app/{company}/property-management/facilities/{facility}/activate`
- `PATCH /api/v1/app/{company}/property-management/facilities/{facility}/deactivate`

No request body required.

## Blocks, Wings, Floors

### Facility-level nested endpoints

- Blocks:
  - `GET /facilities/{facility}/blocks`
  - `POST /facilities/{facility}/blocks`
- Wings:
  - `GET /facilities/{facility}/wings`
  - `POST /facilities/{facility}/wings`
- Floors:
  - `GET /facilities/{facility}/floors`
  - `POST /facilities/{facility}/floors`
  - `POST /facilities/{facility}/floors/generate`

### Global block/wing/floor endpoints

- Blocks:
  - `GET /facilities/blocks/{block}`
  - `PUT/PATCH /facilities/blocks/{block}`
  - `DELETE /facilities/blocks/{block}`
  - `GET /facilities/blocks/{block}/wings`
  - `POST /facilities/blocks/{block}/wings`
  - `GET /facilities/blocks/{block}/floors`
  - `POST /facilities/blocks/{block}/floors`
  - `POST /facilities/blocks/{block}/floors/generate`
- Wings:
  - `GET /facilities/wings/{wing}`
  - `PUT/PATCH /facilities/wings/{wing}`
  - `DELETE /facilities/wings/{wing}`
  - `GET /facilities/wings/{wing}/floors`
  - `POST /facilities/wings/{wing}/floors`
  - `POST /facilities/wings/{wing}/floors/generate`
- Floors:
  - `GET /facilities/floors/{floor}`
  - `PUT/PATCH /facilities/floors/{floor}`
  - `DELETE /facilities/floors/{floor}`

Payload fields:

- Block (`FacilityBlockData`):
  - `name` (required, string)
  - `has_wings` (optional, boolean, default `false`)
  - `leasable_space` (optional, integer)
  - `common_space` (optional, integer)
  - `landlord_space` (optional, integer)
- Wing (`FacilityWingData`):
  - `name` (required, string)
  - `leasable_space` (optional, integer)
  - `common_space` (optional, integer)
  - `landlord_space` (optional, integer)
- Floor (`FacilityFloorDto`):
  - `name` (required, string)
  - `is_parking` (required, boolean)
  - `is_divisible` (required, boolean)
  - `leasable_space` (required, numeric)
  - `common_space` (required, numeric, min `0`)
  - `landlord_space` (required, numeric, min `0`)

AI floor generation payload (`/floors/generate`):

- `prompt` (required, string)

## Facility Roles

Endpoints:

- `GET /facilities/{facility}/roles`
- `PUT /facilities/{facility}/roles/{role}`

Assign role request body:

- `user_id` (required, integer)

## Facility Spaces (Standalone Resource)

Base route:

`/api/v1/app/{company}/property-management/facility-spaces/spaces`

Endpoints:

- `GET /facility-spaces/spaces`
- `POST /facility-spaces/spaces`
- `GET /facility-spaces/spaces/{space}`
- `PUT/PATCH /facility-spaces/spaces/{space}`
- `DELETE /facility-spaces/spaces/{space}`

List query params:

- Filters:
  - `filter[id]`, `filter[name]`, `filter[space_type]`, `filter[size]`, `filter[status]`, `filter[created_at]`, `filter[facility_floor_id]`
- Sort:
  - `sort=id,name,size,status,created_at`
- Include: not supported
- Select fields:
  - `fields[spaces]=id,name,space_type,size,status,created_at`

Create/update payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `name` | Yes | string | - |
| `facility_floor_id` | Yes (create) | integer | Must exist in `facility_floors.id` |
| `space_type` | Yes | string | Allowed values: `landlord`, `common`, `leasable` |
| `size` | Yes | integer | Minimum `0` |
| `is_parking` | No | boolean | Default `false` |
| `is_signage` | No | boolean | Default `false` |

## Facility Utility Prices and Meters

Utility prices:

- `GET /facilities/{facility}/utility-prices` (list with filter/sort/pagination)
- `PUT/PATCH /facilities/{facility}/utility-prices/{utilityPrice}` (update)

Utility price update payload:

- `facility_id` (optional, integer)
- `currency_id` (optional, integer)
- `price_per_unit` (optional, integer)
- `utility_id` (optional, integer)

Utility meters:

- `GET /facilities/{facility}/meters` (list with filter/sort/pagination)
- `PUT/PATCH /facilities/{facility}/meters/{meter}` (update)

Utility meter payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `meter_number` | Yes | string | - |
| `meter_serial_number` | Yes | string | Unique |
| `name` | Yes | string | - |
| `utility_id` | Yes | integer | Must exist in `utilities.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `reading_unit` | Yes | integer | Must exist in `stock_keeping_units.id` |
| `initial_reading` | Yes | number | - |
| `description` | No | string | Optional |
| `status` | No | string | `active`, `inactive` |

## Errors

Use shared error guidance from `docs/frontend/app/README.md`:

- Render `4xx` errors directly to users.
- Show generic fallback for `5xx`.
