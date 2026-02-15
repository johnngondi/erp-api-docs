# Tenant Statements API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/tenant-statements`

## Endpoints

- `GET /tenant-statements` (list)
- `GET /tenant-statements/{tenantStatement}` (show)

## Statement Types

There are two statement modes on the list endpoint:

1. User-level statement summary
   - Filter using `filter[tenant_id]`
   - Returns statement balance grouped at user level
2. Lease-level statement summary
   - Filter using `filter[user_id]`
   - Returns statement balance for a specific lease context

## List Statements

`GET /api/v1/app/{company}/property-management/lease-management/tenant-statements`

Supported query params:

- Filters:
  - `filter[tenant_id]`
  - `filter[user_id]`
- Pagination:
  - `per_page`
  - `page`

Example requests:

- `GET /api/v1/app/12/property-management/lease-management/tenant-statements?filter[tenant_id]=77`
- `GET /api/v1/app/12/property-management/lease-management/tenant-statements?filter[user_id]=77`

Example list response:

```json
{
  "data": [
    {
      "tenant": {
        "id": 77,
        "name": "Jane Tenant"
      },
      "balance": "14500"
    }
  ],
  "links": {},
  "meta": {}
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

