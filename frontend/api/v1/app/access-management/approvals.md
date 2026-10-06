# Access Management: Approval Steps

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

Implemented:

- `GET /approval-steps/{approvalStep}`
- `POST /approval-steps/{approvalStep}`

## Path Parameters

| Parameter | Type | Required | Notes |
| --- | --- | --- | --- |
| `approvalStep` | int | Yes | Approval step ID for the target approvable resource |

Example:

`/approval-steps/45`

## Show Step

`GET /api/v1/app/{company}/access-management/approval-steps/{approvalStep}`

Response includes:

- selected step details
- approvable metadata
- full ordered `approval_steps` timeline for the same approvable
- per-step `is_last` boolean to indicate final step
- per-step `can_act` boolean for current authenticated user

### Step Status Values

- `pending`
- `approved`
- `review`
- `rejected`

## Act On Step

`POST /api/v1/app/{company}/access-management/approval-steps/{approvalStep}`

Create payload table:

| Field | Type | Required | Default | Notes |
| --- | --- | --- | --- | --- |
| `status` | string | Yes | - | One of: `approve`, `review`, `reject` |
| `comment` | string \| null | Conditional | `null` | Required when `status` is `review` or `reject` |

Behavior by `status`:

- `approve`: marks current step as approved and moves to next pending step; on the last step, marks the approvable with the model's `FINAL_STATUS_ON_APPROVAL` and fires the template's post-approval event.
- `review`: marks the current step as `review` with the comment and sends the document back. When an earlier step exists, that step is reset to `pending` and re-prompted. On the **first** step of the chain there is no earlier step, so the document is **returned to its submitter** — see [Returned for changes](#returned-for-changes).
- `reject`: marks the current step as rejected and **terminates the entire workflow** — every remaining pending step for the approvable is also marked rejected (so no later step can be actioned), and the approvable is marked with the model's `FINAL_STATUS_ON_REJECTION`, which is `rejected` on every approvable.

`rejected` is terminal. A rejected resource can be viewed and deleted, and nothing else: it
cannot be edited, resubmitted, cancelled, signed, paid, processed, closed or moved to any other
status. Every `permissions` flag on it except `view` and `delete` is `false`, and the endpoints
behind them answer `403`. Delete removes it outright — it was never posted, so there is nothing
to reverse. Statuses such as `cancelled` or `inactive` now only ever mean somebody did that by
hand.

A rejected resource may also run cleanup of its own, beyond the status. The status move is
written quietly, so this is not something an observer can carry — a resource that needs it says
so, and it runs inside the same transaction as the rejection. Today one resource does:
a rejected **remittance** cancels the management fee it raised and unlinks the receipts and
expenses it had claimed, so the next remittance for that period can pick them up. See
[Remittances → What a rejection releases](../property-management/finance/remittance.md#what-a-rejection-releases).

The applied statuses come from constants declared on each approvable model (`INITIAL_STATUS_ON_CREATE`, `FINAL_STATUS_ON_APPROVAL`, `FINAL_STATUS_ON_REJECTION`), not from the approval template.

A step left in `review` is never skipped. When the step before it is approved again, the
`review` step is reset to `pending` and its actors are prompted again, rather than the chain
jumping past it to the next step or finishing.

Authorization:

- user must be allowed on the current pending step only
- user must hold the step role, or be named for it on the document's property
- user must be in the step's `actors`. A step whose `actors` is empty can be acted on by nobody
  (this used to mean "anyone holding the role")

## Who a step is addressed to

Every approvable belongs to a property: invoices, credit notes and exit notices through their
lease; leases, LOOs, receipts, remittances, contracts and LPOs directly. When a step is put in
front of its people, its `actors` are worked out against that property. Each actor gets a pending
task.

| Step role | `actors` |
| --- | --- |
| Assigned per property (`enforce_on_facility`) | The user named for that role on the property (the property's `roles`). If nobody is named, every holder of the role who is allocated to the property |
| Company-wide | Every holder of the role who is allocated to the property |

Only active members of the company (or its owner) count. Being named on the property is the
assignment, so the named user need not also hold the role. An allocated holder must hold it.

If nobody qualifies, the step is refused with a **422** keyed on `steps`:

```json
{
  "message": "No one is assigned to Senior Property Manager on Absa Towers, so this step cannot be actioned. Assign a holder, or allocate a member of that role to the property.",
  "errors": { "steps": ["No one is assigned to Senior Property Manager on Absa Towers, ..."] }
}
```

This comes back from whatever moved the chain onto that step. That is the create (or submit) of
the document for the first step, and the `approve`/`review` call on this endpoint for the steps
after it. Nothing is saved: the document is not created, or the step stays where it was. The fix
is to name a holder for the role on the property, or allocate a holder of the role to it.

A document with no property (none of today's approvables) falls back to every active holder of
the role in the company.

## Step fields added by the edit grant

Every step in the `approval_steps` array now also carries:

| Field | Type | Meaning |
| --- | --- | --- |
| `attempt` | int | Which run of the chain this step belongs to. See [Attempts](./approval-templates.md#attempts) |
| `allowed_to_edit` | boolean | Whether this step lends its actor editing rights at all |
| `editable_fields` | array[string] \| null | Which fields that grant covers; `null` means all of them |
| `can_edit` | boolean \| null | Whether **you**, the current viewer, may edit the resource right now |

`allowed_to_edit` describes the step; `can_edit` describes you. A step may lend
the right out while you are not the one it is waiting on — then `allowed_to_edit`
is `true` and `can_edit` is `false`, exactly as `can_act` behaves.

These are copied onto the step when the chain is initiated, not read live off the
template. Editing a template does not change a chain already in flight.

When `can_edit` is `true`, the resource's normal update endpoint accepts your
changes even without its update permission. Fields outside `editable_fields` are
refused with **403** naming them, and the request is rejected whole — nothing is
half-saved.

## Returned for changes

A `review` on the first step returns the document to the people who raised it: its creator
(`created_by`) and the person who submitted it for approval (`initiated_by` on the steps). The
document keeps its pending status and nothing is posted.

While it is returned:

- the step stays `review`, with the reviewer in `acted_by` and the request in `comment`;
- the step carries `is_returned: true` and `is_current: true`, and `can_act` is `false` for
  everybody — there is no pending step to act on;
- each person in `returned_to` gets a pending task titled "Changes requested: {resource}" with the
  reviewer's comment, and a notification (in-app only);
- those people may edit the document through its normal update endpoint even without its update
  permission, so the resource's `permissions.update` is `true` for them;
- the chain still counts as open, so the document cannot be submitted for approval a second time.

**Resubmission is implicit.** The first successful update of the document through its update
endpoint — by a returned-to user or anybody else allowed to edit it — sends the step back to
`pending`, addressed to **the reviewer alone**: the step's `actors` becomes just the reviewer,
who gets the pending task back ("Resubmitted after your review: {comment}"), and the
submitters' tasks are removed. If the reviewer can no longer act on the step (they lost the role,
or are no longer named for it on the property or allocated to it), the step is addressed afresh
as described in [Who a step is addressed to](#who-a-step-is-addressed-to). The chain carries on
from there in the same attempt.

Step fields for this:

| Field | Type | Meaning |
| --- | --- | --- |
| `is_returned` | boolean | The step sent the document back to its submitter and is waiting on an edit |
| `returned_to` | array[`{id, name}`] | Who may edit and resubmit it; empty when the step is not returned |
| `initiated_by` | object \| null | The user who submitted the document for approval |

## Attempts and re-submission

Rejecting terminates the chain and sends the resource to `rejected`, which is
final: no resource can be resubmitted once rejected, and the corrected document is
raised as a new record. Attempts remain on every step so the history of a chain is
never rewritten; a chain that is never rejected only ever has attempt `1`.

The `approval_steps` array on a resource always shows the **current** attempt
only. Earlier attempts remain in the database as the record of what happened, but
they are not part of the chain anybody is being asked to act on.

### Show responses only

`approval_steps` is emitted on the record a **show** response is about (and on the
record returned by create/update/action endpoints). List rows and records nested inside
another resource leave it out, because building the chain costs several queries per
record. Pass `?with_approval_steps=1` on a list to get it back for every row.

