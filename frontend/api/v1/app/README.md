# App API Frontend Reference

This reference is for frontend teams integrating app endpoints.

Base URL pattern:

`/api/v1/app/{company}`

Example with company `12`:

`/api/v1/app/12/property-management/lease-management/leases`

## Authentication

Use these headers on authenticated requests:

- `Authorization: Bearer <token>`
- `Accept: application/json`
- `Content-Type: application/json` (for JSON payloads)

## Multi-Tenancy Safety

All app routes are company scoped (`{company}`).

- Always send the currently selected company in the URL.
- Do not reuse cached URLs from a different company session.
- If company changes, clear cached lists/details and refetch.

## List Requests: Filter, Sort, Include, Select Fields

Most list endpoints use Spatie Query Builder:

- Filter: `filter[field]=value`
- Sort: `sort=field,-another_field`
- Include relations: `include=relation1,relation2` (only if endpoint supports includes)
- Select specific fields: `fields[resource]=id,name,status` (only if endpoint supports fields)
- Pagination: `per_page=20&page=2`

Examples:

- Filter + sort:
  - `GET /api/v1/app/12/property-management/lease-management/leases?filter[status]=active&sort=-start_at`
- Include:
  - `GET /api/v1/app/12/property-management/finance/bills?include=items`
- Select fields:
  - `GET /api/v1/app/12/property-management/facilities?fields[facilities]=id,name,status`

If an endpoint does not support `include` or `fields`, sending those params returns a `4xx` error.

## Error Handling (Frontend Behavior)

### 4xx errors (show to user)

Treat `4xx` as user-actionable:

- `400`: malformed request/query
- `401`: unauthenticated
- `403`: forbidden
- `404`: resource not found in current company scope
- `409`: conflict/state issue
- `422`: validation/business rule error
- `429`: rate limited

Recommended UI behavior:

- Show backend `message`.
- If available, show field-level `errors`.
- Keep form state where possible.

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

### 5xx errors (generic message)

Treat `5xx` as backend/system failure:

- Show a generic message like: `Something went wrong. Please try again.`
- Offer retry.
- Log endpoint + request correlation info client-side for support.
- Do not show raw exception text to users.

## Known API Caveats (As of February 11, 2026)

- `Facility Management > Sensors > Entries` routes exist but backend controller is not implemented yet.
  - Endpoints: `/api/v1/app/{company}/facility-management/sensors/{sensor}/entries...`
- `Users` module has partial CRUD coverage on some resources.
  - `roles`: only list/create are implemented.
  - `user-groups`: only list is implemented.
  - `landlords`, `tenants`, `vendors`: update routes exist but are not implemented.
  - `tenants` delete route responds, but deletion logic is not fully wired.
- Integrate frontend for these caveats with graceful fallbacks:
  - Hide unsupported actions in UI where possible.
  - If shown, handle `4xx/5xx` responses with the error strategy above.

## Property Management Docs

- Overview:
  - `docs/frontend/app/property-management/README.md`
- Dashboard:
  - `docs/frontend/app/property-management/dashboard.md`
- Facilities:
  - `docs/frontend/app/property-management/facilities/facilities.md`
  - `docs/frontend/app/property-management/facilities/floor_generation.md`
- Tickets:
  - `docs/frontend/app/property-management/tickets.md`
- Procurement:
  - `docs/frontend/app/property-management/procurement.md`
- Finance:
  - `docs/frontend/app/property-management/finance.md`
- Settings:
  - `docs/frontend/app/property-management/settings.md`
- Lease Management:
  - `docs/frontend/app/property-management/lease-management/leases.md`
  - `docs/frontend/app/property-management/lease-management/invoices.md`
  - `docs/frontend/app/property-management/lease-management/credit-notes.md`
  - `docs/frontend/app/property-management/lease-management/receipts.md`
  - `docs/frontend/app/property-management/lease-management/lease-deposits.md`
  - `docs/frontend/app/property-management/lease-management/lease-escalations.md`
  - `docs/frontend/app/property-management/lease-management/lease-opening-balances.md`
  - `docs/frontend/app/property-management/lease-management/charges-imports.md`

## Facility Management Docs

- Overview:
  - `docs/frontend/app/facility-management/README.md`
- Assets:
  - `docs/frontend/app/facility-management/assets.md`
- Fleet:
  - `docs/frontend/app/facility-management/fleet.md`
- Sensors:
  - `docs/frontend/app/facility-management/sensors.md`
- Power Management:
  - `docs/frontend/app/facility-management/power-management.md`
- Utility Management:
  - `docs/frontend/app/facility-management/utility-management.md`
- Settings:
  - `docs/frontend/app/facility-management/settings.md`

## Project Management Docs

- Overview:
  - `docs/frontend/app/project-management/README.md`
- Projects:
  - `docs/frontend/app/project-management/projects.md`

## Users Docs

- Overview:
  - `docs/frontend/app/users/README.md`
- Landlords:
  - `docs/frontend/app/users/landlords.md`
- Tenants:
  - `docs/frontend/app/users/tenants.md`
- Vendors:
  - `docs/frontend/app/users/vendors.md`
- User Groups:
  - `docs/frontend/app/users/user-groups.md`
- Roles:
  - `docs/frontend/app/users/roles.md`
- Permissions:
  - `docs/frontend/app/users/permissions.md`
