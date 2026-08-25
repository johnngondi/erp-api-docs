# Tenant Statements API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/tenant-statements`

## Endpoints

- `GET /tenant-statements` (balance + transactions)
- `GET /tenant-statements/{tenantStatement}` (show)

## List Statements (Balance + Transactions)

`GET /api/v1/app/{company}/property-management/lease-management/tenant-statements`

Supported query params:

- Filters:
  - `filter[tenant_id]` (required)
  - `filter[lease_id]` (optional)
- Pagination:
  - `per_page`
  - `page`

Example requests:

- `GET /api/v1/app/12/property-management/lease-management/tenant-statements?filter[tenant_id]=77`
- `GET /api/v1/app/12/property-management/lease-management/tenant-statements?filter[tenant_id]=77&filter[lease_id]=101`

Example list response:

```json
{
  "data": {
    "balance": 14500,
    "transactions": {
      "data": [
        {
          "id": 3201,
          "tenant": {
            "id": 77,
            "name": "Jane Tenant"
          }
        }
      ],
      "links": {},
      "meta": {}
    }
  }
}
```

## Show Statement Entry

`GET /api/v1/app/{company}/property-management/lease-management/tenant-statements/{tenantStatement}`

Example show response:

```json
{
  "data": {
    "id": 3201,
    "tenant": {
      "id": 77,
      "name": "Jane Tenant"
    }
  }
}
```
