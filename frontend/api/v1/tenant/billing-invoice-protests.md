# Tenant Invoice Protest API

Base route:

`/api/v1/tenant/billing/invoices`

## Endpoint

- `POST /billing/invoices/{invoice}/protest`
- `DELETE /billing/invoices/{invoice}/protest`

Allows tenant to protest an invoice and open a support ticket.

## Request Body

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Protest description |
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
    "message": "Invoice protest submitted successfully.",
    "ticket": {
      "id": 25,
      "title": "Invoice Protest: INV-1001"
    }
  }
}
```

## Access Rules

- Tenant can only protest invoices from tenant-owned leases.
- Cross-tenant invoice protest returns `404`.

## Withdraw Protest

`DELETE /billing/invoices/{invoice}/protest`

Withdraws an existing protest for the invoice.

### Behavior

- Clears `protested_at`, `protest_details`, `protested_by`, and `protest_ticket_id` on the invoice.
- Marks the related protest ticket as `closed`.

### Success Response

```json
{
  "data": {
    "message": "Invoice protest withdrawn successfully."
  }
}
```
