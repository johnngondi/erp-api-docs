# Settings API

Domain: `Property Management`

Base prefix:

`/api/v1/app/{company}/property-management/settings`


## Procurement Settings

### General

UI placement:

- `SettingsPage > Group Tab (Procurement) > General Tab`

Endpoints:

- `GET /general`
- `PATCH /general/{setting}`

How to fetch procurement general settings:

- Use `filter[group]=procurement`
- Example: `GET /general?filter[group]=procurement`

List query support:

- Filters:
  - `filter[key]`, `filter[name]`, `filter[description]`, `filter[value]`, `filter[group]`
- Sort:
  - `sort=id,key,name,created_at,updated_at`
- Pagination:
  - `per_page` (defaults to `config('app.query.default_per_page')`)

Frontend metadata fields returned by `SettingResource`:

- `type`: frontend input type hint (examples: `select`, `number`, `date`, `text`, )
- `options`: input options source or static options
- `dependency_key`: setting key this field depends on
- `dependency_value`: value required on `dependency_key` for this setting to be active/visible

Frontend behavior for `options`:

- If `options` is `null`, no selectable options are required.
- If `options` is a key-value object (example: `{ "yes": "Yes", "no": "No" }`), render those as static select options.
- If `options` is an array with exactly one element and that element is a link/URI, treat it as a remote options endpoint.
- For remote options endpoint behavior:
  - Send an authenticated request to that URI using the currently authenticated user's API key/token (same `Authorization` context as the logged-in user).
  - You may add a new endpoint inside src\api\endpoints\portals\app\property-management\settings.api.js which will be receiving the endpoint and then transforming the response to proper data consumable by options renderer use searchable select for all options
  - Use the returned payload to populate selectable options in the UI.
  - Do not call the URI anonymously.

Dependency behavior:

- If both `dependency_key` and `dependency_value` are set, only show that setting when the setting identified by `dependency_key` currently equals `dependency_value`.
- If both are `null`, the setting is always available.

Update payload (`PATCH /general/{setting}`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `value` | Yes (present) | scalar or null | Only this field is updated |

Seeded Procurement General settings:

| Key | Default Value | Type | Options | Dependency | Description |
|---|---|---|---|---|---|
| `rfq_default_deadline_days` | `3` | `number` | `null` | none | Number of days added to RFQ issue date to compute default deadline. |
| `emergency_rfq_auto_send_enabled` | `no` | `select` | static: `yes/no` | none | Whether emergency RFQs should be auto-sent when default vendor does not respond. |
| `emergency_rfq_send_after_hours` | `24` | `number` | `null` | `emergency_rfq_auto_send_enabled=yes` | Hours to wait before sending emergency RFQs when default vendor has not submitted a bid. |
| `rfq_prioritize_default_vendor` | `yes` | `select` | static: `yes/no` | none | Whether default vendor gets first chance to bid before RFQ is sent to other vendors. |
| `rfq_default_vendor_exclusive_window_hours` | `24` | `number` | `null` | `rfq_prioritize_default_vendor=yes` | Hours granted to default vendor before RFQ goes to other vendors. |
| `rfq_default_vendor_auto_accept_max_amount` | `50000` | `number` | `null` | `rfq_prioritize_default_vendor=yes` | Maximum default-vendor bid amount that can be accepted directly. |

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

### Property/Facility Types

UI placement:

- `SettingsPage > Lease Management > Property/Facility Types`

Endpoints:

- `GET /api/v1/app/{company}/property-management/facility-types`
- `POST /api/v1/app/{company}/property-management/facility-types`
- `GET /api/v1/app/{company}/property-management/facility-types/{facility_type}`
- `PUT/PATCH /api/v1/app/{company}/property-management/facility-types/{facility_type}`
- `DELETE /api/v1/app/{company}/property-management/facility-types/{facility_type}`

List query support:

- Filters:
  - `filter[id]`, `filter[title]`, `filter[has_tax]`, `filter[division_type]`, `filter[created_at]`
- Sort:
  - `sort=id,title,has_tax,division_type,created_at`
- Pagination:
  - `per_page` (defaults to `config('app.query.default_per_page')`)

Create/Update payload (`FacilityTypeData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `title` | Yes | string | Facility type display name |
| `has_tax` | No | boolean | Defaults to `false` |
| `division_type` | No | string | `size`, `unit`, `both` (defaults to `both`) |

Response item shape (`FacilityTypeResource`):

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Record id |
| `name` | string | Mirrors model field `title` |
| `has_tax` | boolean | Tax applicability flag |
| `division_type` | string | One of `size`, `unit`, `both` |
| `created_at` | datetime string | `Y-m-d H:i:s` |
| `updated_at` | datetime string | `Y-m-d H:i:s` |

Authorization:

- `GET`: `view-facility-type`
- `POST`: `create-facility-type`
- `PUT/PATCH`: `update-facility-type`
- `DELETE`: `delete-facility-type`


### Lease Components
- `GET|POST /settings/lease-management/lease-components`
- `GET|PUT|PATCH|DELETE /settings/lease-management/lease-components/{leaseComponent}`
- `PATCH /settings/lease-management/lease-components/{leaseComponent}/activate`
- `PATCH /settings/lease-management/lease-components/{leaseComponent}/deactivate`

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

### General Settings

UI placement:

- `SettingsPage > Group Tab (e.g. Finance) > General Tab`

Endpoints:

- `GET /general`
- `PATCH /general/{setting}`

Notes:

- `GET /general` is always scoped to Property Management settings only.
- Backend applies a default module constraint of `module contains "pm"`.
- `module` is not exposed in the API response.
- This section is intended to grow over time as more PM general settings are added.

List query support:

- Filters:
  - `filter[key]`, `filter[name]`, `filter[description]`, `filter[value]`
  - `filter[group]` (single or CSV, e.g. `filter[group]=finance,tax`)
- Sort:
  - `sort=id,key,name,created_at,updated_at`
- Pagination:
  - `per_page` (defaults to `config('app.query.default_per_page')`)

Examples:

- `GET /general`
- `GET /general?filter[group]=finance`
- `GET /general?filter[group]=finance,tax&sort=-updated_at`

`GET /general` response item shape:

| Field | Type | Notes |
|---|---|---|
| `id` | integer | Setting id |
| `key` | string | Setting key |
| `name` | string|null | Display name |
| `description` | string|null | Help text |
| `value` | string|null | Current value |
| `group` | array|string|null | Group used for SettingsPage grouping |
| `type` | string|null | Frontend input type hint (`select`, `number`, etc.) |
| `options` | array|object|string|null | Static options or remote options URI container |
| `dependency_key` | string|null | Key of controlling setting |
| `dependency_value` | string|null | Required controlling value to show this setting |
| `created_at` | object | `{ raw, formatted, diff }` |
| `updated_at` | object | `{ raw, formatted, diff }` |

`options` handling rule (applies to all general settings groups):

- When `options` is an array containing only one URI/link value, the frontend must call that URI to fetch option values.
- That request must use the authenticated user's API key/token (`Authorization` header), not an anonymous request.

Update payload (`PATCH /general/{setting}`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `value` | Yes (present) | scalar or null | Only this field is updated |

Update response:

- Wrapped in `DataResource`:
  - `message`: `Setting updated successfully.`
  - `setting`: updated setting resource item

PM seeded general settings starter catalog:

| Key | Group | Type | Options | Dependency | Description |
|---|---|---|---|---|---|
| `default_tax` | `finance`, `tax` | `select` | remote URI array (taxes endpoint) | none | Default tax for PM financial transactions |
| `default_management_fee_expense_type` | `finance` | `select` | remote URI array (facility expense types endpoint) | none | Default expense type for management fees |
| `default_letting_fee_expense_type` | `finance` | `select` | remote URI array (facility expense types endpoint) | none | Default expense type for letting/reletting fees |

<!-- ### Bank Accounts

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
| `alias` | No | string | Optional | -->

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
