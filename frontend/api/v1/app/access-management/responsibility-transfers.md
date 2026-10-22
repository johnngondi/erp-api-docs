# Access Management: Responsibility Transfers

Base prefix:

`/api/v1/app/{company}/access-management`

A responsibility transfer hands one staff member's properties and roles to a colleague, either
temporarily (for leave) or permanently (a real handover). Accepting it hands over the work as well
as the access: open approval and procurement steps, pending tasks, and the documents the
transferer has in flight on those properties. See [What accept hands over](#what-accept-hands-over).

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
- `PATCH /responsibility-transfers/{transfer}/withdraw`

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
| `active` | A **temporary** transfer, accepted. The grants are live until reclaim or `timestamp_to`. |
| `completed` | A **permanent** transfer, accepted. Everything changed hands for good, so it closes at once: it is not open, not in force, cannot be reclaimed, and never blocks another transfer. `accepted` carries its timestamp. |
| `declined` | The recipient said no. Nothing changed hands. |
| `reclaimed` | The transferer pulled it back. Grants and handovers reversed. |
| `returned` | A temporary transfer reached `timestamp_to` and reversed itself. |
| `expired` | Nobody answered before `timestamp_to` passed. Nothing to reverse. |
| `withdrawn` | The transferer took it back before it was answered. Nothing changed hands. |
| `accepted` | Reserved. Accept writes `active` or `completed`; treat `accepted` as in force if you filter on status. |

Open (counts against the overlap rule, listed in `mine/*`): `pending`, `accepted`, `active`.
In force (reclaimable when temporary, routes new steps to the recipient): `accepted`, `active`.

Transitions:

```
pending ──accept (temporary)──▶ active ──reclaim───▶ reclaimed
   │                              │
   │                              └──timestamp_to──▶ returned
   ├──accept (permanent)──▶ completed   (terminal: nothing to reclaim or return)
   ├──decline───▶ declined     (recipient)
   ├──withdraw──▶ withdrawn    (transferer)
   └──timestamp_to──▶ expired
```

So `pending ──accept (permanent)──▶ completed` in one step: a permanent transfer never passes
through `active`.

Every status object is the house `{ value, color }` pair. `completed` is `primary`.

## Rules the UI must respect

- **Several outgoing transfers at once, on properties that do not overlap.** A new transfer's
  `new_facilities` must not share a property with any of the caller's **open** transfers
  (`pending`, `accepted`, `active`), counted across every company they belong to. An overlap is
  `422` on `new_facilities`, naming the properties. A `completed` permanent transfer is not open
  and never blocks. The initiate form may be opened at any time, including while other
  outgoing transfers are pending or in force: there is no "you already have a transfer" block.
  Leave properties that are in one of `mine/outgoing`'s transfers out of the property picker
  rather than letting the user discover the `422`, and when that leaves nothing to pick, say so
  and disable submit.
- **You can only hand over what you hold.**
  - Every offered property must be one the caller is allocated to (`facility_user`), unless the
    caller owns the company (`companies.user_id`), which reaches every property. Otherwise `422`
    on `new_facilities`, naming the properties.
  - Every offered role must be one the caller holds in the company, or one they are named for
    (`facility_role`) on one of the offered properties. Otherwise `422` on `new_roles`, naming the
    roles.
- **A temporary transfer goes one level only.** What the caller was *lent* by a temporary transfer
  still in force cannot be transferred on: if A lends B a property, B cannot pass it to C.
  - A property B is allocated to only because of that transfer: `422` on `new_facilities`, `You
    hold <properties> only through a temporary transfer, so you cannot transfer them on.`
  - A role B is named for on the offered properties only because of that transfer: `422` on
    `new_roles`, `You hold <roles> on these properties only through a temporary transfer, so you
    cannot transfer them on.`
  - What B held before, holds company-wide, or was given by a **permanent** transfer is B's own and
    can be transferred as usual. Once the temporary transfer ends, what it lent is gone anyway.
  - The `422` names the properties or roles; show it beside the picker. To mark them up front, a
    user's active incoming temporary transfers (`GET /{id}` returns their `items`) list what each one
    lent: a `facility_user` item is a lent property, a `facility_role_*` item a lent role on one.
- **Many incoming at once** is normal. `mine/incoming` returns an array.
- The recipient must be an **active member of the same company and the same department** as the
  transferer. Department is matched exactly, null included. Anything else is `422` on `to_user_id`.
- A **temporary** transfer requires `timestamp_to`, and it must be in the future.
- A **permanent** transfer must not send `timestamp_to`, has no countdown, rejects reclaim with
  `422`, and closes as `completed` on acceptance. It may still be **withdrawn** while pending,
  because nothing has changed hands yet. Accepting it takes the properties off the transferer.
- The two levers belong to different people and different moments: **decline** is the recipient's
  answer to a pending transfer, **withdraw** is the transferer taking a pending transfer back, and
  **reclaim** is the transferer ending one already in force. Read `can_withdraw` and `can_reclaim`
  rather than deriving either from `status`.

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
| `new_facilities` | int[] | yes | — | Facility ids, at least one. Must belong to this company, be held by the caller, and not be in another of the caller's open transfers. |
| `new_roles` | int[] | no | `[]` | Role ids to hold on those facilities. Must belong to this company and be held by the caller (company-wide, or named on one of the offered properties). |
| `is_permanent` | bool | no | `false` | A permanent transfer has no expiry, cannot be reclaimed, and completes on acceptance. |
| `timestamp_to` | datetime | conditional | `null` | Required when `is_permanent` is false, must be in the future. Prohibited when it is true. |
| `include_pending_tasks` | bool | no | `false` | Hands the transferer's work on the transferred properties over instead of sharing it. See [What accept hands over](#what-accept-hands-over). Always `true` on a permanent transfer, whatever is sent: show it ticked and locked when "permanent" is chosen. |

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

Only the named recipient. Returns the transfer with `items` and `handovers` loaded, `accepted.raw`
set, and status `active` (temporary) or `completed` (permanent). The transferer is notified.

What acceptance grants depends on permanence and on each role's `enforce_on_facility`:

| Transfer | Role enforced | Effect |
| --- | --- | --- |
| Permanent | yes | The existing `facility_role` holder is replaced by the recipient. |
| Permanent | no | A parallel role row is created, and the recipient is assigned the role company-wide if the transferer holds it. |
| Temporary | either | A parallel role row is created, removed again on return. |

Property access is **additive for the recipient**: nobody loses sight of a property because
somebody else was given sight of it. The one exception is the transferer of a **permanent**
transfer, who handed the properties over and loses their own allocation to them. A role row held
by someone other than the transferer is never displaced; that case falls back to an additive grant.

Answering a transfer that is no longer `pending` returns `422`.

## What accept hands over

Accept runs in one transaction, in this order. Every change beyond access is recorded in the
transfer's `handovers` ledger, which is what reclaim and auto-return restore from.

1. **Access and roles** (`items`), as in the table above.
2. **Permanent only**
   - Each offered company-wide role (not `enforce_on_facility`) the transferer holds is assigned to
     the recipient (`model_has_roles`). Ledger `role_assigned`. Without it, a step for that role on
     the property would reach nobody once the transfer is completed and the transferer is gone.
   - The transferer's own allocation (`facility_user`) to each transferred property is removed.
     Ledger `facility_user_removed`. A company owner has no rows to remove.
   - Neither is ever reversed: a permanent transfer is never reclaimed or returned.
3. **Documents in flight become the recipient's.** The transferer's documents on the transferred
   properties that are still in an approval chain (invoices, credit notes, receipts, LOOs and every
   other approvable), and their procurement requests not yet at LPO, completed or rejected, switch creator
   (`created_by_id`; `generated_by_id` on a LOO) to the recipient, so the recipient can edit and
   resubmit them. If such a document was returned for changes and the transferer holds its
   "Changes requested" task, that task moves to the recipient too, whatever
   `include_pending_tasks` says: it is the creator's task. Ledger `creator` (and `pending_task`).
4. **Open steps.** Every open, current approval step and every current procurement step (pending,
   waiting for quotes, quotes review) on a transferred property, for a transferred role, that the
   transferer is an actor on:
   - `include_pending_tasks: true`, or a permanent transfer: the transferer is **replaced** by the
     recipient;
   - `include_pending_tasks: false` on a temporary transfer: the recipient is **added alongside**
     the transferer.

   The step's pending tasks are re-raised to match its new actors, so the recipient can act on it
   at once. Review steps (a document returned to its creator) are never re-addressed; they follow
   the creator in point 3. Ledger `approval_step_actors` / `procurement_step_actors`.
5. **Direct pending tasks** (`include_pending_tasks: true`, which a permanent transfer always is): tasks of the transferer tied to a
   transferred property that are not already covered by points 3-4 (not a step's task, not a
   procurement request's) move to the recipient. Invoices, credit notes and exit notices are tied
   through their lease's property. Ledger `pending_task`.
6. **Status** becomes `active` (temporary) or `completed` (permanent).
7. **Lease-application tasks** (review and offer tasks raised to the responsible property roles)
   are re-raised for the transferred properties once the status is saved, so they land with
   whoever the roles now answer with (the recipient, or both, on the rule in point 4). They are
   not in the ledger; reversal re-raises them the same way.

**While a temporary transfer is in force**, steps raised later on a transferred property for a
transferred role (a new document, the next step of a chain, a resubmission) go to the recipient
automatically, replaced or added on the same rule as point 4. Transfers chain: if A transferred to
B and B to C, a step that would have gone to A reaches C. A recipient who is no longer an active
member of the company is skipped; if that leaves nobody, the step goes where it would have gone
without the transfer. Lease application review and LOO tasks follow the same rule. The nightly
re-addressing jobs agree with this, so they never hand the work back. A `completed` permanent
transfer is no longer read here: by then the recipient holds the roles and access outright.

**Reclaim and auto-return** restore from the ledger, newest first:

- a step's actors are put back only if the step is still open and current and nobody has
  re-addressed it since; otherwise the record is left as it is;
- a document's creator is put back only if it is still the recipient;
- moved tasks go back to the transferer; a task the recipient held before the transfer stays;
- then the access grants are revoked as before, and steps raised during the transfer that still
  name the recipient are re-addressed to whoever holds the role on the property now.

## Decline

`PATCH /api/v1/app/{company}/access-management/responsibility-transfers/{transfer}/decline`

Only the named recipient. Marks the transfer `declined` and notifies the transferer. No property,
role or pending task is touched, and the properties are free for another transfer.

## Withdraw

`PATCH /api/v1/app/{company}/access-management/responsibility-transfers/{transfer}/withdraw`

Only the transferer, and only while the transfer is still `pending`. Marks it `withdrawn`, stamps
`withdrawn_at` and notifies the recipient — who had it waiting in `mine/incoming`, and would
otherwise watch it disappear without explanation.

Nothing is granted while a transfer is pending, so nothing is revoked here: no property, no role
and no pending task is touched, and the properties are free for another transfer. It is the mirror
of decline, not of reclaim.

- Already accepted, declined, expired or withdrawn: `422` on `status`. Use `can_withdraw` to hide
  the button rather than letting the user discover this.
- Someone other than the transferer: `403`. The recipient's lever is decline.
- A **permanent** transfer may be withdrawn, unlike reclaimed. The rule that a real handover
  cannot be undone applies to a handover that happened; while it is pending there is nothing to
  undo, and refusing would leave a permanent transfer sent to the wrong person endable only by
  that person.

## Reclaim

`PATCH /api/v1/app/{company}/access-management/responsibility-transfers/{transfer}/reclaim`

Only the transferer, and only on a temporary transfer that is currently in force.

- Permanent transfer: `422`. Use `can_reclaim` on the resource to hide the button rather than
  letting the user discover this.
- Someone other than the transferer: `403`.

Reversal is immediate and does not wait for work in flight. It revokes exactly what the transfer
granted and nothing else, restores the original holder of any reassigned enforced role, and puts
back what the handover ledger recorded (see [What accept hands over](#what-accept-hands-over)).

## My transfers

`GET /responsibility-transfers/mine/outgoing` returns `data.responsibility_transfers`: an array of
the caller's open outgoing transfers, newest first, possibly empty. This backs the floating
left-nav indicators (one per transfer), their countdowns and the reclaim popup.

```json
{
  "data": {
    "responsibility_transfers": [
      { "id": 14, "status": { "value": "pending", "color": "warning" }, "...": "..." },
      { "id": 12, "status": { "value": "active", "color": "success" }, "...": "..." }
    ]
  }
}
```

`GET /responsibility-transfers/mine/incoming` returns `data.responsibility_transfers`: an array,
possibly several. This backs the accept/decline prompt.

Both include `pending` and in-force (`accepted`, `active`) transfers, and both are scoped to the
route `{company}`: a transfer sent or received in another company appears when that company is in
the URL, not here. A `completed` permanent transfer is in neither list: it drops out the moment it
is accepted. The overlap rule is counted across every company the caller belongs to, but a
property belongs to one company, so the route company's `mine/outgoing` is all the picker needs.

`GET /api/v1/app/{company}/inbox/summary` carries the same two reads as `outgoing_transfers` (an
array) and `incoming_transfers` (a count) for polling. See
`docs/frontend/api/v1/app/inbox-and-permissions.md`.

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
  "offered_roles": [{ "id": 3, "name": "Procurement Manager" }],
  "offered_facilities": [
    { "id": 7, "name": "Kahawa Side Tower" },
    { "id": 9, "name": "Garden Court" }
  ],
  "is_permanent": false,
  "include_pending_tasks": true,
  "status": { "value": "active", "color": "success" },
  "can_reclaim": true,
  "can_withdraw": false,
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
  "withdrawn": { "raw": null, "formatted": null, "diff": null },
  "returned": { "raw": null, "formatted": null, "diff": null },
  "created": { "raw": "...", "formatted": "...", "diff": "..." },
  "updated": { "raw": "...", "formatted": "...", "diff": "..." },
  "items": [],
  "handovers": []
}
```

`offered_facilities` and `offered_roles` are `new_facilities` and `new_roles` with their names, in the
same order, on every read (list, `mine/*`, show and the action responses). Show these names on the
request: the recipient holds none of the offered properties until they accept, so they cannot look
the names up from the property list. A property deleted since keeps its name; a record that no longer
exists at all has `name: null`.

`items` is present on `show`, `accept`, `reclaim` and `withdraw`; `handovers` (with each
`subject`) on `show`, `accept` and `reclaim`, where the reversed rows show what came back. Neither is on the list or `mine/*` reads.

### Driving the countdown

Tick against `timestamp_to.iso`, and seed the timer from `seconds_remaining` rather than from the
viewer's own clock. That value is the server's view of the remaining time, so a machine with a
skewed clock cannot show a timer that disagrees with the auto-return that is actually going to
fire. It never goes below `0`. On a permanent transfer every field in the object is `null`, which
is the signal to render no countdown at all.

Auto-return runs hourly, so a transfer can sit a few minutes past `has_expired: true` before its
status changes. Treat `has_expired` as "about to return", not as "already returned".

### Transfer items

Each entry records one access grant the transfer made, which is how the reversal knows what to
undo:

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

### Handovers

Each entry records one thing accept moved across beyond access (see
[What accept hands over](#what-accept-hands-over)). `previous_values` is what it was and
`new_values` what it became; their keys depend on `kind`:

| `kind.value` | `subject_type` | `previous_values` → `new_values` | Reversed |
| --- | --- | --- | --- |
| `creator` | The document (e.g. `FacilityInvoice`, `Loo`, `FacilityProcurementRequest`) | `{ "column": "created_by_id", "user_id": 8 }` → `{ "column": "created_by_id", "user_id": 42 }` | Yes, if still the recipient |
| `approval_step_actors` | `ApprovalStep` | `{ "actors": [8, 15] }` → `{ "actors": [42, 15] }` | Yes, if nobody re-addressed the step |
| `procurement_step_actors` | `FacilityProcurementRequestStep` | `{ "actors_ids": [8] }` → `{ "actors_ids": [42] }` | Yes, on the same terms |
| `pending_task` | The task's subject | `{ "user_id": 8, "title": "...", "approval_step_id": null, "pending_task_id": 301 }` → the same keys for the recipient's task | Yes |
| `facility_user_removed` | `Facility` | `{ "user_id": 8 }` → `null` | Never (permanent only) |
| `role_assigned` | `Role` | `null` → `{ "user_id": 42 }` | Never (permanent only) |

`kind` colours: `creator` `primary`, the two step kinds `info`, `pending_task` `secondary`,
`facility_user_removed` `danger`, `role_assigned` `success`.

`subject_type` is the class name without its namespace. `subject` is `{ id, name }` when the record
still exists, and `null` once it is gone. `name` is the first non-empty of the record's `name`,
`title`, `reference`, `number`, `invoice_number` or `code`, and `null` when it has none of them
(an approval step, for one), so fall back to `subject_type` and `subject_id` for the label.
`is_reversed` / `reversed` mirror the item fields: `reversed.formatted` carries the time
(`d M, Y H:i`), `created.formatted` the date only.

```json
{
  "id": 7,
  "responsibility_transfer_id": 12,
  "kind": { "value": "approval_step_actors", "color": "info" },
  "subject_type": "ApprovalStep",
  "subject_id": 911,
  "subject": { "id": 911, "name": null },
  "previous_values": { "actors": [8, 15] },
  "new_values": { "actors": [42, 15] },
  "is_reversed": false,
  "reversed": { "raw": null, "formatted": null, "diff": null },
  "created": { "raw": "...", "formatted": "...", "diff": "..." }
}
```

## Errors

| Code | When |
| --- | --- |
| `403` | Wrong person for the action: not the recipient on accept/decline, not the transferer on reclaim/withdraw, not a participant on `show`. |
| `404` | The transfer does not belong to route `{company}`. |
| `422` | Validation, plus: `new_facilities` overlapping one of the caller's open transfers, a property the caller is not allocated to, a role the caller does not hold (`new_roles`), a property or role the caller holds only through a temporary transfer (`new_facilities` / `new_roles`), a recipient outside the company or department (`to_user_id`), reclaiming a permanent transfer, withdrawing one that is no longer pending, or answering a transfer that is no longer pending (`status`). |

Apply shared auth, tenancy, query, and error handling guidance from:

- `docs/frontend/api/v1/app/README.md`

## Deploy

Run `php artisan migrate`: it creates the handover ledger and closes permanent transfers accepted
before this change (still `active` or `accepted`) as `completed`. Nothing else is needed.
