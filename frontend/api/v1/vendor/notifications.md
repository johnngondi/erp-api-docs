# Notifications API — Vendor Portal

Base route:

`/api/v1/vendor/notifications`

A notification is an informational record of something that happened — a lease application was
received, a document finished its approval chain, here are your alerts for the day. The actionable
counterpart is a **pending task**; a notification never needs to be actioned, only read.

These are Laravel's native database notifications, and the in-app copy is the only one most events
get. Email, SMS and WhatsApp are sent by exception: SMS and WhatsApp only for one-time codes, a
document sent to a tenant or a supplier, an RFQ and reminders to a supplier, and only where the
recipient has an address or a phone number. This API is the in-app copy.

## Endpoints

- `GET /api/v1/vendor/notifications`
- `PATCH /api/v1/vendor/notifications/read-all`
- `PATCH /api/v1/vendor/notifications/{notification}/read`

Authorization: any authenticated Vendor user. **No permission is required** — the query is
always constrained to the caller (`user_id`), so there is nothing further to
authorise. Portal segregation is enforced by the route file's `group:vendor` middleware: a user
signed in to another portal receives `403`.

**What you see here.** `user_group_id` is stamped on every notification when it is sent, so this
endpoint returns the notifications addressed to you **in this portal**; an account that also signs in
elsewhere sees that portal's notifications there, not here. A `null` `user_group_id` means
"unscoped" and is shown everywhere.

`company_id` does **not** narrow this list. This portal carries no company in its URLs, and you may
work across several companies — so everything addressed to you reaches you here, whether it was
filed under a company workspace or against you personally.

## List my notifications

Returns the caller's notifications, newest first.

Supported query params:

- Filters:
  - `filter[type]` — exact. The fully-qualified notification class, as returned in `type`.
  - `filter[unread]` — `true` for unread only, `false` for read only. Omit for both.
- Sorts: `sort=created_at|read_at` (prefix with `-` to reverse). Defaults to newest first.
- Pagination: `per_page` (defaults to the app's configured page size).

### Response

```json
{
  "data": [
    {
      "id": "9b2f8c1e-4d3a-4f21-9b7e-1c2d3e4f5a6b",
      "type": "App\\Notifications\\PropertyManagement\\LeaseApplicationReceivedNotification",
      "company_id": null,
      "user_group_id": 4,
      "read_at": null,
      "read": false,
      "data": {
        "title": "Lease application received",
        "message": "Lease application from Jane Kamau for Riverside Apartments has been received.",
        "lease_application_id": 77,
        "resource_type": "LeaseApplication",
        "resource_id": 77,
        "resource_url": "https://api.example.com/api/v1/vendor/procurement/lpos/12"
      },
      "resource_url": "https://api.example.com/api/v1/vendor/procurement/lpos/12",
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
- `domain` is **not** part of this payload here and `filter[domain]` is **not** supported — a domain
  names an area of the staff app, so it is stamped as null on every notification addressed to a
  Vendor recipient. Sending `filter[domain]` returns `400` rather than a silently empty page. The
  App portal's copy of this endpoint does expose it.

## Notification types

### Quotation reminder

`type`: `App\Notifications\PropertyManagement\Procurement\QuoteSubmissionReminderNotification`

Sent to a supplier who was invited to quote on a procurement request and has not quoted yet. It goes
out **every working day (Monday to Friday) at 09:00** for as long as the request is open for quotes
(its current procurement step is waiting for quotations and the deadline has not passed), and stops
once the supplier submits a quotation or the deadline passes. A supplier receives **at most one per
request per working day**. It is also delivered by email, SMS and WhatsApp where the supplier has an
address or a phone number.

`data` keys:

| Key | Type | Notes |
|---|---|---|
| `title` | string | "Quotation reminder" |
| `message` | string | e.g. "Your quotation for Rewire the lobby at Riverside Apartments is due by 07 Oct 2026, 17:00. Please submit it before then." |
| `procurement_request_id` | integer | The request to quote on. |
| `request_title` | string | The request's title. |
| `facility_name` | string | The property the work is for. |
| `deadline` | string | The quotation deadline, ISO-8601 (e.g. `2026-10-07T17:00:00+03:00`). |
| `resource_type` / `resource_id` / `resource_url` | | The standard resource payload. `resource_url` points at the request on the vendor's open-job page (`/api/v1/vendor/procurement/rfq/open-jobs/{id}`). |

The reminder is queued, so if the scheduled run is ever triggered a second time on the same day before
the queue has drained, a supplier may receive the day's reminder twice.

## Mark one as read

`PATCH /api/v1/vendor/notifications/{notification}/read`

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

`PATCH /api/v1/vendor/notifications/read-all`

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
