# Profile

Base route:

`/api/v1`

The signed-in user's own record: reading it, editing their details, and their signature. Every
route here is about the caller and takes no user id. All require `auth:sanctum` and sit behind the
session lock (`423` while locked, see [session-pin.md](./session-pin.md)).

## Endpoints

| Method | Path | What it does |
|---|---|---|
| `GET` | `/profile` | The caller, their companies, portals and the lock settings |
| `PATCH` | `/profile` | Update name, email, phone, tax PIN (any subset) |
| `PUT` | `/profile/signature` | Save or replace the caller's signature |
| `DELETE` | `/profile/signature` | Remove the caller's signature |

## The user object

Returned under `data.user` by all four endpoints:

```json
{
  "id": 12,
  "name": "Jane Wanjiru",
  "email": "jane@example.test",
  "phone": "0712345678",
  "tax_pin": "A012345678Z",
  "profile_photo_url": "https://...",
  "has_pin": true,
  "has_signature": true,
  "signature": "data:image/png;base64,iVBORw0KGgo..."
}
```

| Field | Notes |
|---|---|
| `phone`, `tax_pin` | **Left out when null**, like every other nullable field on this resource |
| `has_signature` | Whether a signature is saved. Present on every user resource (login too) |
| `signature` | The PNG data URL, or `null`. **Only on these four profile responses.** No other endpoint ever returns another user's signature, or yours, so do not look for it on login or on user lists |

## `GET /profile`

```json
{
  "data": {
    "user": { "...": "the user object above" },
    "companies": [],
    "registered_portals": [],
    "session": { "idle_minutes": 15, "max_attempts": 3, "lockout_minutes": 15, "enforced": true }
  }
}
```

## `PATCH /profile`

Send only what changes. A key left out is left alone; `phone` and `tax_pin` sent as `null` are
cleared.

```json
{
  "name": "Jane W. Kamau",
  "email": "jane.kamau@example.test",
  "phone": "0712345678",
  "tax_pin": "A012345678Z"
}
```

| Field | Rules |
|---|---|
| `name` | sometimes, required, string, max 191 |
| `email` | sometimes, required, email, max 191, unique among users (your own address is allowed) |
| `phone` | sometimes, nullable, string, max 30 |
| `tax_pin` | sometimes, nullable, string, max 30 |

Response `200`:

```json
{ "data": { "message": "Profile updated successfully.", "user": { "...": "the user object" } } }
```

`422` with `errors.email` when another user already holds the address.

## `PUT /profile/signature`

```json
{ "signature": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAA..." }
```

Rules:

- required string;
- must start with `data:image/png;base64,` (PNG only: draw on a canvas and send `canvas.toDataURL('image/png')`);
- the base64 must decode to a real PNG (its first bytes are checked);
- the decoded image may be **at most 1 MB**. Crop the canvas to the ink and keep it small; it is
  printed about 60px tall.

Response `200`:

```json
{ "data": { "message": "Signature saved successfully.", "user": { "has_signature": true, "signature": "data:image/png;base64,...", "...": "" } } }
```

`422` with `errors.signature` for a missing value, a non-PNG, invalid base64, or an image over 1 MB.

The signature is **encrypted at rest**. It is printed on every Letter of Offer the user approves
(`{{approval_signatures}}`): the last two approvals of an offer's chain, each with the approver's
signature, name and role. An approver with no signature gets a blank signing line instead. An offer
picks up a new or changed signature the next time its approval chain moves (a step approved, sent
back or resubmitted), not the moment it is saved.

## `DELETE /profile/signature`

No body. Response `200`:

```json
{ "data": { "message": "Signature removed successfully.", "user": { "has_signature": false, "signature": null, "...": "" } } }
```
