# Inbox summary and `me/permissions` — Frontend

Two light endpoints meant for polling and for refreshing the permission map.

## Inbox summary

Poll this **instead of** the four inbox reads (unread notifications, pending tasks,
`responsibility-transfers/mine/outgoing`, `responsibility-transfers/mine/incoming`). Fetch the
full lists only when the user opens the corresponding panel.

| Portal | URL |
|---|---|
| App (Staff) | `GET /api/v1/app/{company}/inbox/summary` |
| Tenant | `GET /api/v1/tenant/inbox/summary` |
| Vendor | `GET /api/v1/vendor/inbox/summary` |
| Landlord | `GET /api/v1/landlord/inbox/summary` |

Authorization: any signed-in user of that portal. No permission required. Wrong portal, or an App
company you are not a member of: `403`. Not signed in: `401`.

```json
{
  "data": {
    "unread_notifications": 3,
    "pending_tasks": 2,
    "active_alerts": 1,
    "open_tickets": 4,
    "outgoing_transfer": {
      "id": 12,
      "status": "active",
      "timestamp_to": "2026-09-27T10:00:00+03:00",
      "to_user": { "id": 5, "name": "Jane" }
    },
    "incoming_transfers": 1,
    "session": { "locked": false, "locks_in_seconds": 212, "idle_minutes": 5 }
  }
}
```

Counts are for **this portal**: a person signed in to two portals (say App and Vendor) sees each
portal's own tasks, alerts and notifications, plus legacy rows that carry no portal.

- `unread_notifications`: what `GET {portal}/notifications?filter[unread]=true` would total.
- `pending_tasks`: what `GET {portal}/pending-tasks` would total.
- `active_alerts`: what `GET {portal}/alerts` would total.
- `open_tickets`: App only, the Property Management dashboard's open-tickets badge
  (`open_tickets.total` with no filters). `null` on other portals.
- `outgoing_transfer`: App only. The caller's open outgoing responsibility transfer **in this
  company**, or `null`.
  `timestamp_to` is `null` for a permanent transfer. `status` is `pending`, `accepted` or `active`.
  Fetch `mine/outgoing` for the full object (countdown, `can_reclaim`) when you need it.
  `null` on other portals.
- `incoming_transfers`: App only, how many rows `mine/incoming` would return: open transfers naming
  the caller in this company, pending and held (`accepted`/`active`) alike. `0` on other portals.
- `session`: the PIN lock as the server sees it, on every portal. `locked` is whether the token
  has idled past the window and every ordinary request now comes back `423`. **When it is `true`,
  put the lock screen up** - this poll answers behind the lock and does not count as activity,
  so it is how a tab nobody is near learns it has been locked. `locks_in_seconds` counts down to
  the lock and is `null` when the server will never lock this session (enforcement off, or no
  PIN set), which reads as "run nothing". `idle_minutes` is the window, for the label. See
  [session-pin.md](../common/session-pin.md).

Every key is always present on every portal.

## `me/permissions`

`GET /api/v1/app/{company}/me/permissions`

```json
{
  "data": {
    "permissions": ["view-facility", "create-facility", "view-lease", "view-role", "generate-report"],
    "modules": {
      "Property Management": ["view-facility", "create-facility", "view-lease"],
      "Access Management": ["view-role", "generate-report"]
    }
  }
}
```

- `permissions`: every `view-*`, `create-*`, `generate-*` and `export-*` permission your App roles
  give you in this company, across all modules, each name once.
- `modules`: the same names grouped by module. `modules["Property Management"]` equals
  `data.permissions` of `GET /api/v1/app/{company}/property-management`, and
  `modules["Access Management"]` equals that of `GET /api/v1/app/{company}/access-management`.
  A module you hold nothing in is absent. `{}` when you hold nothing.

Use it to refresh the permission map (after switching modules, after a `403`) without running a
dashboard.
