# Tenant Billing Receipts API

Base route:

`/api/v1/tenant/billing/receipts`

Tenant receipt endpoints.

## Endpoints

- `GET /billing/receipts`
- `GET /billing/receipts/{receipt}`
- `POST /billing/receipts/{receipt}/dispute`
- `DELETE /billing/receipts/{receipt}/dispute`

## List Receipts

`GET /api/v1/tenant/billing/receipts`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[receiving_account_id]`
  - `filter[transaction_number]`
  - `filter[transaction_date]`
  - `filter[payment_method_id]`
  - `filter[paying_user_id]`
  - `filter[amount]`
  - `filter[allocated]`
  - `filter[balance]`
  - `filter[receiving_user_id]`
- Sort:
  - `sort=id,receiving_account_id,transaction_number,transaction_date,payment_method_id,paying_user_id,amount,allocated,balance,receiving_user_id`
- Pagination:
  - `per_page`
  - `page`

## Access Rules

- Tenant can view receipts paid by tenant or allocated to tenant invoices.
- Accessing unrelated receipts returns `404`.

## Dispute Receipt

`POST /api/v1/tenant/billing/receipts/{receipt}/dispute`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `details` | Yes | string | Dispute description |
| `ticket_upload_id` | No | integer | Optional attachment upload id (`uploads.id`) |

Behavior:

- Creates a billing ticket and links it to the receipt dispute fields.
- Cancelled receipts cannot be disputed.
- Cross-tenant access returns `404`.

## Withdraw Receipt Dispute

`DELETE /api/v1/tenant/billing/receipts/{receipt}/dispute`

Behavior:

- Closes the linked dispute ticket.
- Clears `disputed_at`, `dispute_details`, `disputed_by`, and `dispute_ticket_id`.
- Cross-tenant access returns `404`.
