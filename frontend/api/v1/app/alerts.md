# Alerts API

Domain: `App > Alerts` (cross-cutting)

Base route:

`/api/v1/app/{company}/alerts`

An alert is a short-lived message raised by the backend for a **single user**, intended to be shown
as a pop-up in the front-end (e.g. "3 invoices are overdue", "a quote is awaiting your selection").

Alerts are never created or deleted over HTTP. They are raised by whichever feature detects the
condition, and cleared by that same feature once the condition is resolved (see
[Raising an alert](#raising-an-alert-backend)). Each alert carries an `active_until` timestamp:
once it passes, the alert stops being returned and is deleted by a nightly command.

## Endpoints

- `GET /alerts`

## List my alerts

`GET /api/v1/app/{company}/alerts`

Returns the authenticated user's **currently active** alerts for the route company, newest first.
Alerts whose `active_until` has passed are excluded; alerts with a `null` `active_until` never
expire and are always returned.

Authorization: any authenticated app user. No permission is required — the query is always scoped
to the caller (`user_id`) and the route company.

Supported query params:

- Filters:
  - `filter[priority]` — exact. One of `high`, `normal`, `low`.
  - `filter[domain]` — exact. One of `access-management`, `property-management`,
    `facility-management`, `project-management`.
  - `filter[user_group_id]` — exact. The portal the alert was scoped to; `null` means unscoped.
  - `filter[alertable_type]` — exact. Fully-qualified model class of the record the alert points at.
- Sorts: `sort=priority|domain|active_until|created_at` (prefix with `-` to reverse).
  Defaults to newest first.
- Includes: `include=user`
- Pagination: `per_page` (defaults to the app's configured page size).

### Response

```json
{
  "data": [
    {
      "id": 12,
      "title": "Three invoices are overdue",
      "notes": "Review the overdue invoices for Riverside Apartments.",
      "priority": { "value": "high", "color": "danger" },
      "domain": {
        "value": "property-management",
        "color": "primary",
        "label": "Property Management"
      },
      "user_group_id": 3,
      "alertable_type": "App\\Models\\FacilityInvoice",
      "alertable_id": 341,
      "active_until": {
        "raw": "2026-08-10T23:59:59.000000Z",
        "formatted": "10 Aug, 2026 23:59",
        "diff": "1 week from now"
      },
      "created": {
        "raw": "2026-08-03T09:12:44.000000Z",
        "formatted": "03 Aug, 2026 09:12",
        "diff": "2 hours ago"
      },
      "resource_type": "FacilityInvoice",
      "resource_id": 341,
      "resource_url": "https://api.example.com/api/v1/app/12/property-management/lease-management/invoices/341"
    }
  ],
  "links": { "...": "standard pagination links" },
  "meta": { "...": "standard pagination meta" }
}
```

Notes for the front-end:

- `priority.color` and `domain.color` are palette keys (`danger`, `primary`, `secondary`, `info`,
  `success`, `warning`) — use them to style the pop-up directly.
- `domain.label` is already humanised and translated; don't un-kebab `domain.value` yourself.
- `domain` may be `null` for alerts that aren't tied to one area of the app. It is only ever set on
  alerts belonging to the **app (staff) portal** — the four domains name areas of the staff app, and
  the tenant, landlord and vendor portals are not divided that way. An alert raised for any other
  portal stores `domain` as `null`.
- `alertable_type` / `alertable_id` identify the record the alert is about. Both may be `null`.

### `resource_url`

Every row carries `resource_url`: the **absolute** URL of the record the alert is about, already
built for this portal's routes. The same record has a different URL in each portal, so never reuse
one across portals and never build your own from `alertable_type`.

It is `null` when the record has no page in this portal, has been deleted, or the alert points at
nothing — render those rows without a link rather than constructing a fallback. `resource_type` and
`resource_id` stay populated either way, so you always know what the row is about.

### The same endpoint on other portals

Alerts are readable from every portal. The payload is identical apart from `domain`, which the App
portal alone returns:

- `docs/frontend/api/v1/tenant/alerts.md`
- `docs/frontend/api/v1/landlord/alerts.md`
- `docs/frontend/api/v1/vendor/alerts.md`

## Raising an alert (backend)

Build an `App\Data\Alert\AlertData` and hand it to `CreateAlertAction`:

```php
use App\Actions\Common\Alert\CreateAlertAction;
use App\Data\Alert\AlertData;
use App\Enums\Domain;
use App\Enums\Priority;
use App\Models\UserGroup;

$data = AlertData::from([
    'user_id' => $invoice->facility->manager_id,
    'title' => __('Three invoices are overdue'),
    'notes' => __('Review the overdue invoices for :facility', ['facility' => $invoice->facility->name]),
    'active_until' => now()->addWeek()->toDateTimeString(),
    'priority' => Priority::High->value,
    'user_group_id' => UserGroup::idForTitle(UserGroup::APP),
    'domain' => Domain::PropertyManagement->value,
])->withAlertable($invoice);

app(CreateAlertAction::class)->execute($data);
```

`user_group_id` is the portal the recipient acts in (`App\Models\UserGroup` — `app`, `tenant`,
`vendor`, `landlord`). It is optional; leaving it null means "unscoped", which is also what every
row written before portal scoping existed carries.

**`domain` is only kept for the `app` portal.** `CreateAlertAction` forces it to `null` when the
alert belongs to another portal, whatever the caller passed, and `AlertData::validateAndCreate()`
rejects the pairing outright — so a vendor, tenant or landlord alert can never carry one. An alert
with no `user_group_id` is unscoped rather than "another portal", so it keeps its domain.
`App\Actions\Common\PendingTask\CreatePendingTaskAction` enforces the identical rule for pending
tasks.

`company_id` is filled automatically from the route company inside a request. **When raising an
alert from a job, command or scheduler you must pass `company_id` explicitly** — the `CompanyScoped`
global scope is inert outside of HTTP.

Sample JSON payload for `AlertData::from(...)`:

```json
{
  "company_id": 1,
  "user_id": 4,
  "user_group_id": 3, // the recipient's portal; null = unscoped. Only the `app` group may carry a domain
  "title": "Three invoices are overdue",
  "notes": "Review the overdue invoices for Riverside Apartments.",
  "active_until": "2026-08-10 23:59:59",
  "priority": "high", // high | normal | low
  "domain": "property-management" // access-management | property-management | facility-management | project-management
}
```

## Clearing an alert (backend)

Once the action that caused the alert has been performed, clear it via `DeleteAlertAction`. Use the
`forAlertable` scope to find the alerts raised against a record:

```php
use App\Actions\Common\Alert\DeleteAlertAction;
use App\Models\Alert;

$deleteAlert = app(DeleteAlertAction::class);

Alert::query()->forAlertable($invoice)->get()
    ->each(fn (Alert $alert) => $deleteAlert->execute($alert));
```

Alerts are hard-deleted — they are transient UI state, not an audit record.

## The sibling concept: pending tasks

An alert is a short-lived pop-up; a **pending task** (`App\Models\PendingTask`) is an item a
specific user must action before it is considered done. The two are deliberately symmetrical:
`PendingTaskData` + `withTaskable()` mirror `AlertData` + `withAlertable()`,
`CreatePendingTaskAction` / `DeletePendingTaskAction` mirror the alert actions above, and
`PendingTask::forTaskable()` mirrors `Alert::forAlertable()`.

Pending tasks come from two producers:

- **Approval-sourced** — `App\Services\ApprovalService` raises one per actor as an approval step is
  prompted (`taskable` is the `ApprovalStep`, `approval_step_id` set) and clears them as steps
  advance. Nothing else needs to manage their lifecycle.
- **Direct** — e.g. "vendor, upload a jobcard on this LPO". The `taskable` is the resource itself.
  These follow the same contract as alerts: **raised by whichever feature detects the condition,
  cleared by that same feature once it is resolved.** Nothing clears a direct task on a timer and
  nothing clears one over HTTP.

Neither kind is ever written over HTTP. The read endpoints for pending tasks are delivered
separately.

## Automatic cleanup

`alerts:purge-expired` runs nightly at `01:15` (registered in `app/Console/Kernel.php`) and deletes
every alert whose `active_until` has passed, across all companies. It routes each row through
`DeleteAlertAction`.
