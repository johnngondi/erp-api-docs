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
- `POST|PUT /auth/pin`, `POST /auth/pin/verify`
- `POST /auth/pin/temporary`, `POST /auth/pin/temporary/verify`
- `POST /auth/touch`

The PIN endpoints and the idle lock are documented separately in
[Session PIN & Idle Lock](session-pin.md).

## Notes

- `GET /auth/portals` returns available user group portals.
- After login, use bearer token for protected endpoints.

