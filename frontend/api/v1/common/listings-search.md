# Common Listings and Search API

Base route:

`/api/v1`

## Public Endpoints

- `GET /listings`

## Protected Endpoints (`auth:sanctum`)

- `GET /profile`
- `GET /locations/countries`
- `GET /locations/countries/{country}/cities`
- `GET /search/purchase-items/{expenseSubType}`

## Notes

- `/profile` returns:
  - authenticated user
  - accessible companies
  - registered portals

