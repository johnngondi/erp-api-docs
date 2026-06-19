# Utility Meter Reading Extraction (AI) API

Reads a utility meter from an uploaded image with Claude and returns the
current reading and the meter number (when visible). The recognised meter
number is verified against the meter the reading is being captured for.

## Endpoint

### Extract a reading for a meter
`POST /api/v1/app/{company}/property-management/facilities/utility-meters/{meter}/extract-reading`

Upload the meter photo first via the file-management uploads endpoint, then
send its id here.

Request body:
- `upload_id` (integer, required) — id of the uploaded meter image.

Response:
```json
{
  "data": {
    "message": "Meter read successfully",
    "verification": {
      "matches_meter": true,
      "expected_meter_number": "MTR-99",
      "recognised_meter_number": "MTR-99"
    },
    "result": {
      "current_reading": "012345",
      "meter_number": "MTR-99",
      "is_legible": true,
      "feedback": null
    }
  }
}
```

Field reference (`result`):
- `current_reading` — the reading exactly as displayed, returned as a string to
  preserve leading zeros and decimals. `null` when unreadable.
- `meter_number` — the meter number if visible on the image; otherwise `null`.
- `is_legible` — `false` when the image could not be read; see `feedback`.
- `feedback` — user-facing message (e.g. "image too blurry"); `null` when legible.

Verification (`verification`):
- `matches_meter` — `true` when the recognised meter number matches the target
  meter (compared ignoring spaces and case).
- `expected_meter_number` — the meter's number on record.
- `recognised_meter_number` — the meter number read from the image (may be `null`).

### Not-legible response
`200 OK` is still returned with null readings and an explanation:
```json
{
  "data": {
    "message": "Meter read successfully",
    "verification": {
      "matches_meter": false,
      "expected_meter_number": "MTR-99",
      "recognised_meter_number": null
    },
    "result": {
      "current_reading": null,
      "meter_number": null,
      "is_legible": false,
      "feedback": "The meter display is not clearly visible. Please retake the photo straight-on with better lighting."
    }
  }
}
```

## Temporary test endpoint (no auth)

> **For development/testing only — will be removed before release.**

`POST /api/v1/settings/file-management/uploads/test-read-meter`

Unauthenticated and not bound to a meter — just send an upload and get the
reading back (no `verification` block). Use it to smoke-test the vision
pipeline without going through auth/tenancy.

Request body:
- `upload_id` (integer, required)

Response:
```json
{
  "data": {
    "message": "Meter read successfully",
    "result": {
      "current_reading": "012345",
      "meter_number": "MTR-99",
      "is_legible": true,
      "feedback": null
    }
  }
}
```

## Notes
- Extraction is synchronous and may take several seconds.
- Always check `is_legible` before using `result`, and `verification.matches_meter`
  before saving the reading against the meter.

## Errors
- `422 Unprocessable Entity` — `upload_id` missing or not an existing upload.
- `403 Forbidden` — the authenticated user cannot create meter readings (authenticated endpoint only).
- `500 Internal Server Error` — extraction failed (e.g. unsupported file type, or the AI provider errored).
