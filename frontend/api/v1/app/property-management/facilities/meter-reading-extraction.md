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
    "image_check": {
      "issue": { "value": "none", "color": "success", "label": "Clear photo", "message": null },
      "flagged": false
    },
    "result": {
      "current_reading": "012345",
      "meter_number": "MTR-99",
      "is_legible": true,
      "feedback": null,
      "image_issue": "none"
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
- `image_issue` — what is wrong with the photo, one of the values below. Anything but `none`
  makes `is_legible` false and the readings null.

### Photo checks (`image_check`, row 200)

The reader first decides whether the photo shows a meter, then whether it can be read.
`image_check.issue` is the enum object (`value`, `color`, `label`, `message`) and
`image_check.flagged` is `true` for anything but `none`. Show `issue.message` (or `result.feedback`,
the model's own wording) and ask for a retake.

| `value` | `label` | `color` | `message` |
| --- | --- | --- | --- |
| `none` | Clear photo | success | `null` |
| `not_a_meter` | Not a meter | danger | This doesn't look like a meter. Retake the photo with the meter's dial and number in the frame. |
| `blurry` | Blurry | warning | The photo is blurry. Hold the phone steady and retake it. |
| `obstructed` | Smudged or obstructed | warning | The dial is smudged, covered or catching glare. Wipe the glass or change the angle, then retake it. |
| `cropped` | Cropped | warning | Part of the dial is cut off. Retake it with the whole dial and the meter number in the frame. |
| `too_dark` | Too dark | warning | The photo is too dark. Use more light or the flash, then retake it. |

The verdict is kept with the photo. A reading saved with that photo (`reading_image_id`) carries it as
`image_issue` (the same enum object) and `image_flagged` on the readings endpoints, so a flagged photo
stays visible after the reading is saved. Both are `null`/`false` for a photo that was never checked.
Saving is not refused for a flagged photo.

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
    "image_check": {
      "issue": { "value": "not_a_meter", "color": "danger", "label": "Not a meter",
                 "message": "This doesn't look like a meter. Retake the photo with the meter's dial and number in the frame." },
      "flagged": true
    },
    "result": {
      "current_reading": null,
      "meter_number": null,
      "is_legible": false,
      "feedback": "This image appears to be a receipt, not a utility meter.",
      "image_issue": "not_a_meter"
    }
  }
}
```

## Notes
- Extraction is synchronous and may take several seconds.
- Always check `is_legible` before using `result`, and `verification.matches_meter`
  before saving the reading against the meter.
- Extraction only reads the image — it never saves a reading. Persist the result via
  `POST /facilities/utility-meters/{meter}/readings` (see the Utility Meters API docs).
- When the meter is faulty or unreadable (e.g. `is_legible: false`, a damaged display,
  or `feedback` indicating a problem) and the user cannot capture a reliable value,
  save the reading with `is_faulty: true` and a `fault_reason` (required when faulty —
  the `feedback` text is a good default). This flags the parent meter as faulty so the
  billing run estimates its consumption instead of using the recorded reading. Both the
  flag and reason are kept on the reading for history, and are cleared automatically by
  the next clean (non-faulty) reading.

## Errors
- `422 Unprocessable Entity` — `upload_id` missing or not an existing upload.
- `403 Forbidden` — the authenticated user cannot create meter readings (authenticated endpoint only).
- `500 Internal Server Error` — extraction failed (e.g. unsupported file type, or the AI provider errored).
