# Lease Deposits API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits`

## Endpoints

- `GET /leases/{lease}/deposits`
- `POST /leases/{lease}/deposits`
- `GET /leases/{lease}/deposits/{leaseDeposit}`
- `PUT/PATCH /leases/{lease}/deposits/{leaseDeposit}`
- `DELETE /leases/{lease}/deposits/{leaseDeposit}`

## List Lease Deposits

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_item_component_id]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_item_component_id,created_at`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/leases/101/deposits?filter[lease_item_component_id]=455&sort=-created_at`

## Create Lease Deposit

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_item_component_id` | Yes | integer | Must exist in `lease_item_components.id` |
| `amount` | Yes | number | - |
| `billed` | Yes | boolean | Usually `false` on create unless intentionally billed |

Example request:

```json
{
  "lease_item_component_id": 455,
  "amount": 500,
  "billed": false
}
```

Example response:

```json
{
  "message": "Lease deposit created successfully",
  "lease_deposit": {
    "id": 45,
    "amount": 500,
    "billed": false
  }
}
```

## Update Lease Deposit

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits/{leaseDeposit}`

Use the same payload shape as create.

## Delete Lease Deposit

`DELETE /api/v1/app/{company}/property-management/lease-management/leases/{lease}/deposits/{leaseDeposit}`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
