# Access Management: Responsibility Transfers

Base prefix:

`/api/v1/app/{company}/access-management`

A responsibility transfer hands one staff member's properties and roles to a colleague, either
temporarily (for leave) or permanently (a real handover).

## Endpoints

Implemented:

- `GET /responsibility-transfers`
- `POST /responsibility-transfers`
- `GET /responsibility-transfers/mine/outgoing`
- `GET /responsibility-transfers/mine/incoming`
- `GET /responsibility-transfers/{transfer}`
- `PATCH /responsibility-transfers/{transfer}/accept`
- `PATCH /responsibility-transfers/{transfer}/decline`
- `PATCH /responsibility-transfers/{transfer}/reclaim`

`mine/outgoing` and `mine/incoming` are registered ahead of the `{transfer}` wildcard, so `mine`
is never read as a transfer id.

## Permissions

| Action | Permission | Additional rule |
| --- | --- | --- |
| Initiate | `transfer-responsibility` | The transferer is always the caller; it is never read from the payload. |
| Accept / Decline | `transfer-responsibility` | Only the named recipient. Anyone else gets `403`. |
| Reclaim | `transfer-responsibility` | Only the transferer. Anyone else gets `403`. |
| History (`index` / `show`) | `transfer-responsibility` | Visible to the transferer and the recipient only. |
| `mine/*` | none | Caller-scoped: the query only ever returns the caller's own rows. |

No approval chain is involved in any of these.

## Status lifecycle

| Status | Meaning |
| --- | --- |
| `pending` | Sent, awaiting the recipient's answer. Nothing has been granted yet. |
| `active` | Accepted. The grants are live. |
| `declined` | The recipient said no. Nothing changed hands. |
| `reclaimed` | The transferer pulled it back. Grants reversed. |
| `returned` | A temporary transfer reached `timestamp_to` and reversed itself. |
| `expired` | Nobody answered before `timestamp_to` passed. Nothing to reverse. |
| `accepted` | Reserved. Accept writes `active`; treat both as in force if you filter on status. |

Transitions:

```
pending ──accept──▶ active ──reclaim───▶ reclaimed
   │                  │
   │                  └──timestamp_to──▶ returned
   ├──decline──▶ declined
   └──timestamp_to──▶ expired
```

Every status object is the house `{ value, color }` pair.

## Rules the UI must respect

- **One open outgoing transfer per user**, counted across every company they belong to. A second
  initiate returns `422` on `to_user_id`. This is why `mine/outgoing` returns a single object.
- **Many incoming at once** is normal. `mine/incoming` returns an array.
- The recipient must be an **active member of the same company and the same department** as the
  transferer. Department is matched exactly, null included. Anything else is `422` on `to_user_id`.
- A **temporary** transfer requires `timestamp_to`, and it must be in the future.
- A **permanent** transfer must not send `timestamp_to`, has no countdown, and rejects reclaim
  with `422`.

## List transfers

`GET /api/v1/app/{company}/access-management/responsibility-transfers`

Returns only transfers the caller sent or received.

Supported query params:

- Filters:
  - `filter[status]`
  - `filter[is_permanent]`
  - `filter[direction]` (`outgoing` or `incoming`)
  - `filter[from_user_id]`
  - `filter[to_user_id]`
  - `filter[created_at]` (date: `YYYY-MM-DD`)
- Sort:
  - `sort=status`, `sort=timestamp_to`, `sort=accepted_at`, `sort=created_at`
- Pagination:
  - `per_page`, `page`

## Initiate a transfer

`POST /api/v1/app/{company}/access-management/responsibility-transfers`

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `to_user_id` | int | yes | — | Must be an active member of this company, in the caller's department, and not the caller. |
| `new_facilities` | int[] | yes | — | Facility ids, at least one. Must belong to this company. |
| `new_roles` | int[] | no | `[]` | Role ids to hold on those facilities. Must belong to this company. |
| `is_permanent` | bool | no | `false` | A permanent transfer has no expiry and cannot be reclaimed. |
| `timestamp_to` | datetime | conditional | `null` | Required when `is_permanent` is false, must be in the future. Prohibited when it is true. |
| `include_pending_tasks` | bool | no | `false` | Moves the transferer's pending tasks on the transferred properties to the recipient. |

Sample payload:

```json
{
  "to_user_id": 42,
  "new_facilities": [7, 9],
  "new_roles": [3],
  "is_permanent": false,
  "timestamp_to": "2026-10-01 17:00:00",
  "include_pending_tasks": true
}
```

Response: `data.responsibility_transfer`, status `pending`. Nothing is granted yet. The recipient
is notified.

## Accept

`PATCH /api/v1/app/{company}/access-management/responsibility-transfers/{transfer}/accept`

Only the named recipient. Returns the transfer with status `active` and `accepted.raw` set.
The transferer is notified.

What acceptance grants depends on permanence and on each role's `enforce_on_facility`:

| Transfer | Role enforced | Effect |
| --- | --- | --- |
| Permanent | yes | The existing `facility_role` holder is replaced by the recipient. |
| Permanent | no | A parallel role row is created. The original holder keeps theirs. |
| Temporary | either | A parallel role row is created, removed again on return. |

General property access is **always additive**: nobody loses sight of a property because somebody
else was given sight of it. A role row held by someone other than the transferer is never
displaced; that case falls back to an additive grant.

Answering a transfer that is no longer `pending` returns `422`.

## Decline

`PATCH /api/v1/app/{company}/access-management/responsibility-transfers/{transfer}/decline`

Only the named recipient. Marks the transfer `declined` and notifies the transferer. No property,
role or pending task is touched, and the transferer's outgoing slot is freed.

## Reclaim

`PATCH /api/v1/app/{company}/access-management/responsibility-transfers/{transfer}/reclaim`

Only the transferer, and only on a temporary transfer that is currently in force.

- Permanent transfer: `422`. Use `can_reclaim` on the resource to hide the button rather than
  letting the user discover this.
- Someone other than the transferer: `403`.

Reversal is immediate and does not wait for work in flight. It revokes exactly what the transfer
granted and nothing else, restores the original holder of any reassigned enforced role, and moves
the pending tasks back.

## My transfers

`GET /responsibility-transfers/mine/outgoing` returns `data.responsibility_transfer`: a single
object, or `null`. This backs the floating left-nav indicator, the countdown and the reclaim popup.

`GET /responsibility-transfers/mine/incoming` returns `data.responsibility_transfers`: an array,
possibly several. This backs the accept/decline prompt.

Both include `pending` and in-force transfers, and both are unscoped by company, because the
indicator belongs to the caller rather than to the company they happen to be looking at.

## Resource shape

```json
{
  "id": 12,
  "company_id": 1,
  "from_user_id": 8,
  "from_user": { "id": 8, "name": "..." },
  "to_user_id": 42,
  "to_user": { "id": 42, "name": "..." },
  "new_roles": [3],
  "new_facilities": [7, 9],
  "is_permanent": false,
  "include_pending_tasks": true,
  "status": { "value": "active", "color": "success" },
  "can_reclaim": true,
  "timestamp_to": {
    "raw": "2026-10-01T17:00:00.000000Z",
    "iso": "2026-10-01T17:00:00+00:00",
    "formatted": "01 Oct, 2026 17:00",
    "diff": "in 3 weeks",
    "seconds_remaining": 1814400,
    "has_expired": false
  },
  "accepted": { "raw": "...", "formatted": "...", "diff": "..." },
  "declined": { "raw": null, "formatted": null, "diff": null },
  "reclaimed": { "raw": null, "formatted": null, "diff": null },
  "returned": { "raw": null, "formatted": null, "diff": null },
  "created": { "raw": "...", "formatted": "...", "diff": "..." },
  "updated": { "raw": "...", "formatted": "...", "diff": "..." },
  "items": []
}
```

### Driving the countdown

Tick against `timestamp_to.iso`, and seed the timer from `seconds_remaining` rather than from the
viewer's own clock. That value is the server's view of the remaining time, so a machine with a
skewed clock cannot show a timer that disagrees with the auto-return that is actually going to
fire. It never goes below `0`. On a permanent transfer every field in the object is `null`, which
is the signal to render no countdown at all.

Auto-return runs hourly, so a transfer can sit a few minutes past `has_expired: true` before its
status changes. Treat `has_expired` as "about to return", not as "already returned".

### Transfer items

`items` is present on `show` only. Each entry records one grant the transfer made, which is how
the reversal knows what to undo:

| `grant_type.value` | What was granted | How it reverses |
| --- | --- | --- |
| `facility_user` | General access to a property | The row is detached |
| `facility_role_created` | A new role row | The row is deleted |
| `facility_role_reassigned` | An existing enforced role row changed hands | `previous_user_id` is put back |

```json
{
  "id": 30,
  "responsibility_transfer_id": 12,
  "grant_type": { "value": "facility_role_reassigned", "color": "warning" },
  "facility_id": 7,
  "role_id": 3,
  "facility_role_id": 55,
  "user_id": 42,
  "previous_user_id": 8,
  "is_reversed": false,
  "reversed": { "raw": null, "formatted": null, "diff": null },
  "created": { "raw": "...", "formatted": "...", "diff": "..." }
}
```

Access the recipient already held before the transfer is deliberately absent from this list, which
is what guarantees reversal cannot take it away.

## Errors

| Code | When |
| --- | --- |
| `403` | Wrong person for the action: not the recipient on accept/decline, not the transferer on reclaim, not a participant on `show`. |
| `404` | The transfer does not belong to route `{company}`. |
| `422` | Validation, plus: a second open outgoing transfer, a recipient outside the company or department, reclaiming a permanent transfer, or answering a transfer that is no longer pending. |

Apply shared auth, tenancy, query, and error handling guidance from:

- `docs/frontend/api/v1/app/README.md`
