# Tenant Billing Credit Notes API

Base route:

`/api/v1/tenant/billing/credit-notes`

Tenant credit note endpoints.

## Endpoints

- `GET /billing/credit-notes`
- `GET /billing/credit-notes/{creditNote}`
- `POST /billing/credit-notes/{creditNote}/dispute`
- `DELETE /billing/credit-notes/{creditNote}/dispute`

## List Credit Notes

`GET /api/v1/tenant/billing/credit-notes`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_id]`
  - `filter[invoice_id]`
  - `filter[due_at]`
  - `filter[cu_reference_number]`
  - `filter[cu_invoice_number]`
  - `filter[status]`
  - `filter[amount]`
  - `filter[paid]`
  - `filter[total]`
- Sort:
  - `sort=id,lease_id,invoice_id,due_at,amount,paid,total,status,created_at`
- Pagination:
  - `per_page`
  - `page`

## Supporting document

List and show return the document staff attached when raising the credit note, the same shape as
a receipt's `pop_upload`. Read-only for the tenant.

| Field | Type | Notes |
|---|---|---|
| `supporting_upload_id` | integer\|null | - |
| `supporting_upload` | object\|null | `{ id, title, description, source_url, thumbnail_url, file_name, type, extension, size, created }`. Open `source_url` to view the file |

## Access Rules

- Tenant can only view credit notes linked to their own leases.
- Accessing another tenant credit note returns `404`.

## Dispute Credit Note

`POST /api/v1/tenant/billing/credit-notes/{creditNote}/dispute`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

Behavior:

- Creates a billing ticket and links it to the credit note dispute fields.
- Cancelled credit notes cannot be disputed.
- Only an open dispute blocks a new one (`422` keyed `credit_note`, naming the ticket, with
  `open_ticket: { "id": … }`): `disputed_at`
  is set, or the linked ticket is not `closed`. Once staff close the ticket the credit note can be
  disputed again, on a new ticket; the closed one stays as the first dispute's history.
- A credit note is disputed while `disputed_at.raw` is set. `disputed_at` is always sent, with `raw: null`
  when there is no open dispute, so read `raw`, not whether the key is there. `dispute_ticket` stays
  after a dispute is settled and carries its `status` (`open` / `closed`), so a closed one can be shown
  as history next to the Dispute action. `permissions.dispute` follows the credit note's status only, so the Dispute button stays while a
  dispute is open and pressing it shows the `422`.
- Cross-tenant access returns `404`.

## Withdraw Credit Note Dispute

`DELETE /api/v1/tenant/billing/credit-notes/{creditNote}/dispute`

Behavior:

- Closes the linked dispute ticket.
- Clears `disputed_at`, `dispute_details`, `disputed_by`, and `dispute_ticket_id`.
- Cross-tenant access returns `404`.
