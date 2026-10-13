# Pending Tasks API — App (Staff) Portal

Base route:

`/api/v1/app/{company}/pending-tasks`

A pending task is the general **"something is waiting on you"** record: any item a specific user
must action before it is considered done. Approvals are one producer of pending tasks, not the
definition of one — a vendor asked to upload a jobcard against an LPO holds one too.

## Endpoints

- `GET /api/v1/app/{company}/pending-tasks`
- `GET /api/v1/app/{company}/pending-tasks/resource-types`
- `GET /api/v1/app/{company}/pending-tasks/{pendingTask}/comparables`
- `POST /api/v1/app/{company}/pending-tasks/decisions/otp`
- `POST /api/v1/app/{company}/pending-tasks/decisions`

## List my pending tasks

Returns the authenticated user's own tasks, newest first.

Authorization: any authenticated App (Staff) user. **No permission is required** — the query is
always constrained to the caller (`user_id`) and to the route company, so there is nothing further to
authorise. Portal segregation is enforced by the route file's `group:app` middleware: a user
signed in to another portal receives `403`.

**Portal.** Each row carries the `user_group_id` of the portal it was raised for, and only this
portal's rows are returned: an account signed in to two portals (say App and Vendor) sees each
portal's own tasks rather than one merged list. A `null` `user_group_id` means "unscoped" and is
shown everywhere, which is what rows written before this scoping existed carry.

Supported query params:

- Filters:
  - `filter[user_group_id]` — exact. The portal the task was scoped to; `null` means unscoped.
  - `filter[taskable_type]` — exact. Fully-qualified model class of the row the task hangs off.
  - `filter[domain]` — exact. One of `access-management`, `property-management`,
    `facility-management`, `project-management`.
  - `filter[resource_type]` — the record behind the task, by **bare model name**
    (`FacilityProcurementRequest`, `FacilityInvoice`, `Loo`, ...). An approval task matches on
    the record being approved, not on `ApprovalStep`. Several names may be comma-separated. A name
    the server does not know returns an empty page, not an error. The names on offer come from
    [`resource-types`](#kinds-of-record-among-my-tasks).
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
      "context": {
        "property": { "id": 4, "name": "Kahawa House" },
        "lease_id": 77,
        "tenant": { "id": 120, "name": "Acme Traders Ltd" },
        "creator": { "id": 9, "name": "Jane Clerk" }
      },
      "summary": {
        "kind": "Invoice",
        "reference": "#341",
        "amount": "125000.00",
        "currency": "KES",
        "party": { "label": "Tenant", "name": "Acme Traders Ltd" },
        "facts": [
          { "label": "Date", "value": "03 Sep, 2026" },
          { "label": "Due", "value": "10 Sep, 2026" }
        ],
        "comparables": null
      },
      "can_decide": true,
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

### `summary` and `can_decide` (this portal only)

What the approvals inbox shows about the record at a glance, and whether the row can be decided
there. Both are computed from what the list already loads, so they cost no extra request. Every
`summary` key is present on every row, `null` (or `[]` for `facts`) where the record cannot answer.

| Key | Meaning |
|---|---|
| `kind` | What the record is, in words: `LPO`, `Invoice`, `Credit note`, `Receipt`, `Remittance`, `Contract`, `Letter of offer`, `Exit notice`, `Lease`, `Lease application`, `Procurement request`, `Bill`, `Payment voucher`, `Quotation`, `Budget`, `Ticket`, `Utility`, `Property`, `Responsibility transfer` |
| `reference` | The number people know it by (LPO document number, CU invoice number, receipt transaction number, LOO `our_ref`, contract or ticket title...), else `#{id}`. A procurement request reads `Request#{id} - {title}` |
| `amount` | The money it is about, as a **2-decimal string** (`"2500.50"`), or `null`. Format it with `currency` |
| `currency` | Currency code of `amount`, or `null` |
| `party` | `{ label, name }` of the outside party: `Supplier`, `Tenant`, `Landlord`, `Payee`, `Applicant` or `From`. `label` says which, since one column mixes them. `null` when there is none |
| `facts` | Up to three `{ label, value }` pairs worth knowing (expense type and account on an LPO or bill, dates on a document, the type of a procurement request). Values are display strings |
| `comparables` | `"quotes"` (work request) or `"options"` (purchase request) on a procurement-request row: the row has a [comparables](#comparables-procurement-requests) popup. `null` on every other row |

A procurement request's `party` is the supplier of the quote it stands behind (selected, else
recommended, else the system's pick) and `amount` that quote's total. The amount stays `null` until
the request is in quotes review past its deadline (sealed quotes), exactly as on the request page.
A purchase request has no `party`; its `amount` is the selected (else recommended) option's total.

`can_decide` is `true` when the row can be approved, sent back or rejected from the inbox with just
a comment:

- an approval task (`taskable_type` `App\Models\ApprovalStep`) whose step is `pending` and assigned
  to the reader (invoices, credit notes, receipts, remittances, contracts, direct LPOs, LOOs, leases,
  exit notices);
- a procurement-request task whose current step is `pending`, lists the reader as an actor, and
  neither handles procurement (quotes, supplier or option choice) nor picks the expense category.

Everything else (a step that handles the quotes, a "changes requested" task, a lease application,
a jobcard...) is `false`: open the record to act on it. `can_decide` is advisory; the decision
endpoint re-checks everything.

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

Direct tasks raised today:

| Feature | Taskable | Who | Raised and cleared by |
|---|---|---|---|
| Procurement request steps | `FacilityProcurementRequest` | The current step's actors, or the creator when the first step sends it back | `SyncProcurementRequestTasksAction`, every time the request moves. See [Procurement → Pending tasks on a request](property-management/procurement.md#pending-tasks-on-a-request) |
| Lease applications | `LeaseApplication` | The reviewers, then the Letter of Offer role | `SyncLeaseApplicationTasksAction` |
| LPO jobcard (supplier portal) | `FacilityProcurementLpo` | The supplier | `IssueLpoToVendorService`, cleared by `SubmitLpoDocumentController` |

This mirrors the alert contract documented in `docs/frontend/api/v1/app/alerts.md`.

## Kinds of record among my tasks

`GET /api/v1/app/{company}/pending-tasks/resource-types`

The options of the type filter: each kind of record among the caller's open tasks for this portal
and company, once, with how many tasks it has. Caller-scoped and permission-free, like the list.
Sorted by label. An approval task counts under the record being approved.

```json
{
  "data": [
    { "value": "FacilityInvoice", "label": "Facility Invoices", "count": 3 },
    { "value": "FacilityProcurementRequest", "label": "Facility Procurement Requests", "count": 2 }
  ]
}
```

Send `value` back as `filter[resource_type]`. The same endpoint exists on the tenant, vendor and
landlord portals (without `{company}`).

## Comparables (procurement requests)

`GET /api/v1/app/{company}/pending-tasks/{pendingTask}/comparables`

What a procurement-request task is choosing between, for the inbox's popup: the quotes submitted on
a work request, or the price options of a purchase request. Read-only, with no side effects.
`{pendingTask}` is the task id (numeric).

- `404` when the task is not one of the caller's own (in this company and portal).
- `422` keyed `pending_task` when the task is not about a procurement request.

Work request (`comparables.type` = `quotes`). `items` are the quotes still in play (shelved ones
left out), shaped like the request page's bids (`FacilityProcurementRequestBidResource`). `upload`
is the quotation file (`null` when none was attached); preview it from `upload.source_url`. The
figures (`amount`, `discount_*`, `amount_after_discount`, `tax`, `total`) are present only once the
request is in quotes review past its deadline, as on the request page.

```json
{
  "data": {
    "request": { "id": 51, "title": "Rewire the lobby", "type": "work" },
    "comparables": {
      "type": "quotes",
      "items": [
        {
          "id": 301,
          "notes": null,
          "procurement_request_id": 51,
          "currency": { "id": 1, "code": "KES", "name": "Kenyan Shilling" },
          "vendor": { "id": 88, "name": "Sparkle Electricals Ltd", "email": "info@sparkle.test", "profile_photo_url": "https://ui-avatars.com/api/?name=S+E+L" },
          "amount": "1500.00000",
          "discount_type": "none",
          "discount_value": "0.00",
          "discount_amount": "0.00000",
          "amount_after_discount": "1500.00000",
          "tax": "0.00000",
          "total": "1500.00000",
          "upload": { "id": 9, "file_name": "quotation.pdf", "source_url": "https://api.example.com/..." },
          "delivery": { "raw": "2026-10-20T00:00:00.000000Z", "formatted": "20 Oct, 2026", "diff": "1 week from now" },
          "status": { "value": "recommended", "color": "primary" },
          "is_computer_recommended": true,
          "round": 1,
          "shelved": null,
          "is_selected": true,
          "created": { "raw": "2026-10-10T09:00:00.000000Z", "formatted": "10 Oct, 2026", "diff": "3 days ago" }
        }
      ]
    }
  }
}
```

Purchase request (`comparables.type` = `options`). `items` are the price options, raw:

```json
{
  "data": {
    "request": { "id": 52, "title": "Office chairs", "type": "purchase" },
    "comparables": {
      "type": "options",
      "items": [
        { "id": 7, "title": "Ergonomic chairs", "type": "split", "total": "1400.00000", "is_computer_recommended": false, "is_selected": true }
      ]
    }
  }
}
```

## Batch decisions

Approve, send back or reject several tasks at once from the inbox, authorised by **one** one-time
code. Only rows with `can_decide: true` are worth sending; anything else comes back `ok: false`.

### Request a code

`POST /api/v1/app/{company}/pending-tasks/decisions/otp` (no body; `throttle:5,1`)

Sends a 6-digit code by email, SMS and WhatsApp (every channel the person can be reached on). It
works for 10 minutes, once, and asking again spends the previous code.

```json
{
  "data": {
    "message": "Approval code sent.",
    "channels": ["email", "sms", "whatsapp"],
    "phone": "•••• ••• 123",
    "email": "j••••n@example.com",
    "expires_at": "2026-10-13T10:10:00+03:00",
    "resend_after": 60
  }
}
```

`phone` / `email` are masked and **left out** when the person has none. Show a resend countdown
from `resend_after` (seconds).

Errors (`422`):

| Key | When |
|---|---|
| `otp` | A code was sent less than `resend_after` seconds ago ("Wait 42 seconds..."), or more than 10 codes in the hour |
| `otp_send_failed` | Nothing could be sent (no email or phone); no code is live |

### Decide

`POST /api/v1/app/{company}/pending-tasks/decisions` (`throttle:10,1`)

```json
{
  "code": "482913",
  "decisions": [
    { "pending_task_id": 88, "status": "approve" },
    { "pending_task_id": 90, "status": "review", "comment": "Attach the delivery note." },
    { "pending_task_id": 91, "status": "reject", "comment": "Duplicate of #341." }
  ]
}
```

- `code` — required, 6 digits.
- `decisions` — required, 1 to 50 items.
- `decisions.*.pending_task_id` — required integer, no repeats.
- `decisions.*.status` — `approve` | `review` (send back for changes) | `reject`.
- `decisions.*.comment` — required for `review` and `reject`, max 2000 characters.

The code is checked first. A wrong, expired, used or exhausted code is a `422` keyed `code` and
**nothing is decided**; five wrong tries spend the code (ask for a new one). Input errors are the
usual `422` (`decisions.0.comment`, ...) and do not use up the code.

Then each decision runs on its own: one that is refused does not undo the others. The response
always lists every decision, in the order sent:

```json
{
  "data": {
    "message": "2 of 3 decisions submitted.",
    "results": [
      { "pending_task_id": 88, "status": "approve", "ok": true, "message": "Approved." },
      { "pending_task_id": 90, "status": "review", "ok": true, "message": "Sent back for changes." },
      { "pending_task_id": 91, "status": "reject", "ok": false, "message": "This step needs the full review form. Open the request to act on it." }
    ],
    "processed": 2,
    "failed": 1
  }
}
```

A decision comes back `ok: false` when the task is not one of the caller's (or is already done:
"This task was not found..."), when it cannot be decided from the inbox ("Open the record to act on
it."), when the step needs the procurement review form, when the caller may not act on the step now
("You are not allowed to act on this step now."), or with the engine's own validation message.

What a decision does is exactly what the record's own page does: an approval step goes through the
approval engine (the next step is prompted, the document gets its final status on the last one, a
send-back goes to the checkpoint or the submitter), a procurement step through the request's step
review (the next step becomes current and its actors get a task). Afterwards re-read the list and
the inbox summary: decided tasks are gone and new ones may exist for others.
