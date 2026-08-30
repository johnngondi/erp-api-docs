# Notifications API — App (Staff) Portal

Base route:

`/api/v1/app/{company}/notifications`

A notification is an informational record of something that happened — a lease application was
received, a document finished its approval chain, here are your alerts for the day. The actionable
counterpart is a **pending task**; a notification never needs to be actioned, only read.

These are Laravel's native database notifications. The same events are also delivered by email and
SMS where the recipient has an address or a phone number; this API is the in-app copy.

## Endpoints

- `GET /api/v1/app/{company}/notifications`
- `PATCH /api/v1/app/{company}/notifications/read-all`
- `PATCH /api/v1/app/{company}/notifications/{notification}/read`

Authorization: any authenticated App (Staff) user. **No permission is required** — the query is
always constrained to the caller (`user_id`), so there is nothing further to
authorise. Portal segregation is enforced by the route file's `group:app` middleware: a user
signed in to another portal receives `403`.

**What you see here.** Two things narrow the list beyond "your own notifications":

- **Workspace.** A notification with a `company_id` belongs under that company and is returned only
  while that company is in the URL. One with `company_id: null` belongs to **you** rather than to a
  workspace, and is returned whichever company you are in.
- **Portal.** `user_group_id` is stamped on every notification when it is sent, so an account that
  signs in to more than one portal sees each portal's own notifications rather than one merged list.
  A `null` here means "unscoped" and is shown everywhere — which is what rows written before this
  scoping existed carry.

## List my notifications

Returns the caller's notifications, newest first.

Supported query params:

- Filters:
  - `filter[type]` — exact. The fully-qualified notification class, as returned in `type`.
  - `filter[unread]` — `true` for unread only, `false` for read only. Omit for both.
  - `filter[domain]` — exact. One of `access-management`, `property-management`,
    `facility-management`, `project-management`. **App portal only.**
- Sorts: `sort=created_at|read_at|domain` (prefix with `-` to reverse). Defaults to newest first.
- Pagination: `per_page` (defaults to the app's configured page size).

### Response

```json
{
  "data": [
    {
      "id": "9b2f8c1e-4d3a-4f21-9b7e-1c2d3e4f5a6b",
      "type": "App\\Notifications\\PropertyManagement\\LeaseApplicationReceivedNotification",
      "domain": {
        "value": "property-management",
        "color": "primary",
        "label": "Property Management"
      },
      "company_id": 12,
      "user_group_id": 3,
      "read_at": null,
      "read": false,
      "data": {
        "title": "Lease application received",
        "message": "Lease application from Jane Kamau for Riverside Apartments has been received.",
        "lease_application_id": 77,
        "resource_type": "LeaseApplication",
        "resource_id": 77,
        "resource_url": "https://api.example.com/api/v1/app/12/property-management/lease-management/lease-applications/77"
      },
      "resource_url": "https://api.example.com/api/v1/app/12/property-management/lease-management/lease-applications/77",
      "created": {
        "raw": "2026-09-03T09:12:44.000000Z",
        "formatted": "03 Sep, 2026 09:12",
        "diff": "2 hours ago"
      }
    }
  ],
  "links": { "...": "standard pagination links" },
  "meta": { "...": "standard pagination meta" }
}
```

Notes for the front-end:

- `id` is a **UUID string**, not an integer.
- `company_id` is the workspace the notification was filed under, or `null` when it belongs to you
  rather than to a company.
- `user_group_id` is the portal it was sent in. It is stamped on every notification — unlike
  `domain` and `company_id`, the portal of whoever is being notified is always known at send time.
- `data` is the payload frozen at send time. Its keys vary by notification `type`; `title` and
  `message` are present on every notification in this milestone and are safe to render directly.
- `resource_url` is lifted out of `data` to the top level so all three inbox resources read the same
  way. It was resolved **for this recipient's portal at send time**, which is why it is absolute and
  why it is never recomputed on read. It is `null` for anything stored without one — a notification
  predating this milestone, or a daily alerts digest, which is about many records at once.
- `domain` names the area of the staff app the notification belongs to, and is returned **only** on
  this portal. It is `null` for a notification that belongs to no single area — an alerts digest
  bundling several areas, or anything sent before domains were stamped.

## Mark one as read

`PATCH /api/v1/app/{company}/notifications/{notification}/read`

Marks a single notification read and returns it. A notification id belonging to another user returns
`404` — the lookup never leaves the caller's own rows.

```json
{
  "data": {
    "message": "Notification marked as read.",
    "notification": {
      "id": "9b2f8c1e-4d3a-4f21-9b7e-1c2d3e4f5a6b",
      "read": true,
      "read_at": "2026-09-03T11:00:00.000000Z",
      "...": "every other field as in the list response"
    }
  }
}
```

## Mark all as read

`PATCH /api/v1/app/{company}/notifications/read-all`

Marks every unread notification the caller holds as read, and only theirs. Returns how many rows
were affected.

```json
{
  "data": {
    "message": "All notifications marked as read.",
    "marked": 7
  }
}
```

`read-all` is registered before the `{notification}` route, so it is never mistaken for a
notification id.
