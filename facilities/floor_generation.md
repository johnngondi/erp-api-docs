# Facility Floor Generation API

These endpoints generate facility floors and spaces from a prompt, using the OpenAI-powered layout service.

## Endpoints

### Generate floors for a facility
`POST /api/v1/crm/facilities/{facility}/floors/generate`

Request body:
- `prompt` (string, required)

Response:
```json
{
  "data": [
    {
      "id": 123,
      "name": "Ground Floor",
      "is_parking": false,
      "is_divisible": true,
      "leasable_space": 1200,
      "common_space": 300,
      "landlord_space": 0,
      "allocated_leasable_space": 600,
      "allocated_common_space": 300,
      "allocated_landlord_space": 0,
      "spaces": [
        {
          "id": 1,
          "name": "Shop 1",
          "space_type": "leasable",
          "size": 600,
          "status": "available",
          "is_parking": false,
          "is_signage": false
        },
        {
          "id": 2,
          "name": "Lobby",
          "space_type": "common",
          "size": 300,
          "status": "available",
          "is_parking": false,
          "is_signage": false
        }
      ],
      "created_at": {
        "raw": "2026-01-28T16:21:00.000000Z",
        "formatted": "28 Jan, 2026",
        "diff": "1 second ago"
      }
    }
  ]
}
```

Notes:
- Returns all floors for the facility, with `facilitySpaces` loaded.
- Uses `FacilityFloorResource` for formatting.

### Generate floors for a block
`POST /api/v1/crm/facilities/blocks/{block}/floors/generate`

Request body:
- `prompt` (string, required)

Response:
- Same shape as facility generation above.

### Generate floors for a wing
`POST /api/v1/crm/facilities/wings/{wing}/floors/generate`

Request body:
- `prompt` (string, required)

Response:
- Same shape as facility generation above.

## Errors

- `422 Unprocessable Entity` when:
  - `prompt` is missing or not a string.
  - Space allocation validation fails (e.g., insufficient `leasable_space` or `common_space`).
- `403 Forbidden` if the authenticated user is not authorized.

