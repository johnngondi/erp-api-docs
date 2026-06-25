# Utility Meters API

Domain: `Property Management`

Base prefix:

`/api/v1/app/{company}/property-management/facilities/utility-meters`

Utility meters are now a standalone resource (no longer nested under a specific
facility). The owning facility is supplied in the payload via `facility_id`.

## Utility Meters

Endpoints:

- `GET /facilities/utility-meters`
- `POST /facilities/utility-meters`
- `GET /facilities/utility-meters/{meter}`
- `PUT/PATCH /facilities/utility-meters/{meter}`
- `DELETE /facilities/utility-meters/{meter}`
- `PATCH /facilities/utility-meters/{meter}/activate`
- `PATCH /facilities/utility-meters/{meter}/deactivate`
- `POST /facilities/utility-meters/submit-readings`

List query support:

- Filters:
  - `filter[search]` (Scout-backed search across meter number, serial number, name, and the related facility/utility names; supports CSV IDs)
  - `filter[meter_number]`, `filter[meter_serial_number]`, `filter[name]`
  - `filter[utility_id]`, `filter[facility_id]`, `filter[reading_unit_id]`, `filter[status]`, `filter[created_at]`
- Sort:
  - `sort=meter_number,meter_serial_number,name,utility_id,facility_id,reading_unit_id,status,created_at`

Create/Update payload (`UtilityMeterData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `meter_number` | Yes | string | - |
| `meter_serial_number` | Yes | string | Unique across utility meters (ignores the meter being updated). |
| `name` | Yes | string | - |
| `description` | No | string | Optional |
| `facility_id` | Yes | integer | `facilities.id` |
| `utility_id` | Yes | integer | `lease_components.id` — populate the picker from lease components where `is_utility_charge = true`. |
| `reading_unit_id` | Yes | integer | `stock_keeping_units.id` (SKU). |
| `initial_reading` | Yes | number | Decimal, stored with 5 dp. Seeds `last_reading` on create. |
| `space_ids` | No | integer[] | Nullable. Each must exist in `facility_spaces.id`. Synced as a many-to-many relationship; send the full desired set on update. |
| `status` | No | string | `active`, `inactive` (defaults to `active`). |

Example create payload:

```json
{
    "meter_number": "12345",
    "meter_serial_number": "SN-987654322",
    "name": "Main Utility Meter",
    "description": "This is the main utility meter for the facility.",
    "facility_id": 1,
    "space_ids": [1, 2],
    "utility_id": 1,
    "reading_unit_id": 2,
    "initial_reading": "5667788989.023"
}
```

Response (`data`):

- `message`
- `utility_meter` — includes `utility` (lease component), `facility`, `reading_unit` (SKU), `spaces`, `last_reading`, `last_reading_at`, `last_reading_image`, `reading_status`, `submitted_at`, `is_faulty`, `fault_reason`, and `permissions`.

`last_reading` tracks the latest recorded reading. It is set to `initial_reading`
when the meter is created (with `last_reading_at` null, since no reading exists yet)
and is bumped automatically each time a reading is added. `last_reading_at` records
the timestamp of that most recent reading.

`last_reading_image` is the `Upload` attached to the most recent reading (same
shape as other upload fields; `null` when the latest reading had no image). It is
kept in sync with `last_reading`/`last_reading_at` whenever a reading is recorded,
so list and detail views can show the photo without loading the readings.

`reading_status` is derived from `last_reading_at` against the current month and is
intended to drive a status badge:

| Condition | `value` | `label` | `color` |
|---|---|---|---|
| `last_reading_at` is within the current month | `read` | `Read` | `success` |
| `last_reading_at` is in a prior month or null | `not read` | `Not Read` | `danger` |

```json
"reading_status": {
    "value": "read",
    "color": "success",
    "label": "Read"
}
```

`submitted_at` is the timestamp at which the meter's readings were submitted for the
current period's billing (see below); `null` means the meter is still open for the
month. It is returned as a date block (`raw`, `formatted`, `diff`).

### Submitting readings for billing

`POST /facilities/utility-meters/submit-readings`

Marks the supplied meters as submitted (`submitted_at = now()`) so their current
readings can be used to generate the period's bills. Sending a meter that is already
submitted simply refreshes its `submitted_at` timestamp.

Payload (`SubmitUtilityMeterReadingsData`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `meter_ids` | Yes | integer[] | At least one. Each must exist in `utility_meters.id`. The request fails (422) if any id cannot be found within the company. |

```json
{
    "meter_ids": [1, 2, 3]
}
```

Response (`data`):

- `message`
- `utility_meters` — the submitted meters as a `UtilityMeterResource` collection (now carrying a populated `submitted_at`).

Authorized by `update-utility-meter`. Every meter's `submitted_at` is cleared back to
`null` automatically on the first day of each month by the
`utility-meters:reset-submissions` scheduled command, reopening meters for the new
billing period.

## Utility Meter Readings

Readings are nested under a meter.

Endpoints:

- `GET /facilities/utility-meters/{meter}/readings`
- `POST /facilities/utility-meters/{meter}/readings`
- `GET /facilities/utility-meters/{meter}/readings/{reading}`
- `PUT/PATCH /facilities/utility-meters/{meter}/readings/{reading}`
- `DELETE /facilities/utility-meters/{meter}/readings/{reading}`

List query support:

- Filters:
  - `filter[current_reading]`, `filter[utility_meter_id]`, `filter[created_at]`
  - `filter[is_faulty]` (e.g. `filter[is_faulty]=1` to list only faulty readings)
- Sort:
  - `sort=current_reading,utility_meter_id,created_at`

Create/Update payload (`UtilityMeterReadingData`):

| Field | Required | Type | Notes |
|---|---|---|---|
| `current_reading` | Yes | number | Decimal, stored with 5 dp. Must be **≥ the meter's `last_reading`** (a meter cannot read backwards). |
| `reading_image_id` | No | integer | Nullable. Must exist in `uploads.id` when provided. |
| `is_faulty` | No | boolean | Defaults to `false`. Set `true` when the meter is faulty/unreadable. The value is mirrored onto the parent meter's `is_faulty`, so a faulty reading flags the meter and a later clean reading clears it. |
| `fault_reason` | No* | string | Max 500 chars. **Required when `is_faulty` is `true`.** Stored on the reading (fault history) and mirrored onto the meter's `fault_reason` while it stays faulty; cleared to `null` by the next clean reading. |

Example reading payload:

```json
{
    "current_reading": 123.67890,
    "reading_image_id": null,
    "is_faulty": true,
    "fault_reason": "Display cracked and unreadable"
}
```

Each reading exposes `is_faulty` and `fault_reason` in its resource (so faults have a
history). The parent meter resource exposes `is_faulty` and `fault_reason` too
(reflecting the latest reading); while `is_faulty` is `true`, billing should estimate
the meter's consumption rather than trust the recorded reading.

## Permissions

| Action | Permission |
|---|---|
| List / view meter | `view-utility-meter` |
| Create meter | `create-utility-meter` |
| Update meter | `update-utility-meter` (or meter creator) |
| Delete meter | `delete-utility-meter` (or meter creator) |
| Activate / deactivate | `activate-utility-meter` / `deactivate-utility-meter` |
| Submit readings for billing | `update-utility-meter` |
| View reading | `view-utility-meter-reading` |
| Create / update reading | `update-utility-meter-reading` (or reading creator) |
| Delete reading | `delete-utility-meter-reading` (or reading creator) |

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Show `4xx` messages/validation details.
- Show generic fallback for `5xx`.
