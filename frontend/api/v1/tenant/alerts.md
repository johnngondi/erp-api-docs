# Alerts API — Tenant Portal

Base route:

`/api/v1/tenant/alerts`

An alert is a short-lived message raised by the backend for a **single user**, meant to be shown as
a pop-up (e.g. "3 invoices are overdue", "a quote is awaiting your selection").

Alerts are never created or deleted over HTTP. They are raised by whichever backend feature detects
the condition and cleared by that same feature once it is resolved — the backend contract lives in
`docs/frontend/api/v1/app/alerts.md`. Each alert carries an `active_until` timestamp; once it
passes, the alert stops being returned and is deleted by a nightly command.

## Endpoints

- `GET /api/v1/tenant/alerts`

## List my alerts

Returns the authenticated user's **currently active** alerts, newest first. Alerts whose
`active_until` has passed are excluded; alerts with a `null` `active_until` never expire.

Authorization: any authenticated Tenant user. **No permission is required** — the query is
always constrained to the caller (`user_id`), so there is nothing further to
authorise. Portal segregation is enforced by the route file's `group:tenant` middleware: a user
signed in to another portal receives `403`.

Supported query params:

- Filters:
  - `filter[priority]` — exact. One of `high`, `normal`, `low`.
  - `filter[user_group_id]` — exact. The portal the alert was scoped to; `null` means unscoped.
  - `filter[alertable_type]` — exact. Fully-qualified model class of the record the alert is about.
- Sorts: `sort=priority|active_until|created_at` (prefix with `-` to reverse). Defaults to newest first.
- Includes: `include=user`
- Pagination: `per_page` (defaults to the app's configured page size).

### Response

```json
{
  "data": [
    {
      "id": 12,
      "title": "Your invoice is overdue",
      "notes": "Invoice INV-00341 for Riverside Apartments is past its due date.",
      "priority": { "value": "high", "color": "danger", "label": "High" },
      "user_group_id": 2,
      "alertable_type": "App\\Models\\FacilityInvoice",
      "alertable_id": 341,
      "active_until": {
        "raw": "2026-09-10T23:59:59.000000Z",
        "formatted": "10 Sep, 2026 23:59",
        "diff": "1 week from now"
      },
      "created": {
        "raw": "2026-09-03T09:12:44.000000Z",
        "formatted": "03 Sep, 2026 09:12",
        "diff": "2 hours ago"
      },
      "resource_type": "FacilityInvoice",
      "resource_id": 341,
      "resource_url": "https://api.example.com/api/v1/tenant/billing/invoices/341"
    }
  ],
  "links": { "...": "standard pagination links" },
  "meta": { "...": "standard pagination meta" }
}
```

Notes for the front-end:

- `priority.color` is a palette key (`danger`, `primary`, `secondary`, `info`, `success`, `warning`)
  — use it to style the pop-up directly.
- `domain` is **not** part of this payload and `filter[domain]` is **not** supported here — domains
  name areas of the staff app, so they would be permanently null outside it. Sending
  `filter[domain]` returns `400` rather than a silently empty page.
- `alertable_type` / `alertable_id` identify the record the alert is about. Both may be `null`.

### `resource_url`

Every row carries `resource_url`: the **absolute** URL of the record the row is about, already built
for **this** portal's routes. The same record has a different URL in each portal, so never reuse one
across portals and never build your own from `alertable_type`.

It is `null` when the record has no page in this portal, has been deleted, or the row points at
nothing. Render those rows **without** a link rather than constructing a fallback.

`resource_type` and `resource_id` sit alongside it and stay populated even when `resource_url` is
`null`, so you always know what the row is about.
