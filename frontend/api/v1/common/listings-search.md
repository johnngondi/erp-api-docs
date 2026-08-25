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

### `GET /listings`

Unauthenticated. Returns only what a prospective tenant needs to browse and map a
property — no landlord contact details, tax identifiers or permission flags.

Filters: `id`, `name`, `facility_type_id`, `status`, `description`, `country_id`,
`city_id`, `construction_date`. Sorts: `name`, `construction_date`, `created_at`.

```json
{
  "id": 4,
  "name": "Riverside Mixed-Use Plaza",
  "physical_address": "Riverside Drive, Westlands",
  "coordinates": "-1.2700,36.8000",
  "available_space": 28600,
  "available_parkings": 80,
  "residential_unit_types": [
    {
      "id": 1,
      "title": "Studio",
      "bedrooms": 0,
      "available": 10,
      "indicative_rent_per_unit": 45000,
      "indicative_service_charge_per_unit": 7500
    }
  ],
  "indicative_rent_rate_per_space_unit": 150,
  "indicative_service_charge_rate_per_space_unit": 30,
  "deposit_months": 3,
  "landlord": { "name": "Riverside Holdings Ltd" }
}
```

Availability is measured against leases, not against the facility's `allocated_*`
counters — carving space into blocks, floors or spaces does not make it
unavailable. A space counts as let while a lease holds its rent, so a space on
notice is still let until the lease ends.

- `available_space` — `leasable_space` less the size of every let leasable space.
- `available_parkings` — declared open + closed parking lots less every let one.
- `residential_unit_types[].available` — the type's `quantity` less the units let
  against it, including those let through a block, wing or floor unit type.

Per-unit indicative rates come from the facility-level unit type and are null when
unset. The facility-level `indicative_*_rate_per_space_unit` fields are per space
unit, so they price commercial space rather than residential units.
