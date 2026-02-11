# App API Docs (Backend Reference)

Scope: `app` routes only.

Base prefix:
```text
/api/v1/app/{company}
```

This docs set is a technical reference generated from routes, DTOs, and resources for Property Management, Facility Management, Project Management, and App Users modules.

## Contents
- `docs/backend/api/v1/app/routes-catalog.md`: full route catalog (method + URI + controller).
- `docs/backend/api/v1/app/query-capabilities.md`: list endpoint query options (`filter`, `sort`, `include`, `fields`).
- `docs/backend/api/v1/app/request-contracts.md`: request DTO field contracts (required/optional + enum inputs + validation hints).
- `docs/backend/api/v1/app/enums.md`: enum values and color mappings from code.

## Auth and Headers
Required headers for almost all app routes:
- `Authorization: Bearer <token>`
- `Accept: application/json`
- `Content-Type: application/json` (for body requests)

## Company Scope (Multi-Tenancy)
All app routes are company-scoped through `{company}`.

Frontend rule:
- Always call endpoints with the current company id.
- Never reuse URLs cached from another company session.
- If company context changes, clear cached list/detail responses and reload.

## Request Contract Rules
Use `docs/backend/api/v1/app/request-contracts.md` as source of truth for payload fields.

Conventions:
- `Required = Yes`: field must be sent.
- `Required = No`: optional field.
- `Allowed Values (Enum)`: accepted enum string values for input.
- `Rules` column may include conditional requirements such as `RequiredIf`, `RequiredUnless`.

Example (enum input):
```json
{
  "status": "active"
}
```

## List Querying (filter/sort/include/fields)
Use `docs/backend/api/v1/app/query-capabilities.md` to check exactly which endpoint supports what.

### Filter
```http
GET /api/v1/app/12/property-management/lease-management/leases?filter[status]=active
```

### Sort
```http
GET /api/v1/app/12/facility-management/fleet/vehicles?sort=-created_at,license_plate
```

### Include
```http
GET /api/v1/app/12/property-management/finance/bills?include=items
```

### Sparse Fields
```http
GET /api/v1/app/12/property-management/facilities?fields[facilities]=id,name,status
```

### Pagination
```http
GET /api/v1/app/12/project-management/projects?per_page=20&page=2
```

Notes:
- If a list endpoint does not advertise `include`/`fields` in the matrix, do not send those params.
- Invalid filter/sort/include/fields will typically return a `4xx` validation-style response.

## Static Example Responses
These are realistic static examples based on current API resources.

### Project item (status enum with color)
```json
{
  "id": 44,
  "title": "Parking Expansion",
  "status": {
    "value": "in-progress",
    "color": "primary"
  },
  "estimated_start_date": {
    "raw": "2026-01-10T00:00:00.000000Z",
    "formatted": "10 Jan, 2026",
    "diff": "1 month ago"
  }
}
```

### Ticket item (priority/status enums with color)
```json
{
  "id": 901,
  "title": "Water leak at Block A",
  "ticket_type": "facility",
  "priority": {
    "value": "high",
    "color": "danger"
  },
  "status": {
    "value": "open",
    "color": "primary"
  }
}
```

### Lease item (status enum with color)
```json
{
  "id": 101,
  "billing_cycle": "monthly",
  "status": {
    "value": "active",
    "color": "success"
  },
  "next_due_at": {
    "raw": "2026-03-01T00:00:00.000000Z",
    "formatted": "01 Mar, 2026",
    "diff": "in 2 weeks"
  }
}
```

## Error Handling Guidelines (Frontend)

### 4xx (show to user)
Treat as user-actionable whenever possible.

- `400 Bad Request`: malformed request/query params.
- `401 Unauthorized`: not authenticated. Redirect to login / refresh token flow.
- `403 Forbidden`: authenticated but not allowed (permission/company/portal mismatch).
- `404 Not Found`: missing/invalid resource id in current scope.
- `409 Conflict`: state conflict (if returned by endpoint behavior).
- `422 Unprocessable Entity`: validation/business-rule error.
- `429 Too Many Requests`: throttled, retry later.

Recommended UI behavior:
- Render `message` and field-level `errors` (for validation).
- Keep form data when possible.
- Do not show generic crash toast for 4xx.

Typical 422 shape:
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

### 5xx (generic message)
Treat as backend/system failure.

Recommended UI behavior:
- Show generic message: `Something went wrong. Please try again.`
- Optionally provide retry action.
- Log request id / endpoint / payload fingerprint for debugging.
- Do not expose raw backend exception text to end users.

## Enum Colors
For all enum badge/chip colors, use:
- `docs/backend/api/v1/app/enums.md`

If an endpoint returns `{ "value": "...", "color": "..." }`, prefer the backend-provided `color` directly.
