# Common API v1

Base route:

`/api/v1`

Purpose:

- Shared endpoints used across portals (auth, profile, global settings, lookups).

Authentication:

- Public endpoints: under `/auth` (login/register/portals) and `/listings`.
- Protected endpoints: require `auth:sanctum`.

## Docs Index

- `docs/frontend/api/v1/common/auth.md`
- `docs/frontend/api/v1/common/listings-search.md`
- `docs/frontend/api/v1/common/settings.md`

