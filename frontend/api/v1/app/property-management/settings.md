# Settings API

Domain: `Property Management`

Base prefix:

`/api/v1/app/{company}/property-management/settings`

## Procurement Settings

### Request Step Template Groups

Endpoints:

- `GET /procurement/request-steps-template-groups`
- `POST /procurement/request-steps-template-groups`
- `GET /procurement/request-steps-template-groups/{request_steps_template_group}`
- `PUT/PATCH /procurement/request-steps-template-groups/{request_steps_template_group}`
- `DELETE /procurement/request-steps-template-groups/{request_steps_template_group}`

List query support:

- Filters:
  - `filter[description]`, `filter[title]`, `filter[created_at]`, `filter[is_default]`, `filter[facilities]`
- Sort: not documented
- Include: not documented
- Fields: not documented

Create/Update payload (`ProcurementRequestStepTemplateGroupData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `title` | Yes | string | Must be unique |
| `description` | No | string | Optional |
| `is_default` | No | boolean | Default `false` |
| `facilities` | No | array | Optional facility bindings |

### Request Step Templates (inside a group)

Endpoints:

- `POST /procurement/request-steps-template-groups/{group}/templates`
- `GET /procurement/request-steps-template-groups/{group}/templates/{template}`
- `PUT/PATCH /procurement/request-steps-template-groups/{group}/templates/{template}`
- `DELETE /procurement/request-steps-template-groups/{group}/templates/{template}`

Notes:

- Nested template `index` route exists but is currently not implemented in the controller.

Create/Update payload (`ProcurementRequestStepTemplateData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `title` | Yes | string | Must be unique |
| `role_id` | Yes | integer | Role id used for step actor |
| `description` | No | string | Optional |
| `select_preferred_vendor` | No | boolean | Default `false` |
| `handle_procurement` | No | boolean | Default `false` |

## Lease Management Settings

### Lease Component Payment Priorities

Endpoints:

- `GET /lease-management/payment-priorities`
- `POST /lease-management/payment-priorities`
- `GET /lease-management/payment-priorities/{payment_priority}`
- `PUT/PATCH /lease-management/payment-priorities/{payment_priority}`
- `DELETE /lease-management/payment-priorities/{payment_priority}`

List query support:

- Filters:
  - `filter[lease_component_id]`, `filter[facility_id]`
- Sort:
  - `sort=lease_component_id,facility_id`

Create/Update payload (`LeaseComponentsPaymentPriorityData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `lease_component_id` | Yes | integer | Must exist in `lease_components.id` |
| `priority` | Yes | integer | Numeric order/weight |

## Finance Settings

### Bank Accounts

Endpoints:

- `GET /finance/banks/accounts`
- `POST /finance/banks/accounts`
- `GET /finance/banks/accounts/{account}`
- `PUT/PATCH /finance/banks/accounts/{account}`
- `DELETE /finance/banks/accounts/{account}`

List query support:

- Filters:
  - `filter[type]`, `filter[account_name]`, `filter[account_number]`, `filter[status]`
  - `filter[user_group_id]`, `filter[user_id]`, `filter[bank_branch_id]`
- Sort:
  - `sort=id,account_name,status`
- Include:
  - `include=userGroup,user,bankBranch`

Create/Update payload (`BankAccountData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `bank_branch_id` | Yes | integer | Must exist in `bank_branches.id` |
| `account_name` | Yes | string | - |
| `account_number` | Yes | string | - |
| `type` | No | string | `system`, `user` (defaults to `user`) |
| `status` | No | string | `active`, `inactive` (defaults to `active`) |
| `user_group_id` | No | integer | Must exist in `user_groups.id` |
| `user_id` | No | integer | Must exist in `users.id`; defaults to current user for `type=user` |
| `alias` | No | string | Optional |

### Expense Types and Sub-Types

Expense types endpoint implemented:

- `GET /finance/expense-types`

Sub-types endpoint implemented:

- `GET /finance/expense-types/{type}/sub-types`

Notes:

- Routes for creating/updating/deleting expense types and sub-types exist but are currently not implemented in these controllers.

Expense types list query support:

- Filters:
  - `filter[id]`, `filter[name]`, `filter[description]`, `filter[expense_category_id]`
  - `filter[is_procurable]`, `filter[can_have_default_vendor]`, `filter[can_have_preferred_vendor]`, `filter[created_at]`
- Include:
  - `include=expenseSubTypes`

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Render `4xx` messages and validation details.
- Show generic fallback for `5xx`.
