# Fleet API

Domain: `Facility Management`

Base prefix:

`/api/v1/app/{company}/facility-management/fleet`

## Drivers

Endpoints:

- `GET /drivers`
- `POST /drivers`
- `GET /drivers/{driver}`
- `PUT/PATCH /drivers/{driver}`
- `DELETE /drivers/{driver}`
- `PATCH /drivers/{driver}/suspend`
- `PATCH /drivers/{driver}/activate`
- `PATCH /drivers/{driver}/deactivate`

List query support:

- Filters:
  - `filter[id]`, `filter[user.name]`, `filter[user.email]`, `filter[user.phone]`
  - `filter[facility_id]`, `filter[facility.name]`
- Sort:
  - `sort=id,name,experience`
- Include:
  - `include=vehicles`
- Fields: not documented

Create/Update payload (`DriverData`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `name` | Yes | string | - |
| `phone` | Yes | string | - |
| `email` | Yes | string | - |
| `facility_id` | Yes | integer | Facility id |
| `experience` | Yes | integer | Years or numeric experience value |
| `license_upload_id` | No | integer | Upload id |
| `can_drive_manual` | No | boolean | Default `true` |
| `qualified_vehicle_categories` | No | array | Vehicle category IDs |

Suspend payload:

| Field | Required | Type | Notes |
|---|---|---|---|
| `days` | Yes | integer | Min `1` |
| `reason` | Yes | string | Suspension reason |

## Vehicles

Endpoints:

- `GET /vehicles`
- `POST /vehicles`
- `GET /vehicles/{vehicle}`
- `PUT/PATCH /vehicles/{vehicle}`
- `DELETE /vehicles/{vehicle}`
- `PATCH /vehicles/{vehicle}/suspend`
- `PATCH /vehicles/{vehicle}/activate`
- `PATCH /vehicles/{vehicle}/decommission`

List query support:

- Filters:
  - `filter[id]`, `filter[license_plate]`, `filter[description]`, `filter[vehicle_type]`
  - `filter[ownership_type]`, `filter[vehicle_make_id]`, `filter[vehicle_category_id]`
  - `filter[facility_id]`, `filter[facility.name]`
- Sort:
  - `sort=id,license_plate`
- Include:
  - `include=drivers`
- Fields: not documented

Create/Update payload (`VehicleData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `license_plate` | Yes | string | Must be unique |
| `color` | Yes | string | - |
| `vehicle_type` | Yes | string | `personal`, `passengers`, `cargo` |
| `ownership_type` | Yes | string | `own`, `lease` |
| `transmission_type` | No | string | `auto`, `manual` |
| `passengers` | No | integer | Default `1` |
| `description` | No | string | Optional |
| `vehicle_photo_upload_id` | No | integer | `uploads.id` |
| `document_upload_id` | No | integer | `uploads.id` |
| `vehicle_category_id` | No | integer | `vehicle_categories.id` |
| `vehicle_make_id` | No | integer | `vehicle_makes.id` |
| `manage_as_asset` | No | boolean | Default `true` |
| `allow_scheduling` | No | boolean | Default `true` |
| `asset_id` | No | integer | `assets.id` |
| `creator_id` | No | integer | `users.id` |
| `purchased_at` | No | date | - |
| `currency_id` | No | string | Backend currently types this as string |
| `gps_tracker_id` | No | integer | `sensors.id` |
| `purchase_price` | No | integer | - |
| `current_value` | No | integer | - |
| `status` | No | string | `in use`, `decommissioned`, `suspended` |
| `drivers` | No | array | Driver IDs |

## Trips

Endpoints:

- `GET /trips`
- `POST /trips`
- `GET /trips/{trip}`
- `PUT/PATCH /trips/{trip}`
- `DELETE /trips/{trip}`

List query support:

- Filters:
  - `filter[id]`, `filter[departure.name]`, `filter[destination.name]`
  - `filter[vehicle_id]`, `filter[vehicle.license_plate]`
  - `filter[driver_id]`, `filter[driver.name]`
  - `filter[departure_city_id]`, `filter[destination_city_id]`
  - `filter[depart_at]`, `filter[arrive_at]`, `filter[status]`, `filter[stops.city_id]`
- Sort:
  - `sort=id,depart_at,arrive_at,status`
- Include:
  - `include=bookings,stops`

Create/Update payload (`TripData`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `vehicle_id` | Yes | integer | `vehicles.id` |
| `driver_id` | Yes | integer | `drivers.id` |
| `departure_city_id` | Yes | integer | `cities.id` |
| `destination_city_id` | Yes | integer | `cities.id` |
| `depart_at` | Yes | date/datetime | Must be now or in the future |

Status values seen in responses:

- `open`, `full`, `in-progress`, `completed`

## Trip Stops

Endpoints:

- `GET /trips/{trip}/stops`
- `POST /trips/{trip}/stops`
- `GET /trips/{trip}/stops/{stop}`
- `PUT/PATCH /trips/{trip}/stops/{stop}`
- `DELETE /trips/{trip}/stops/{stop}`

List query support:

- Filters:
  - `filter[city.name]`, `filter[city_id]`

Create/Update payload (`TripStopData`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `city_id` | Yes | integer | `cities.id` |
| `stop_duration` | Yes | integer | Duration unit defined by backend |
| `order` | No | integer | Optional order index |

## Bookings

Endpoints:

- `GET /bookings`
- `POST /bookings`
- `GET /bookings/{booking}`
- `PUT/PATCH /bookings/{booking}`
- `DELETE /bookings/{booking}`
- `PATCH /bookings/{booking}/confirm`

List query support:

- Filters:
  - `filter[id]`, `filter[trip_id]`, `filter[user_id]`, `filter[pickup_city_id]`
  - `filter[status]`, `filter[pickup_at]`, `filter[luggage_size]`
  - `filter[has_luggage]`, `filter[trip.driver_id]`
- Sort:
  - `sort=id,pickup_at,status`
- Include:
  - `include=trip.vehicle,trip.driver,user`

Create/Update payload (`BookingData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `trip_id` | Yes | integer | `trips.id` |
| `user_id` | Yes | integer | `users.id` |
| `exact_pickup_location` | Yes | string | Max 255 chars |
| `pickup_at` | Yes | date/datetime | Must be now or in the future |
| `seats` | Yes | integer | Seats requested |
| `has_luggage` | Yes | boolean | - |
| `pickup_city_id` | No | integer | `cities.id` |
| `luggage_size` | No | string | `small`, `medium`, `large`, `huge` |
| `description` | No | string | Optional |

Status values seen in responses:

- `new`, `confirmed`, `boarded`

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show `4xx` messages/validation details.
- Show generic fallback for `5xx`.
