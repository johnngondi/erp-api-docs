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

- `approve`: marks current step as approved and moves to next pending step; on last step, marks approvable as template `final_status`.
- `review`: marks current step as review, sends flow back to previous step, and resets previous step to pending.
- `reject`: marks current step as rejected and ends the approval flow without moving approvable to final status.

Authorization:

- user must be allowed on the current pending step only
- user must hold the step role
- if `actors` is populated, user must be in `actors`
