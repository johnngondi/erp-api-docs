# Facilities and Structures API (Frontend Developer Guide)

Domain: `Property Management`

Base path:
`/api/v1/app/{company}/property-management`

Example with company `12`:
`/api/v1/app/12/property-management`

This document is the frontend contract for:
- facility setup
- structures (blocks, wings, floors, spaces)
- capacity/availability enforcement (size, units, parking)
- facility admins and role assignment

## 1. Authentication and Error Shape

All endpoints require authenticated app user context.

Common responses:
- `200 OK` for reads/updates
- `201 Created` for successful creates
- `204 No Content` may be used for some deletes depending on client handling
- `401/403` for auth/authorization
- `404` when route model is not found
- `422` for validation and availability conflicts

Validation error shape:

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "field_name": [
      "Validation message"
    ]
  }
}
```


## 2. Data Model and Hierarchy

Supported structure trees:
- `Facility -> Floor -> Space`
- `Facility -> Block -> Floor -> Space`
- `Facility -> Wing -> Floor -> Space`
- `Facility -> Block -> Wing -> Floor -> Space`

### 2.1 Topology Rules (Backend-Enforced)

These rules are now enforced during creation:

1. Facility flags:
- `has_blocks` and `has_wings` cannot both be `true`.
- Valid states are:
  - `has_blocks=true, has_wings=false`
  - `has_blocks=false, has_wings=true`
  - `has_blocks=false, has_wings=false`

2. Facility creation paths:
- If `has_blocks=true`: only blocks can be created directly on facility.
- If `has_wings=true`: only wings can be created directly on facility.
- If both are `false`: only floors can be created directly on facility.

3. Block creation paths:
- If `block.has_wings=true`: only wings can be created on that block (no floors).
- If `block.has_wings=false`: only floors can be created on that block (no wings).

4. Wing creation paths:
- On wing, only floors can be created.

5. Space creation:
- All spaces must be created on floors (`facility_floor_id` is required).

### 2.2 Allowed vs Forbidden Create Routes

| Parent | Parent Flags | Allowed | Forbidden |
|---|---|---|---|
| Facility | `has_blocks=true` | `POST /facilities/{facility}/blocks` | `.../wings`, `.../floors` |
| Facility | `has_wings=true` | `POST /facilities/{facility}/wings` | `.../blocks`, `.../floors` |
| Facility | `has_blocks=false`, `has_wings=false` | `POST /facilities/{facility}/floors` | `.../blocks`, `.../wings` |
| Block | `has_wings=true` | `POST /facilities/blocks/{block}/wings` | `.../floors` |
| Block | `has_wings=false` | `POST /facilities/blocks/{block}/floors` | `.../wings` |
| Wing | n/a | `POST /facilities/wings/{wing}/floors` | blocks/wings/spaces direct |
| Floor | n/a | `POST /facility-spaces/spaces` | n/a |

Capacity is maintained at each structure level (facility, block, wing, floor):
- space buckets: `leasable_space`, `common_space`, `landlord_space`
- parking buckets: `open_parkings`, `closed_parkings`
- allocated mirrors: `allocated_*`
- residential allocations: `residential_units[*].quantity` vs `residential_units[*].allocated`

## 3. Division Rules (Critical for frontend)

Decision source: root facility type `division_type` (`size`, `unit`, `both`).

Rule summary:

| Scenario | What is enforced |
|---|---|
| `division_type = size` | Space size buckets (`*_space`) + parking |
| `division_type = unit` | Unit allocation by residential unit id + parking |
| `division_type = both` | Space + parking (unit id not required) |

Special space rules:
- `is_signage = true`
  - unlimited
  - skips size/unit/parking availability checks
- `is_parking = true`
  - always enforced against parking capacity (open or closed), regardless of division type
  - consumes exactly 1 parking slot
Unit allocation rule:
- when effective division type is `unit`, space creation must reference `facility_residential_unit_id`
- backend validates that the selected unit belongs to one of the current floor ancestors (`floor`, `wing`, `block`, or root `facility`)

Residential unit hierarchy rule:
- for block/wing/floor residential unit rows, `parent_facility_residential_unit_id` is required
- parent must belong to the immediate parent structure level
- child `quantity` cannot exceed parent remaining balance after sibling child allocations
- child `title` and `bedrooms` are derived from the parent row

### 3.1 `type_id` rule for spaces when parent is `both`

For `FacilitySpace` create/update:
- if parent/root facility division is `both`
- and space is not parking/signage
- then `type_id` is required

Reason:
- system uses `type_id -> facility_types.division_type` to decide effective enforcement for that specific space in mixed (`both`) hierarchies.

## 4. Endpoint Inventory

## 4.1 Facility Types

- `GET /facility-types`

## 4.2 Facilities

- `GET /facilities`
- `POST /facilities`
- `GET /facilities/{facility}`
- `PUT /facilities/{facility}`
- `PATCH /facilities/{facility}`
- `DELETE /facilities/{facility}`
- `PATCH /facilities/{facility}/activate`
- `PATCH /facilities/{facility}/deactivate`
- `GET /facilities/{facility}/admins`
- `GET /facilities/{facility}/spaces`
- `GET /facilities/{facility}/roles`
- `PUT /facilities/{facility}/roles/{role}`

## 4.3 Facility Utilities

- `GET /facilities/{facility}/utilities`
- `POST /facilities/{facility}/utilities`
- `GET /facilities/{facility}/utilities/{utility}`
- `PUT /facilities/{facility}/utilities/{utility}`
- `PATCH /facilities/{facility}/utilities/{utility}`
- `DELETE /facilities/{facility}/utilities/{utility}`

Create/update payload:

```json
{
  "utility_id": 6,
  "utility_bill_distribution_method": "utility provider rate",
  "vendor_id": 1
}
```

Note:
- `utility_id` maps to `lease_components.id`.


## 4.5 Blocks

- `GET /facilities/{facility}/blocks`
- `POST /facilities/{facility}/blocks`
- `GET /facilities/blocks/{block}`
- `PUT /facilities/blocks/{block}`
- `PATCH /facilities/blocks/{block}`
- `DELETE /facilities/blocks/{block}`

Nested under block:
- `GET /facilities/blocks/{block}/wings`
- `POST /facilities/blocks/{block}/wings`
- `GET /facilities/blocks/{block}/floors`
- `POST /facilities/blocks/{block}/floors`
- `POST /facilities/blocks/{block}/floors/generate`

## 4.6 Wings

- `GET /facilities/{facility}/wings`
- `POST /facilities/{facility}/wings`
- `GET /facilities/wings/{wing}`
- `PUT /facilities/wings/{wing}`
- `PATCH /facilities/wings/{wing}`
- `DELETE /facilities/wings/{wing}`

Nested under wing:
- `GET /facilities/wings/{wing}/floors`
- `POST /facilities/wings/{wing}/floors`
- `POST /facilities/wings/{wing}/floors/generate`

## 4.7 Floors

- `GET /facilities/{facility}/floors`
- `POST /facilities/{facility}/floors`
- `GET /facilities/floors/{floor}`
- `PUT /facilities/floors/{floor}`
- `PATCH /facilities/floors/{floor}`
- `DELETE /facilities/floors/{floor}`
- `POST /facilities/{facility}/floors/generate`

## 4.8 Spaces

Base group: `/facility-spaces`

- `GET /facility-spaces/spaces`
- `POST /facility-spaces/spaces`
- `GET /facility-spaces/spaces/{space}`
- `PUT /facility-spaces/spaces/{space}`
- `PATCH /facility-spaces/spaces/{space}`
- `DELETE /facility-spaces/spaces/{space}`

## 5. Query Parameters (List endpoints)

## 5.1 Facilities list

`GET /facilities`

Supports:
- `filter[id]`
- `filter[name]`
- `filter[landlord_id]`
- `filter[landlord.name]`
- `sort=id,name,created_at` (prefix `-` for desc)
- `per_page`
- `fields[facilities]=...`

Example:
`GET /facilities?filter[name]=Parkview&sort=-created_at&per_page=20`

## 5.2 Blocks list

`GET /facilities/{facility}/blocks`

Supports:
- filters: `id`, `name`, `created_at`
- sorts: `id`, `name`, `created_at`
- `per_page`

## 5.3 Wings list

`GET /facilities/{facility}/wings` and `GET /facilities/blocks/{block}/wings`

Supports:
- filters: `id`, `name`, `created_at`
- sorts: `id`, `name`, `created_at`
- `per_page`

## 5.4 Floors list

`GET /facilities/{facility}/floors`, `GET /facilities/blocks/{block}/floors`, `GET /facilities/wings/{wing}/floors`

Supports:
- filters: `id`, `name`, `is_parking`, `is_divisible`, `common_space`, `leasable_space`, `landlord_space`, `total_space`
- sorts: `id`, `name`, `created_at`
- includes: `include=facilitySpaces`
- `per_page`

## 5.5 Spaces list

`GET /facility-spaces/spaces`

Supports:
- filters: `id`, `name`, `space_type`, `size`, `status`, `created_at`, `facility_floor_id`
- sorts: `id`, `name`, `size`, `status`, `created_at`
- `per_page`

## 5.6 Facility aggregate spaces

`GET /facilities/{facility}/spaces`

Scopes spaces across the full facility tree.

Supports:
- filters: `status`, `name`, `space_type`, `size`
- sorts: `name`, `size`, `status`, `space_type`, `created_at`
- `per_page`

## 6. Request Payloads by Resource

## 6.1 Facility Type (read only)

Create/update payload: not applicable.

Example list item:

```json
{
  "id": 1,
  "name": "Commercial",
  "has_tax": true,
  "enforce_with_size": false
}
```

Note:
- allocation logic uses `facility_types.division_type` from DB (`size|unit|both`)
- `division_type` may not be exposed directly in this resource payload

## 6.2 Facility

Create/update payload example:

```json
{
  "facility_type_id": 1,
  "space_unit_id": 1,
  "landlord_id": 10,
  "reporting_currency_id": 1,
  "name": "Kahiu House",
  "description": null,
  "construction_date": null,
  "utility_bill_distribution_method": "utility provider rate",
  "lease_component_allocation_priority_method": "proportional",
  "has_blocks": false,
  "has_wings": false,
  "leasable_space": 500,
  "common_space": 0,
  "landlord_space": 0,
  "residential_units": [
    {
      "title": "Studio",
      "bedrooms": 0,
      "quantity": 5,
      "indicative_rent_per_unit": 5000,
      "indicative_service_charge_per_unit": 2000
    }
  ],
  "open_parkings": 0,
  "closed_parkings": 0,
  "indicative_rent_rate_per_space_unit": 120,
  "indicative_service_charge_rate_per_space_unit": 25,
  "indicative_open_parking_fee_per_lot": 3000,
  "indicative_closed_parking_fee_per_lot": null,
  "deposit_months": 3,
  "bill_legal_fees": false,
  "legal_fee_type": "per space unit",
  "indicative_legal_rate": null,
  "country_id": 1,
  "city_id": 74,
  "physical_address": "Kilimani",
  "coordinates": null,
  "auto_generate_utility_meters": true,
  "utilities": [
    {
      "utility_id": 6,
      "utility_bill_distribution_method": "utility provider rate",
      "vendor_id": 1
    },
    {
      "utility_id": 7,
      "utility_bill_distribution_method": "defined",
      "vendor_id": 1
    }
  ],
  "contract": {
    "management_fee": {
      "enabled": true,
      "type": "percentage",
      "rate": 5
    },
    "letting_fee": {
      "enabled": true,
      "type": "fixed",
      "rate": "one_month_rent"
    },
    "re_letting_fee": {
      "enabled": true,
      "type": "fixed",
      "rate": "one_month_rent"
    },
    "wages": {
      "has_wages": true,
      "items": [
        {
          "role": "Caretaker",
          "wages": 50000
        },
        {
          "role": "Property Manager",
          "wages": 100000
        }
      ]
    }
  },
  "roles": [
    {
      "role_id": 6,
      "user_id": 1
    },
    {
      "role_id": 7,
      "user_id": 1
    },
    {
      "role_id": 11,
      "user_id": 2
    }
  ]
}
```

Allowed values for `utility_bill_distribution_method`:
- `utility provider rate`
- `distribute to tenants`
- `defined`

Notes:
- `utilities[*].utility_id` maps to `lease_components.id` (not `utilities.id`).
- `legal_fee_type` allowed values are `per space unit` and `fixed`.

Backend-required baseline fields in DTO:
- `facility_type_id`
- `space_unit_id`
- `name`
- `leasable_space`
- `common_space`
- `landlord_space`
- `physical_address`

Facility response example:

```json
{
  "id": 10,
  "name": "Parkview Mixed Use",
  "description": "Mixed commercial and residential complex",
  "has_blocks": true,
  "has_wings": false,
  "utility_bill_distribution_method": "utility provider rate",
  "leasable_space": 18000,
  "common_space": 4200,
  "landlord_space": 1200,
  "allocated_leasable_space": 3000,
  "allocated_common_space": 500,
  "allocated_landlord_space": 200,
  "open_parkings": 60,
  "closed_parkings": 40,
  "allocated_open_parkings": 8,
  "allocated_closed_parkings": 3,
  "indicative_open_parking_fee_per_lot": 3000,
  "indicative_closed_parking_fee_per_lot": null,
  "residential_units": [
    {
      "id": 12,
      "title": "Studio",
      "bedrooms": 0,
      "quantity": 50,
      "allocated": 18,
      "available": 32,
      "indicative_rent_per_unit": 5000,
      "indicative_service_charge_per_unit": 2000
    }
  ],
  "physical_address": "Westlands, Nairobi",
  "coordinates": "-1.2641,36.8106",
  "space_unit": "SqFt",
  "status": {
    "value": "active",
    "color": "success"
  },
  "facility_type": {
    "id": 3,
    "details": {
      "id": 3,
      "name": "Mixed Use",
      "has_tax": true,
      "enforce_with_size": false
    }
  }
}
```

## 6.3 Block

Create/update payload:

```json
{
  "name": "Block A",
  "has_wings": true,
  "leasable_space": 6000,
  "common_space": 1200,
  "landlord_space": 300,
  "residential_units": [
    {
      "parent_facility_residential_unit_id": 12,
      "quantity": 10,
      "indicative_rent_per_unit": 5000,
      "indicative_service_charge_per_unit": 2000
    }
  ],
  "open_parkings": 20,
  "closed_parkings": 12
}
```

Response item example:

```json
{
  "id": 31,
  "name": "Block A",
  "has_wings": true,
  "leasable_space": 6000,
  "common_space": 1200,
  "landlord_space": 300,
  "allocated_leasable_space": 800,
  "allocated_common_space": 150,
  "allocated_landlord_space": 50,
  "residential_units": [],
  "open_parkings": 20,
  "closed_parkings": 12,
  "allocated_open_parkings": 3,
  "allocated_closed_parkings": 1
}
```

## 6.4 Wing

Create/update payload (same shape for wings under facility or under block):

```json
{
  "name": "North Wing",
  "leasable_space": 3000,
  "common_space": 700,
  "landlord_space": 150,
  "residential_units": [
    {
      "parent_facility_residential_unit_id": 21,
      "quantity": 6,
      "indicative_rent_per_unit": 5500,
      "indicative_service_charge_per_unit": 2200
    }
  ],
  "open_parkings": 8,
  "closed_parkings": 6
}
```

Response item example:

```json
{
  "id": 55,
  "name": "North Wing",
  "leasable_space": 3000,
  "common_space": 700,
  "landlord_space": 150,
  "allocated_leasable_space": 500,
  "allocated_common_space": 90,
  "allocated_landlord_space": 40,
  "residential_units": [],
  "open_parkings": 8,
  "closed_parkings": 6,
  "allocated_open_parkings": 1,
  "allocated_closed_parkings": 1
}
```

## 6.5 Floor

Create/update payload (facility/block/wing floor creation all use same structure):

```json
{
  "name": "Floor 1",
  "is_parking": false,
  "is_divisible": true,
  "leasable_space": 1400,
  "common_space": 260,
  "landlord_space": 80,
  "residential_units": [
    {
      "parent_facility_residential_unit_id": 33,
      "quantity": 4,
      "indicative_rent_per_unit": 6000,
      "indicative_service_charge_per_unit": 2500
    }
  ],
  "open_parkings": 4,
  "closed_parkings": 3
}
```

Response item example:

```json
{
  "id": 78,
  "name": "Floor 1",
  "is_parking": false,
  "is_divisible": true,
  "leasable_space": 1400,
  "common_space": 260,
  "landlord_space": 80,
  "allocated_leasable_space": 300,
  "allocated_common_space": 40,
  "allocated_landlord_space": 10,
  "residential_units": [],
  "open_parkings": 4,
  "closed_parkings": 3,
  "allocated_open_parkings": 1,
  "allocated_closed_parkings": 0,
  "spaces": []
}
```

## 6.6 Floor generation (AI prompt)

Endpoints:
- `POST /facilities/{facility}/floors/generate`
- `POST /facilities/blocks/{block}/floors/generate`
- `POST /facilities/wings/{wing}/floors/generate`

Payload:

```json
{
  "prompt": "Create 3 floors with mixed room units and realistic parking split."
}
```

Response:
- collection of created floors (`FacilityFloorResource`) including generated `spaces`.

## 6.7 Space

Create payload:

```json
{
  "name": "A-101",
  "facility_floor_id": 78,
  "space_type": "leasable",
  "size": 1,
  "facility_residential_unit_id": 31,
  "is_parking": false,
  "is_signage": false,
  "type_id": 2
}
```

Update payload:
- same as create except `facility_floor_id` is taken from existing record on backend.

Parking payload:

```json
{
  "name": "P-017",
  "facility_floor_id": 78,
  "space_type": "common",
  "size": 1,
  "is_parking": true,
  "parking_type": "closed",
  "is_signage": false
}
```

Signage payload:

```json
{
  "name": "Signage-East-1",
  "facility_floor_id": 78,
  "space_type": "common",
  "size": 1,
  "is_parking": false,
  "is_signage": true
}
```

Space response example:

```json
{
  "id": 120,
  "name": "A-101",
  "space_type": "leasable",
  "size": 1,
  "is_parking": false,
  "is_signage": false,
  "status": {
    "value": "available",
    "color": "success"
  },
  "allocated_components_ids": [1, 2],
  "floor": {
    "id": 78,
    "name": "Floor 1"
  },
  "facility": {
    "id": 10,
    "name": "Parkview Mixed Use"
  },
  "wing": {
    "id": 55,
    "name": "North Wing"
  },
  "block": {
    "id": 31,
    "name": "Block A"
  }
}
```

## 6.8 Facility role assignment

List roles:
- `GET /facilities/{facility}/roles`

Assign user to facility role:
- `PUT /facilities/{facility}/roles/{role}`

Payload:

```json
{
  "user_id": 34
}
```

Response example:

```json
{
  "id": 9,
  "role": {
    "id": 4,
    "name": "Manager"
  },
  "user": {
    "id": 34,
    "name": "Jane Doe"
  },
  "availableUsers": []
}
```

## 6.9 Facility admins

- `GET /facilities/{facility}/admins`

Payload: none.

Response:
- collection of active users attached to facility.


## 7. Frontend Guardrails (Do this before submit)

Topology guardrails:
- Never submit facility create/update with both `has_blocks=true` and `has_wings=true`.
- Drive structure create buttons from parent topology:
  - facility blocks-only -> show block create only
  - facility wings-only -> show wing create only
  - facility plain -> show floor create only
  - block with wings -> show wing create only
  - block without wings -> show floor create only
  - wing -> show floor create only
- For invalid topology attempts, expect `422` with keys like `has_blocks`, `has_wings`, or `floorable`.

For space create/update:
- always send `size >= 1`
- if `is_parking = true`, require `parking_type` (`open` or `closed`)
- if `is_signage = true`, do not block on capacity UI (backend treats signage as unlimited)
- if root facility division is `both` and space is not parking/signage, require `type_id`
- if effective division resolves to `unit`, send `facility_residential_unit_id`

For structures (facility/block/wing/floor):
- do not send negative values for space/parking/residential-unit quantities
- when API returns `422` over-allocation, display backend field errors directly near matching inputs

For parking spaces:
- assume one parking space consumes one slot
- enforce parking availability in UI for all division types (`size`, `unit`, `both`)

## 8. Common Success Envelope

Most mutating endpoints return:

```json
{
  "data": {
    "message": "Human readable success message",
    "<resource_key>": {}
  }
}
```

Collection endpoints return paginated resources under `data` and include pagination `links` and `meta`.

## 9. Postman Quick Payloads

These are copy-paste payloads for the new residential units design.

### 9.1 Block

Create block route:
`POST /api/v1/app/{company}/property-management/facilities/{facility}/blocks`

```json
{
  "name": "Block A",
  "has_wings": true,
  "leasable_space": 12000,
  "common_space": 2400,
  "landlord_space": 500,
  "residential_units": [
    {
      "parent_facility_residential_unit_id": 1,
      "quantity": 30,
      "indicative_rent_per_unit": 18000,
      "indicative_service_charge_per_unit": 2500
    },
    {
      "parent_facility_residential_unit_id": 2,
      "quantity": 24,
      "indicative_rent_per_unit": 26000,
      "indicative_service_charge_per_unit": 3000
    }
  ],
  "open_parkings": 30,
  "closed_parkings": 12
}
```

Update block route:
`PUT /api/v1/app/{company}/property-management/facilities/blocks/{block}`

```json
{
  "name": "Block A - Rev 2",
  "has_wings": true,
  "leasable_space": 12500,
  "common_space": 2500,
  "landlord_space": 600,
  "residential_units": [
    {
      "id": 401,
      "parent_facility_residential_unit_id": 1,
      "quantity": 32,
      "indicative_rent_per_unit": 18500,
      "indicative_service_charge_per_unit": 2600
    },
    {
      "parent_facility_residential_unit_id": 5,
      "quantity": 12,
      "indicative_rent_per_unit": 42000,
      "indicative_service_charge_per_unit": 4000
    }
  ],
  "open_parkings": 32,
  "closed_parkings": 14
}
```

### 9.2 Wing

Create wing route:
- under facility: `POST /api/v1/app/{company}/property-management/facilities/{facility}/wings`
- under block: `POST /api/v1/app/{company}/property-management/facilities/blocks/{block}/wings`

```json
{
  "name": "North Wing",
  "leasable_space": 5000,
  "common_space": 800,
  "landlord_space": 200,
  "residential_units": [
    {
      "parent_facility_residential_unit_id": 21,
      "quantity": 18,
      "indicative_rent_per_unit": 28000,
      "indicative_service_charge_per_unit": 3200
    },
    {
      "parent_facility_residential_unit_id": 23,
      "quantity": 8,
      "indicative_rent_per_unit": 65000,
      "indicative_service_charge_per_unit": 6000
    }
  ],
  "open_parkings": 10,
  "closed_parkings": 8
}
```

Update wing route:
`PUT /api/v1/app/{company}/property-management/facilities/wings/{wing}`

```json
{
  "name": "North Wing - Rev 1",
  "leasable_space": 5200,
  "common_space": 900,
  "landlord_space": 220,
  "residential_units": [
    {
      "id": 510,
      "parent_facility_residential_unit_id": 21,
      "quantity": 20,
      "indicative_rent_per_unit": 28500,
      "indicative_service_charge_per_unit": 3300
    }
  ],
  "open_parkings": 11,
  "closed_parkings": 8
}
```

### 9.3 Floor

Create floor route:
- under facility: `POST /api/v1/app/{company}/property-management/facilities/{facility}/floors`
- under block: `POST /api/v1/app/{company}/property-management/facilities/blocks/{block}/floors`
- under wing: `POST /api/v1/app/{company}/property-management/facilities/wings/{wing}/floors`

```json
{
  "name": "Floor 1",
  "is_parking": false,
  "is_divisible": true,
  "leasable_space": 2000,
  "common_space": 300,
  "landlord_space": 100,
  "residential_units": [
    {
      "parent_facility_residential_unit_id": 33,
      "quantity": 10,
      "indicative_rent_per_unit": 17000,
      "indicative_service_charge_per_unit": 2200
    }
  ],
  "open_parkings": 6,
  "closed_parkings": 4
}
```

Update floor route:
`PUT /api/v1/app/{company}/property-management/facilities/floors/{floor}`

```json
{
  "name": "Floor 1 - Rev 1",
  "is_parking": false,
  "is_divisible": true,
  "leasable_space": 2100,
  "common_space": 320,
  "landlord_space": 100,
  "residential_units": [
    {
      "id": 680,
      "parent_facility_residential_unit_id": 33,
      "quantity": 11,
      "indicative_rent_per_unit": 17250,
      "indicative_service_charge_per_unit": 2300
    }
  ],
  "open_parkings": 6,
  "closed_parkings": 5
}
```

### 9.4 Space

Create space route:
`POST /api/v1/app/{company}/property-management/facility-spaces/spaces`

Leasable space (unit-based facility or unit-based type resolution):

```json
{
  "name": "A-101",
  "facility_floor_id": 78,
  "space_type": "leasable",
  "size": 1,
  "facility_residential_unit_id": 680,
  "is_parking": false,
  "is_signage": false,
  "type_id": 2
}
```

Parking space:

```json
{
  "name": "P-017",
  "facility_floor_id": 78,
  "space_type": "common",
  "size": 1,
  "is_parking": true,
  "parking_type": "closed",
  "is_signage": false
}
```

Signage space:

```json
{
  "name": "Signage-East-1",
  "facility_floor_id": 78,
  "space_type": "common",
  "size": 1,
  "is_parking": false,
  "is_signage": true
}
```

Update space route:
`PUT /api/v1/app/{company}/property-management/facility-spaces/spaces/{space}`

```json
{
  "name": "A-101A",
  "space_type": "leasable",
  "size": 1,
  "facility_residential_unit_id": 680,
  "is_parking": false,
  "is_signage": false,
  "type_id": 2
}
```

### 9.5 Residential units sync behavior on update

- If `residential_units` is omitted from update payload, existing residential unit types are kept unchanged.
- If `residential_units` is sent as `[]`, backend attempts to remove all unit types; any type with `allocated > 0` is blocked.
- If `residential_units` is sent with rows containing `id`, those rows are updated in place.
- Rows without `id` are created as new unit types.
