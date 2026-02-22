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

Availability failures (over-allocation) are returned as `422` with field-specific keys (for example `leasable_space`, `two_bedroom_units`, `open_parkings`).

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
- unit buckets: `studio_units`, `one_bedroom_units`, `two_bedroom_units`, `three_bedroom_units`, `four_plus_bedroom_units`, `mansions`
- parking buckets: `open_parkings`, `closed_parkings`
- allocated mirrors: `allocated_*`

## 3. Division Rules (Critical for frontend)

Decision source: root facility type `division_type` (`size`, `unit`, `both`).

Rule summary:

| Scenario | What is enforced |
|---|---|
| `division_type = size` | Space size buckets (`*_space`) + parking |
| `division_type = unit` | Unit buckets (`*_units`, `mansions`) + parking |
| `division_type = both` | Space and unit + parking |

Special space rules:
- `is_signage = true`
  - unlimited
  - skips size/unit/parking availability checks
- `is_parking = true`
  - always enforced against parking capacity (open or closed), regardless of division type
  - consumes exactly 1 parking slot
- `is_mansion = true`
  - mansion mapping takes precedence over `rooms`
- room mapping for non-parking/non-signage spaces:
  - `rooms <= 0` => `studio`
  - `rooms = 1` => `one_bedroom`
  - `rooms = 2` => `two_bedroom`
  - `rooms = 3` => `three_bedroom`
  - `rooms >= 4` => `four_plus_bedroom`

Important legacy compatibility rule:
- when effective division type is `unit`, do not rely on `*_space` to enforce availability for unit spaces. Unit consumption is 1 per space item (size is still sent but not capacity-driving for unit enforcement).

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
  "facility_type_id": 3,
  "space_unit_id": 1,
  "name": "Parkview Mixed Use",
  "description": "Mixed commercial and residential complex",
  "construction_date": "2024-01-15",
  "utility_bill_distribution_method": "utility provider rate",
  "has_blocks": true,
  "has_wings": false,
  "leasable_space": 18000,
  "common_space": 4200,
  "landlord_space": 1200,
  "studio_units": 30,
  "one_bedroom_units": 45,
  "two_bedroom_units": 32,
  "three_bedroom_units": 12,
  "four_plus_bedroom_units": 6,
  "mansions": 2,
  "open_parkings": 60,
  "closed_parkings": 40,
  "country_id": 110,
  "city_id": 901,
  "physical_address": "Westlands, Nairobi",
  "coordinates": "-1.2641,36.8106"
}
```

Allowed values for `utility_bill_distribution_method`:
- `utility provider rate`
- `distribute to tenants`
- `defined`

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
  "studio_units": 30,
  "one_bedroom_units": 45,
  "two_bedroom_units": 32,
  "three_bedroom_units": 12,
  "four_plus_bedroom_units": 6,
  "mansions": 2,
  "allocated_studio_units": 4,
  "allocated_one_bedroom_units": 7,
  "allocated_two_bedroom_units": 3,
  "allocated_three_bedroom_units": 1,
  "allocated_four_plus_bedroom_units": 0,
  "allocated_mansions": 0,
  "open_parkings": 60,
  "closed_parkings": 40,
  "allocated_open_parkings": 8,
  "allocated_closed_parkings": 3,
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
  "studio_units": 10,
  "one_bedroom_units": 20,
  "two_bedroom_units": 18,
  "three_bedroom_units": 8,
  "four_plus_bedroom_units": 3,
  "mansions": 1,
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
  "studio_units": 10,
  "one_bedroom_units": 20,
  "two_bedroom_units": 18,
  "three_bedroom_units": 8,
  "four_plus_bedroom_units": 3,
  "mansions": 1,
  "allocated_studio_units": 2,
  "allocated_one_bedroom_units": 4,
  "allocated_two_bedroom_units": 1,
  "allocated_three_bedroom_units": 0,
  "allocated_four_plus_bedroom_units": 0,
  "allocated_mansions": 0,
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
  "studio_units": 6,
  "one_bedroom_units": 14,
  "two_bedroom_units": 10,
  "three_bedroom_units": 4,
  "four_plus_bedroom_units": 2,
  "mansions": 0,
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
  "studio_units": 6,
  "one_bedroom_units": 14,
  "two_bedroom_units": 10,
  "three_bedroom_units": 4,
  "four_plus_bedroom_units": 2,
  "mansions": 0,
  "allocated_studio_units": 1,
  "allocated_one_bedroom_units": 2,
  "allocated_two_bedroom_units": 0,
  "allocated_three_bedroom_units": 0,
  "allocated_four_plus_bedroom_units": 0,
  "allocated_mansions": 0,
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
  "studio_units": 4,
  "one_bedroom_units": 7,
  "two_bedroom_units": 5,
  "three_bedroom_units": 2,
  "four_plus_bedroom_units": 1,
  "mansions": 0,
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
  "studio_units": 4,
  "one_bedroom_units": 7,
  "two_bedroom_units": 5,
  "three_bedroom_units": 2,
  "four_plus_bedroom_units": 1,
  "mansions": 0,
  "allocated_studio_units": 1,
  "allocated_one_bedroom_units": 1,
  "allocated_two_bedroom_units": 0,
  "allocated_three_bedroom_units": 0,
  "allocated_four_plus_bedroom_units": 0,
  "allocated_mansions": 0,
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
  "rooms": 2,
  "is_mansion": false,
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

Mansion payload:

```json
{
  "name": "Mansion-01",
  "facility_floor_id": 78,
  "space_type": "leasable",
  "size": 1,
  "rooms": 10,
  "is_mansion": true,
  "is_parking": false,
  "is_signage": false,
  "type_id": 2
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
- if `is_mansion = true`, rooms input can exist but mansion logic is used

For structures (facility/block/wing/floor):
- do not send negative values for space/unit/parking buckets
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
