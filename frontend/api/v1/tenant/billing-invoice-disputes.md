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
- Only an open dispute blocks a new one (`422` keyed `invoice`, naming the ticket, with
  `open_ticket: { "id": … }`): `disputed_at` is
  set, or the linked ticket is not `closed`. Once staff close the ticket the invoice can be disputed
  again, on a new ticket; the closed one stays as the first dispute's history.
- A invoice is disputed while `disputed_at.raw` is set. `disputed_at` is always sent, with `raw: null`
  when there is no open dispute, so read `raw`, not whether the key is there. `dispute_ticket` stays
  after a dispute is settled and carries its `status` (`open` / `closed`), so a closed one can be shown
  as history next to the Dispute action. `permissions.dispute` follows the invoice's status only, so the Dispute button stays while a
  dispute is open and pressing it shows the `422`.

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

