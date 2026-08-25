# Tenant Invoice Dispute API

Base route:

`/api/v1/tenant/billing/invoices`

## Endpoint

- `POST /billing/invoices/{invoice}/dispute`
- `DELETE /billing/invoices/{invoice}/dispute`

Allows tenant to dispute an invoice and open a support ticket.

## Request Body

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

## Behavior

- Ticket title is generated from invoice details.
- Ticket description includes invoice info and submitted details.
- Ticket category is resolved to billing category when available, else first category.
- If `ticket_upload_id` is provided, upload is attached to the created ticket.

## Example Request

```json
{
  "details": "Invoice has an incorrect service charge.",
  "ticket_upload_id": 10
}
```

## Success Response

```json
{
  "data": {
    "message": "Invoice dispute submitted successfully.",
    "ticket": {
      "id": 25,
      "title": "Invoice Dispute: INV-1001"
    }
  }
}
```

## Access Rules

- Tenant can only dispute invoices from tenant-owned leases.
- Cross-tenant invoice dispute returns `404`.

## Withdraw Dispute

`DELETE /billing/invoices/{invoice}/dispute`

Withdraws an existing dispute for the invoice.

### Behavior

- Clears `disputed_at`, `dispute_details`, `disputed_by`, and `dispute_ticket_id` on the invoice.
- Marks the related dispute ticket as `closed`.

### Success Response

```json
{
  "data": {
    "message": "Invoice dispute withdrawn successfully."
  }
}
```

