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
- Cross-tenant access returns `404`.

## Withdraw Credit Note Dispute

`DELETE /api/v1/tenant/billing/credit-notes/{creditNote}/dispute`

Behavior:

- Closes the linked dispute ticket.
- Clears `disputed_at`, `dispute_details`, `disputed_by`, and `dispute_ticket_id`.
- Cross-tenant access returns `404`.
