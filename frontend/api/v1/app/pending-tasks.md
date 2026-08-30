# Pending Tasks API — App (Staff) Portal

Base route:

`/api/v1/app/{company}/pending-tasks`

A pending task is the general **"something is waiting on you"** record: any item a specific user
must action before it is considered done. Approvals are one producer of pending tasks, not the
definition of one — a vendor asked to upload a jobcard against an LPO holds one too.

## Endpoints

- `GET /api/v1/app/{company}/pending-tasks`

## List my pending tasks

Returns the authenticated user's own tasks, newest first.

Authorization: any authenticated App (Staff) user. **No permission is required** — the query is
always constrained to the caller (`user_id`) and to the route company, so there is nothing further to
authorise. Portal segregation is enforced by the route file's `group:app` middleware: a user
signed in to another portal receives `403`.

Supported query params:

- Filters:
  - `filter[user_group_id]` — exact. The portal the task was scoped to; `null` means unscoped.
  - `filter[taskable_type]` — exact. Fully-qualified model class of the row the task hangs off.
  - `filter[domain]` — exact. One of `access-management`, `property-management`,
    `facility-management`, `project-management`.
- Sorts: `sort=due_at|created_at|domain` (prefix with `-` to reverse). Defaults to newest first.
- Includes: `include=user`, `include=approvalStep`, `include=taskable`
- Pagination: `per_page` (defaults to the app's configured page size).

### Response

```json
{
  "data": [
    {
      "id": 88,
      "title": "Approval required: Invoice",
      "description": "Step 1 is awaiting your action.",
      "domain": {
        "value": "property-management",
        "color": "primary",
        "label": "Property Management"
      },
      "user_group_id": 3,
      "user_group": { "id": 3, "title": "app", "label": "Staff" },
      "taskable_type": "App\\Models\\ApprovalStep",
      "taskable_id": 51,
      "approval_step_id": 51,
      "due_at": { "raw": null, "formatted": null, "diff": null },
      "created": {
        "raw": "2026-09-03T09:12:44.000000Z",
        "formatted": "03 Sep, 2026 09:12",
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

- `domain` names an area of the staff app and is returned **only** on this portal. It may be `null`
  on rows written before portal scoping existed — treat null as "unscoped, show everywhere".
- `user_group` is present only when the relation is loaded; `user_group_id` is always present.
- `approvalStep` appears when you ask for `include=approvalStep`, carrying `id`, `step_order`,
  `status` and `prompted_at`.

## Two kinds of task, one payload

- **Approval-sourced** — `taskable_type` is `App\\Models\\ApprovalStep` and `approval_step_id` is set.
- **Direct** — `taskable_type` is the record itself (an LPO awaiting a jobcard, say) and
  `approval_step_id` is `null`.

**You do not need to tell them apart to link the user through.** `resource_url` is already resolved
to the record the user has to open: for an approval-sourced task that is the step's *approvable*
(the invoice, LPO or LOO), not the step itself. Use `taskable_type` / `taskable_id` for grouping or
iconography only.

### `resource_url`

Every row carries `resource_url`: the **absolute** URL of the record the row is about, already built
for **this** portal's routes. The same record has a different URL in each portal, so never reuse one
across portals and never build your own from `taskable_type`.

It is `null` when the record has no page in this portal, has been deleted, or the row points at
nothing. Render those rows **without** a link rather than constructing a fallback.

`resource_type` and `resource_id` sit alongside it and stay populated even when `resource_url` is
`null`, so you always know what the row is about.

## Where tasks come from (backend)

Tasks are **never** written over HTTP. There is no create, update or delete endpoint, by design.

- **Approval-sourced tasks** are raised by `App\Services\ApprovalService` as an approval step is
  prompted, one per actor, and cleared automatically as the chain advances. Nothing else has to
  manage their lifecycle.
- **Direct tasks** are raised by whichever feature needs the action, through
  `App\Actions\Common\PendingTask\CreatePendingTaskAction`, and cleared by that same feature
  through `DeletePendingTaskAction` over `PendingTask::forTaskable($record)` once the condition is
  resolved. Nothing clears a direct task on a timer. A feature that raises one without clearing it
  leaves a permanent phantom entry in someone's list.

This mirrors the alert contract documented in `docs/frontend/api/v1/app/alerts.md`.
