# Invoices API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/invoices`

## Endpoints

- `GET /invoices`
- `POST /invoices`
- `GET /invoices/{invoice}`
- `PUT/PATCH /invoices/{invoice}`
- `PATCH /invoices/{invoice}/cancel`
- `POST /invoices/{invoice}/sign`
- `POST /invoices/{invoice}/dispute`
- `DELETE /invoices/{invoice}/dispute`
- `DELETE /invoices/{invoice}`

## List Invoices

`GET /api/v1/app/{company}/property-management/lease-management/invoices`

Supported query params:

- Filters:
  - `filter[search]` — free text
  - **Exact** (`=`): `filter[lease_id]`, `filter[status]`, `filter[facility_id]` (through the
    lease), `filter[user_id]` (the lease's tenant)
  - **Partial** (`LIKE`): `filter[due_at]`, `filter[created_at]`
- Sort:
  - `sort=id,due_at,created_at,amount,tax,total,paid,balance`
- Include: not supported
> **Exact versus partial matters here.** `lease_id` and `status` are exact. They were declared as
> bare strings, which Spatie turns into a *partial* match — so `filter[lease_id]=6` returned leases
> 6, 16, 26, 60-69 and anything else whose id contains a 6, putting another tenant's billing on a
> lease's page. `filter[status]=paid` also matched `unpaid` and `partially paid`.
>
> Dates stay partial on purpose, so `filter[created_at]=2026-09` means "that month".

- Select fields: not supported
- Pagination: `per_page`, `page`

Examples:

- `GET /api/v1/app/12/property-management/lease-management/invoices?filter[lease_id]=101&sort=-due_at&per_page=10`
- `GET /api/v1/app/12/property-management/lease-management/invoices?filter[status]=pending`

## Create Invoice

`POST /api/v1/app/{company}/property-management/lease-management/invoices`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | - |
| `due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `notes` | Yes | string | - |
| `items` | Yes | array | At least one item |
| `cu_reference_number` | No | string | Optional |
| `is_credit` | No | boolean | Defaults to `false` |
| `paid` | No | number | Defaults to `0` |
| `cu_invoice_number` | No | string | Optional |
| `cu_serial_number` | No | string | Optional |
| `cu_invoice_verify_url` | No | string | Optional |

`items[]` fields:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_item_component_id` | Yes | integer | Must belong to the supplied lease |
| `notes` | Yes | string | - |
| `quantity` | Yes | integer | Minimum `1` |
| `amount` | Yes | number | Minimum `0` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |

Notes:

- `lease_item_component_id` must be unique within the same request.

Example request:

```json
{
  "lease_id": 101,
  "due_at": "2026-03-01",
  "notes": "March 2026 billing",
  "cu_reference_number": "INV-REF-3001",
  "items": [
    {
      "lease_item_component_id": 455,
      "notes": "Base rent",
      "quantity": 1,
      "amount": 950,
      "tax_id": 1
    }
  ]
}
```

Example response:

```json
{
  "message": "Invoice created successfully",
  "invoice": {
    "id": 9001,
    "lease": { "id": 101 },
    "amount": 950,
    "tax": 152,
    "total": 1102,
    "status": { "value": "pending", "color": "secondary" }
  }
}
```

## Update Invoice

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}`

Use the same payload shape as create.

## Cancel Invoice

`PATCH /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/cancel`

No request body required.

Possible status values returned by API resource:

- `pending` (`secondary`)
- `unpaid` (`warning`)
- `partially paid` (`primary`)
- `paid` (`success`)
- `cancelled` (`danger`)

## Sign Invoice (ETR)

`POST /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/sign`

Signs the invoice on the property's ETR (KRA fiscal) device and stores the CU details on the invoice. No request body.

The device comes from the property's `esd_type` / `esd_config` (see Facilities). A property with no device set uses the system default (Incotex). The seller PIN sent to the device is always the **landlord's KRA PIN**, and the buyer PIN is the tenant's.

When to show the action: `permissions.sign` on the invoice resource is `true`. It is `false` when:

- the user lacks the `sign-facility-invoice` permission
- the invoice is `pending` (awaiting approval) or `cancelled`
- the invoice is already signed (`cu_invoice_number` is set)

Signing runs as a queued job. On a synchronous queue (the default locally) the request returns after the device answers; on a real queue it returns straight away and the CU fields fill in once the job runs. Refetch the invoice (or poll `GET /invoices/{invoice}`) until `etr_signed_at` or `etr_error` is set.

Success response (`200`):

```json
{
  "message": "Invoice signed successfully",
  "invoice": {
    "id": 9001,
    "cu_invoice_number": "0010000042",
    "cu_serial_number": "KRAMW017202209000001",
    "cu_invoice_verify_url": "https://itax.kra.go.ke/KRA-Portal/invoiceChk.htm?actionCode=loadPage&invoiceNo=0010000042",
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

When the job is queued rather than run inline, `message` is `"Invoice queued for ETR signing"` and the CU fields are still `null`.

Errors:

| Status | When | Body |
|---|---|---|
| `403` | `permissions.sign` is `false` | standard forbidden response |
| `422` | The device refused the invoice, could not be reached, or the invoice cannot be signed as it stands | `{ "message": "...", "errors": { "etr": ["<reason>"] } }` |

Typical `errors.etr` reasons:

- `The landlord of this property has no KRA PIN.`: add the PIN on the landlord, then sign again.
- `No URL is configured for the incotex ETR device.`: set the property's `esd_config.url` (or ask for the default device to be configured).
- `The incotex ETR device rejected the document (HTTP 4xx): ...`: the device's own message follows.

The last failure is also saved on the invoice as `etr_error`, so show it next to the Sign button. Signing again clears it.

ETR fields on the invoice resource:

| Field | Type | Notes |
|---|---|---|
| `cu_invoice_number` | string\|null | CU invoice number from the device |
| `cu_serial_number` | string\|null | CU serial number |
| `cu_invoice_verify_url` | string\|null | KRA verification URL (render as a QR code) |
| `etr_signed_at` | object\|null | `{ raw, formatted, diff }` when the system signed it |
| `etr_error` | string\|null | Reason the last signing attempt failed |

## Delete Invoice

`DELETE /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}`

One endpoint, three outcomes, decided by the invoice.

| Invoice | Outcome |
|---|---|
| **Payments allocated** | Refused. `permissions.delete` is `false`, so hide the action; a direct call returns **403**. The money is reallocated to another invoice first |
| **Issued and carrying a CU number** | **Reversed**, not deleted. A credit note for the full value is raised and signed to ETR. The invoice survives |
| **Pending, or no CU number** | Cancelled and removed, along with its tenant statement entries and lease billings |

A **pending** invoice is always removed outright, even if it carries a CU number: it is a draft the
tenant never received, so there is nothing filed to reverse.

Response:

| Field | Notes |
|---|---|
| `action` | `"cancelled"` or `"reversed"`. Always present — switch on this |
| `credit_note_id` | The reversing credit note. **Absent** when nothing was reversed, because null values are stripped from responses |

### Reversed invoices read as "paid"

Applying a full credit note drives the balance to zero, and the status follows the balance — so a
reversed invoice comes to rest as `paid` when nobody paid anything.

`reversal_credit_note_id` on the invoice is what tells the two apart, and
**`filter[reversed]=true|false`** filters on it. Combine it to get real revenue:

```
filter[status]=paid&filter[reversed]=false
```

An invoice counts as reversed only when a credit note covers its **full total**, whether the
credit note came from deleting or cancelling a filed invoice or was raised by hand. A partial credit
note leaves `reversal_credit_note_id` unchanged. Cancelling an **unsigned** reversing credit note
clears the field and reopens the invoice.

The same filter exists on the credit notes list, and a reversing credit note carries
`reversed_invoice_id` (`null` on a partial credit). The credit note's `invoice_id` is the only
stored link. `reversed_invoice_id` is worked out from it and the invoice's
`reversal_credit_note_id`.

The statement line reads `CN#{id} - Reversal for INV#{id} - {invoice notes}`.

### Invoices re-issued from a cancelled credit note

A credit note signed on the ETR cannot be withdrawn. Cancelling one raises a new invoice for the
same amount instead (see the credit notes doc). That invoice carries
**`reissued_from_credit_note_id`** (`null` on every other invoice), and
**`filter[reissued]=true|false`** filters on it. Its notes and statement line read
`INV#{id} - Re-issued for cancelled ETR signed CN#{creditNoteId} - {credit note notes}`.

## Dispute Invoice

`POST /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/dispute`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

## Withdraw Invoice Dispute

`DELETE /api/v1/app/{company}/property-management/lease-management/invoices/{invoice}/dispute`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
