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
- `review`: marks current step as review, sends flow back to previous step, and resets previous step to pending.
- `reject`: marks the current step as rejected and **terminates the entire workflow** — every remaining pending step for the approvable is also marked rejected (so no later step can be actioned), and the approvable is marked with the model's `FINAL_STATUS_ON_REJECTION`.

The applied statuses come from constants declared on each approvable model (`INITIAL_STATUS_ON_CREATE`, `FINAL_STATUS_ON_APPROVAL`, `FINAL_STATUS_ON_REJECTION`), not from the approval template.

Authorization:

- user must be allowed on the current pending step only
- user must hold the step role
- if `actors` is populated, user must be in `actors`

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

## Attempts and re-submission

Rejecting terminates the chain and sends the resource to its rejection status.
Where that status is still editable, the resource can be corrected and submitted
again; that opens a **new attempt** rather than reopening the rejected one.

The `approval_steps` array on a resource always shows the **current** attempt
only. Earlier attempts remain in the database as the record of what happened, but
they are not part of the chain anybody is being asked to act on.
