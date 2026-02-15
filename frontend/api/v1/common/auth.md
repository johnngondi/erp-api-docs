# Common Auth API

Base route:

`/api/v1/auth`

## Public Endpoints

- `POST /auth/login`
- `POST /auth/register`
- `GET /auth/portals`

## Protected Endpoints (`auth:sanctum`)

- `POST /auth/logout`
- `POST /auth/logout-all`

## Notes

- `GET /auth/portals` returns available user group portals.
- After login, use bearer token for protected endpoints.

