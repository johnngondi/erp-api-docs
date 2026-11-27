# Attendance Channels (Frontend)

An attendance channel is one way of recording attendance at an office:

- **Biometric**: a fingerprint or face device, read through a service (BioTime). The channel stores
  the service's connection details.
- **QR code**: a code employees scan with their phone; a scan counts only within `radius_meters`
  of the channel's location.

Recording attendance itself comes with the next ticket. This page is the setup: channels, the
service catalogue, testing a device connection, and the ID each employee has on a device.

Base: `/api/v1/app/{company}/human-resource/admin`

| Endpoint | Permission |
| --- | --- |
| `GET attendance-channels` | `view-attendance-channel` |
| `GET attendance-channels/summary` | `view-attendance-channel` |
| `GET attendance-channels/{channel}` | `view-attendance-channel` |
| `POST attendance-channels` | `create-attendance-channel` |
| `PATCH attendance-channels/{channel}` | `update-attendance-channel` |
| `DELETE attendance-channels/{channel}` | `delete-attendance-channel` |
| `PATCH attendance-channels/{channel}/activate`, `.../deactivate` | `update-attendance-channel` |
| `POST attendance-channels/{channel}/test-connection` | `update-attendance-channel` |
| `GET attendance-services` | `view-attendance-channel` |
| `GET attendance-channels/{channel}/employees` | `view-attendance-channel` |
| `PUT attendance-channels/{channel}/employees` | `update-attendance-channel` |
| `DELETE attendance-channels/{channel}/employees/{employee}` | `update-attendance-channel` |

`GET /me/permissions` carries `view-attendance-channel` and `create-attendance-channel`. For the
row buttons, read the channel's own `permissions`: `update`, `delete`, `activate`, `deactivate`
and `testConnection` are `true` only when the caller may and the action would work (for example
`activate` only on an inactive channel, `testConnection` only on a biometric one).

## The channel

```json
{
  "id": 4,
  "name": "HQ main gate",
  "code": "BIO-HQ-01",
  "company_office_id": 2,
  "company_office": { "id": 2, "name": "Head Office" },
  "type": { "value": "biometric", "color": "primary", "label": "Biometric" },
  "status": { "value": "active", "color": "success", "label": "Active" },
  "allowed_events": [
    { "value": "check_in", "color": "success", "label": "Check In" },
    { "value": "check_out", "color": "secondary", "label": "Check Out" }
  ],
  "service": { "value": "biotime", "color": "primary", "label": "BioTime" },
  "config": {
    "url": "https://biotime.example.com",
    "username": "admin",
    "has_password": true,
    "terminal_sn": "CQZ7232460012"
  },
  "qr_refresh_seconds": null,
  "latitude": null,
  "longitude": null,
  "radius_meters": null,
  "last_synced_at": null,
  "last_activity_at": "2026-11-27T08:02:11+03:00",
  "last_error": null,
  "last_error_at": null,
  "employees_count": 18,
  "permissions": { "view": true, "update": true, "delete": true, "activate": false, "deactivate": true, "testConnection": true }
}
```

- **The password is never returned.** `config.has_password` says whether one is stored. Show the
  password input empty with a "stored" hint.
- `employees_count`: employees with a device ID on this channel.
- A QR channel has `service` and `config` `null` and its location filled; a biometric channel the
  reverse.
- Health: `last_activity_at` is the device's last activity as of the last successful connection
  test; `last_error` and `last_error_at` are the last failed test, cleared by a successful one.
  `last_synced_at` is filled by the BioTime sync (next tickets).

## List

`GET attendance-channels`

| Parameter | Kind | Values |
| --- | --- | --- |
| `filter[search]` | partial | Name or code |
| `filter[type]` | exact | `biometric`, `qr_code` |
| `filter[status]` | exact | `active`, `inactive` |
| `filter[service]` | exact | `biotime` |
| `filter[company_office_id]` | exact | Office id |
| `sort` | | `name`, `code`, `created_at`, `last_activity_at` (`-` for descending). Default: newest first. |
| `include` | | `companyOffice` (already loaded on the list) |
| `per_page`, `page` | | Paginated |

## Summary

`GET attendance-channels/summary`

```json
{
  "data": {
    "channels": { "total": 5, "active": 4, "inactive": 1 },
    "biometric": 3,
    "qr_code": 2,
    "employees_without_device_id": 7,
    "unmatched_scans_today": null
  }
}
```

- `biometric` and `qr_code` count every channel of that type, active or not.
- `employees_without_device_id`: employees on probation or active whose office has an active
  biometric channel, and who have a device ID on none of that office's active biometric channels.
  One device ID per office is enough.
- `unmatched_scans_today` stays `null` until the BioTime sync ticket.

## Service catalogue

`GET attendance-services`

Build the biometric connection form from this, not from hard-coded fields.

```json
{
  "data": {
    "services": [
      {
        "value": "biotime",
        "label": "BioTime",
        "available": true,
        "fields": [
          { "name": "url", "label": "Server URL", "type": "url", "required": true, "secret": false },
          { "name": "username", "label": "Username", "type": "text", "required": true, "secret": false },
          { "name": "password", "label": "Password", "type": "password", "required": true, "secret": true },
          { "name": "terminal_sn", "label": "Device serial number", "type": "text", "required": true, "secret": false }
        ]
      }
    ]
  }
}
```

- `available: false` means PropERP cannot reach that service yet (its connector is the BioTime
  ticket). A channel can still be set up; its connection test says the service is not connected.
- A `secret` field is write-only (see Update).

## Create

`POST attendance-channels`

Biometric:

```json
{
  "name": "HQ main gate",
  "code": "BIO-HQ-01",
  "company_office_id": 2,
  "type": "biometric", // biometric, qr_code
  "allowed_events": ["check_in", "check_out"], // check_in, check_out, movement
  "service": "biotime", // biotime
  "config": {
    "url": "https://biotime.example.com",
    "username": "admin",
    "password": "s3cret",
    "terminal_sn": "CQZ7232460012"
  }
}
```

QR code:

```json
{
  "name": "Reception QR",
  "code": "QR-HQ-01",
  "company_office_id": 2,
  "type": "qr_code", // biometric, qr_code
  "allowed_events": ["check_in", "check_out", "movement"], // check_in, check_out, movement
  "latitude": -1.286389,
  "longitude": 36.817223,
  "radius_meters": 100,
  "qr_refresh_seconds": 30 // 30, 60, or null for a printed code
}
```

| Field | Rules |
| --- | --- |
| `name` | Required, up to 150 characters |
| `code` | Required, up to 50: letters, numbers, `-`, `/`, `_`. Unique in the company (`Another channel of this company already uses this code.`). |
| `company_office_id` | Required, an office of this company |
| `type` | Required, `biometric` or `qr_code`. **Fixed once created.** |
| `allowed_events` | Required, at least one of `check_in`, `check_out`, `movement`, no repeats |
| `service` | Biometric: required. QR: refused (`A QR code channel has no service.`). |
| `config` | Biometric: required, with every `required` field of the service (from the catalogue): `config.url` (a URL), `config.username`, `config.password`, `config.terminal_sn`. Other keys are dropped. QR: refused. |
| `latitude`, `longitude` | QR: required, -90 to 90 and -180 to 180. Biometric: refused (`Only a QR code channel has a location.`). |
| `radius_meters` | QR: required, 20 to 1000. Biometric: refused. |
| `qr_refresh_seconds` | QR: `30`, `60` or `null` (a printed code that never changes). Biometric: refused. |

A new channel is `active`. Answers `200` with `message` and `attendance_channel`.

## Update

`PATCH attendance-channels/{channel}`

Send only what changes; a key left out keeps its value.

- **Password:** send `config` without `password` (or with it empty) and the stored one is kept.
  Send a new one to replace it.
- When `config` is sent, it replaces the stored config: send every non-secret field, as the form
  holds them.
- Changing `service` needs its whole `config`, password included.
- `type` cannot change (`The type of a channel cannot be changed. Create a new channel instead.`).
- Wrong-type fields are refused as on create.

## Activate, deactivate

`PATCH attendance-channels/{channel}/activate` and `.../deactivate`, no body.

An inactive channel keeps its setup and device IDs but records nothing. Activating an active channel
(or the reverse) is a `422` on `status`: `This channel is already active.` /
`This channel is already inactive.`

## Delete

`DELETE attendance-channels/{channel}`

Removes the channel and its device IDs. A channel that has attendance records cannot be deleted
(`422` on `attendance_channel`: `This channel has attendance records, so it cannot be deleted. Deactivate it instead.`);
records arrive with the next ticket, so until then every channel can be deleted.

## Test connection

`POST attendance-channels/{channel}/test-connection`, no body. Biometric channels only (`422` on
`type` otherwise: `Only a biometric channel connects to a device.`).

Answers `200` whether or not the device was reached:

```json
{
  "data": {
    "message": "The device answered.",
    "connection": {
      "reachable": true,
      "device_name": "BioTime CQZ7232460012",
      "last_activity_at": "2026-11-27T08:02:11+03:00",
      "message": null
    },
    "attendance_channel": { "...": "the channel, health fields updated" }
  }
}
```

When not reachable: `message` is `The device could not be reached.`, `connection.reachable` is
`false` and `connection.message` says why (for example
`BioTime is not connected to PropERP yet, so the device cannot be reached.`). The channel's
`last_error` keeps that reason until a test succeeds.

## Device IDs

Each employee is known to a biometric device by an ID (the number enrolled on the device). Map
them here so punches can be matched to employees.

### List

`GET attendance-channels/{channel}/employees`

Employees based at the channel's office (on probation, active or suspended), with or without a
device ID, plus anyone else who already has one on this channel.

```json
{
  "data": [
    {
      "id": 75,
      "staff_no": "NWR-0075",
      "name": "Amina Hassan",
      "job_title": "Accountant",
      "status": { "value": "active", "color": "success", "label": "Active" },
      "company_office_id": 2,
      "device_user_code": "101",
      "mapped": true,
      "based_here": true
    }
  ],
  "links": {}, "meta": {}
}
```

- `based_here: false`: mapped on this channel but based at another office (or no longer
  employed). Show it, so the mapping can be removed.
- Filters: `filter[search]` (staff no., names, job title, work email), `filter[mapped]=true|false`.
- Sorts: `first_name` (default), `staff_no`, `device_user_code`. Paginated.

### Save

`PUT attendance-channels/{channel}/employees`

```json
{
  "employees": [
    { "employee_id": 75, "device_user_code": "101" },
    { "employee_id": 76, "device_user_code": "102" }
  ]
}
```

- Gives or changes the device IDs sent. Employees not in the list keep theirs.
- Any employee of the company who is not a draft or terminated may be mapped, wherever they are
  based.
- One ID per employee and one employee per ID on a channel. A code another employee holds is a
  `422` on that row: `employees.0.device_user_code`:
  `Device ID 101 is already given to Amina Hassan on this channel.` Two employees can swap codes in
  one request.
- An employee or code twice in the list is a `422` on those rows.
- Up to 500 rows per request.

Answers `200`: `{ "message": "2 device IDs saved.", "employees_count": 18 }`.

### Remove

`DELETE attendance-channels/{channel}/employees/{employee}`

Answers `200` (`Device ID removed.`), or `422` on `employee` when they have none on this channel.

## Errors

- `403` without the endpoint's permission.
- `404` for another company's channel or employee.
