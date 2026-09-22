# Credit Notes API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/credit-notes`

## Endpoints

- `GET /credit-notes`
- `POST /credit-notes`
- `GET /credit-notes/{creditNote}`
- `PUT/PATCH /credit-notes/{creditNote}`
- `PATCH /credit-notes/{creditNote}/cancel`
- `POST /credit-notes/{creditNote}/sign`
- `POST /credit-notes/{creditNote}/dispute`
- `DELETE /credit-notes/{creditNote}/dispute`
- `DELETE /credit-notes/{creditNote}`

## List Credit Notes

`GET /api/v1/app/{company}/property-management/lease-management/credit-notes`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_id]`
  - `filter[invoice_id]`
  - `filter[credit_note_id]`
  - `filter[due_at]`
  - `filter[cu_reference_number]`
  - `filter[is_credit]`
  - `filter[paid]`
  - `filter[cu_invoice_number]`
  - `filter[status]`
- Sort:
  - `sort=lease_id,invoice_id,credit_note_id,due_at,cu_reference_number,is_credit,paid,cu_invoice_number,status`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/credit-notes?filter[lease_id]=101&sort=-due_at`

## Create Credit Note

`POST /api/v1/app/{company}/property-management/lease-management/credit-notes`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | - |
| `due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `notes` | Yes | string | - |
| `items` | Yes | array | At least one item |
| `invoice_id` | No | integer | Optional |
| `cu_reference_number` | No | string | Optional |
| `is_credit` | No | boolean | Defaults to `true` |
| `paid` | No | number | Defaults to `0` |
| `cu_invoice_number` | No | string | Optional |
| `cu_serial_number` | No | string | Optional |
| `cu_credit_note_verify_url` | No | string | Optional |

`items[]` fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_item_component_id` | Yes | integer | Must belong to supplied lease |
| `notes` | Yes | string | - |
| `quantity` | Yes | integer | Minimum `1` |
| `amount` | Yes | number | Minimum `0` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |

Notes:

- `lease_item_component_id` must be unique within the same request.
- Response exposes `cu_invoice_verify_url` in resource output.

Example request:

```json
{
  "lease_id": 101,
  "due_at": "2026-03-01",
  "notes": "Overcharge adjustment",
  "items": [
    {
      "lease_item_component_id": 455,
      "notes": "Base rent correction",
      "quantity": 1,
      "amount": 120,
      "tax_id": 1
    }
  ]
}
```

Example response:

```json
{
  "message": "Credit note created successfully.",
  "credit_note": {
    "id": 410,
    "total": 139.2,
    "status": { "value": "pending", "color": "secondary" }
  }
}
```

## Update Credit Note

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}`

Use the same payload shape as create.

## Cancel Credit Note

`PATCH /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/cancel`

No request body required.

Possible status values returned by API resource:

- `pending` (`secondary`)
- `applied` (`success`)
- `cancelled` (`danger`)

## Sign Credit Note (ETR)

`POST /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/sign`

Signs the credit note on the property's ETR (KRA fiscal) device and stores the CU details on the credit note. No request body.

The device comes from the property's `esd_type` / `esd_config` (see Facilities). A property with no device set uses the system default (Incotex). The seller PIN sent to the device is always the **landlord's KRA PIN**, and the buyer PIN is the tenant's.

When to show the action: `permissions.sign` on the credit note resource is `true`. It is `false` when:

- the user lacks the `sign-facility-credit-note` permission
- the credit note is `pending` (awaiting approval) or `cancelled`
- the credit note is already signed (`cu_invoice_number` is set)
- it is linked to an invoice that is not signed yet (sign the invoice first), or it has neither a linked invoice nor a `cu_reference_number`

Signing runs as a queued job. On a synchronous queue (the default locally) the request returns after the device answers; on a real queue it returns straight away and the CU fields fill in once the job runs. Refetch the credit note (or poll `GET /credit-notes/{creditNote}`) until `etr_signed_at` or `etr_error` is set.

Success response (`200`):

```json
{
  "message": "Credit note signed successfully",
  "credit_note": {
    "id": 9001,
    "cu_invoice_number": "0010000042",
    "cu_serial_number": "KRAMW017202209000001",
    "cu_credit_note_verify_url": "https://itax.kra.go.ke/KRA-Portal/invoiceChk.htm?actionCode=loadPage&invoiceNo=0010000042",
    "etr_signed_at": {
      "raw": "2026-09-22T10:15:00.000000Z",
      "formatted": "22 Sep, 2026 10:15",
      "diff": "1 second ago"
    },
    "etr_error": null,
    "permissions": { "sign": false }
  }
}
```

When the job is queued rather than run inline, `message` is `"Credit note queued for ETR signing"` and the CU fields are still `null`.

Errors:

| Status | When | Body |
|---|---|---|
| `403` | `permissions.sign` is `false` | standard forbidden response |
| `422` | The device refused the credit note, could not be reached, or the credit note cannot be signed as it stands | `{ "message": "...", "errors": { "etr": ["<reason>"] } }` |

Typical `errors.etr` reasons:

- `The landlord of this property has no KRA PIN.`: add the PIN on the landlord, then sign again.
- `No URL is configured for the incotex ETR device.`: set the property's `esd_config.url` (or ask for the default device to be configured).
- `The incotex ETR device rejected the document (HTTP 4xx): ...`: the device's own message follows.
- `The credit note has no signed invoice to reference.`: sign the original invoice first.

The last failure is also saved on the credit note as `etr_error`, so show it next to the Sign button. Signing again clears it.

ETR fields on the credit note resource:

| Field | Type | Notes |
|---|---|---|
| `cu_invoice_number` | string\|null | CU invoice number from the device |
| `cu_serial_number` | string\|null | CU serial number |
| `cu_credit_note_verify_url` | string\|null | KRA verification URL (render as a QR code) |
| `etr_signed_at` | object\|null | `{ raw, formatted, diff }` when the system signed it |
| `etr_error` | string\|null | Reason the last signing attempt failed |

## Delete Credit Note

`DELETE /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}`

## Dispute Credit Note

`POST /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/dispute`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

## Withdraw Credit Note Dispute

`DELETE /api/v1/app/{company}/property-management/lease-management/credit-notes/{creditNote}/dispute`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
