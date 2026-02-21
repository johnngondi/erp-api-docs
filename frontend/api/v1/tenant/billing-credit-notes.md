# Tenant Billing Credit Notes API

Base route:

`/api/v1/tenant/billing/credit-notes`

Read-only tenant credit note endpoints.

## Endpoints

- `GET /billing/credit-notes`
- `GET /billing/credit-notes/{creditNote}`

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

