# Tenant Billing Invoices API

Base route:

`/api/v1/tenant/billing/invoices`

Read-only tenant invoice endpoints.

## Endpoints

- `GET /billing/invoices`
- `GET /billing/invoices/{invoice}`

## List Invoices

`GET /api/v1/tenant/billing/invoices`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_id]`
  - `filter[due_at]`
  - `filter[is_credit]`
  - `filter[amount]`
  - `filter[total]`
  - `filter[paid]`
  - `filter[cu_invoice_number]`
  - `filter[status]`
- Sort:
    - `sort=id,lease_id,due_at,amount,paid,total,status,created_at,cu_invoice_number`
- Pagination:
  - `per_page`
  - `page`

## Access Rules

- Tenant can only view invoices linked to their own leases.
- Accessing another tenant invoice returns `404`.

