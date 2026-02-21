# Tenant Billing Receipts API

Base route:

`/api/v1/tenant/billing/receipts`

Read-only tenant receipt endpoints.

## Endpoints

- `GET /billing/receipts`
- `GET /billing/receipts/{receipt}`

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

