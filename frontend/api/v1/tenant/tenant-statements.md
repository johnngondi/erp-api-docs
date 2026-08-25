# Tenant Statements API

Base route:

`/api/v1/tenant/tenant-statements`

These endpoints are read-only for tenant users.

## Endpoints

- `GET /tenant-statements` (balance + transactions)
- `GET /tenant-statements/{tenantStatement}` (single transaction)

## List Statements (Balance + Transactions)

`GET /api/v1/tenant/tenant-statements`

Supported query params:

- Filters:
  - `lease_id` (optional, narrows both balance and transactions to one lease)
- Pagination:
  - `per_page`
  - `page`

Example requests:

- `GET /api/v1/tenant/tenant-statements`
- `GET /api/v1/tenant/tenant-statements?lease_id=101&per_page=10`

Example list response:

```json
{
  "data": {
    "balance": 14500,
    "transactions": {
      "data": [
        {
          "id": 3201,
          "lease_id": 101,
          "transaction_id": 66,
          "transaction_type": "invoice",
          "notes": "January rent",
          "debit": 15000,
          "credit": 500,
          "status": { "value": "confirmed", "color": "success" }
        }
      ],
      "links": {},
      "meta": {}
    }
  }
}
```

## Show Statement

`GET /api/v1/tenant/tenant-statements/{tenantStatement}`

Example show response:

```json
{
  "data": {
    "id": 3201,
    "lease_id": 101,
    "transaction_id": 66,
    "transaction_type": "invoice",
    "notes": "January rent",
    "debit": 15000,
    "credit": 500,
    "status": { "value": "confirmed", "color": "success" },
    "transaction_at": { "raw": "2026-02-01T00:00:00.000000Z", "formatted": "01 Feb, 2026", "diff": "2 weeks ago" },
    "created_at": { "raw": "2026-02-01T08:00:00.000000Z", "formatted": "01 Feb, 2026", "diff": "2 weeks ago" }
  }
}
```

If the statement does not belong to the authenticated tenant, the API returns `404`.
