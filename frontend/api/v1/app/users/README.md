# Users API (Frontend)

Base prefix:

`/api/v1/app/{company}/users`

## Modules

- Landlords
  - `docs/frontend/app/users/landlords.md`
- Tenants
  - `docs/frontend/app/users/tenants.md`
- Vendors
  - `docs/frontend/app/users/vendors.md`
- User Groups
  - `docs/frontend/app/users/user-groups.md`
- Roles
  - `docs/frontend/app/users/roles.md`
- Permissions
  - `docs/frontend/app/users/permissions.md`

## Implementation Notes

- Several users routes exist but are only partially implemented in backend controllers.
- Each module file documents its current implementation status; check that section before wiring edit/delete UI flows.

Apply shared auth, tenancy, query, and error handling guidance from:

- `docs/frontend/app/README.md`
