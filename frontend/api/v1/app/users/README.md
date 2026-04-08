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

## Implementation Notes

- User-group/role/permission APIs were moved under:
  - `docs/frontend/api/v1/app/access-management/README.md`
- Landlords/tenants/vendors remain in this users domain.

Apply shared auth, tenancy, query, and error handling guidance from:

- `docs/frontend/app/README.md`
