# Procurement API

Domain: `Property Management`

Base prefix:

`/api/v1/app/{company}/property-management/procurement`

## Procurement Requests

Endpoints:

- `GET /requests`
- `POST /requests`
- `GET /requests/{procurementRequest}`
- `PUT/PATCH /requests/{procurementRequest}`
- `DELETE /requests/{procurementRequest}`
- `GET /requests/{procurementRequest}/items`
- `POST /requests/{procurementRequest}/items`
- `GET /requests/{procurementRequest}/items/{item}`
- `PUT/PATCH /requests/{procurementRequest}/items/{item}`
- `DELETE /requests/{procurementRequest}/items/{item}`
- `GET /requests/{procurementRequest}/steps/{step}`
- `PUT/PATCH /requests/{procurementRequest}/steps/{step}`

### List query support

`GET /api/v1/app/{company}/property-management/procurement/requests`

- Filters:
  - `filter[id]`, `filter[title]`, `filter[ticket_id]`, `filter[facility_id]`, `filter[asset_id]`
  - `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[expense_sub_type_id]`, `filter[created_at]`, `filter[priority]`, `filter[type]`
- Sort:
  - `sort=id,title,type,created_at,priority`
- Include: not supported
- Fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/procurement/requests?filter[type]=work&sort=-created_at`

### Create/Update payload

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `title` | Yes | string | - |
| `description` | Yes | string | - |
| `type` | Yes | string | `purchase`, `work` |
| `expense_type_id` | Yes | integer | Must exist in `expense_types.id` |
| `expense_category_id` | No | integer | Must exist in `expense_categories.id`; when omitted, backend derives from `expense_type_id` |
| `expense_sub_type_id` | Yes | integer | Must belong to selected expense type |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `ticket_id` | No | integer | Must exist in `tickets.id` |
| `asset_id` | No | integer | Must exist in `assets.id` |
| `owner_id` | No | integer | Must exist in `users.id`, defaults to current user |
| `items` | Conditional | array | Required when `type=purchase` |

`items[]` object (for `type=purchase`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `purchase_item_id` | Yes | integer | Must exist in `purchase_items.id` |
| `quantity` | Yes | integer | Minimum `1` |

### Review step payload

`PUT/PATCH /requests/{procurementRequest}/steps/{step}`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `status` | Yes | string | `approved`, `review`, `rejected` |
| `comment` | Conditional | string | Required in many non-approve flows |
| `preferred_vendor_id` | Conditional | integer | Required if `has_preferred_vendor=true` |
| `selected_option` | Conditional | integer | Required for specific quote-step states |
| `expense_category_id` | Conditional | integer | Required if current step has `select_expense_category=true` and `status=approved` |
| `vendors` | Conditional | array | Required if `override=true` |
| `deadline` | No | string/date | Optional |
| `has_preferred_vendor` | No | boolean | Default `false` |
| `override` | No | boolean | Default `false` |

## LPOs

Endpoints currently implemented:

- `GET /lpos`
- `GET /lpos/{lpo}`
- `PUT /lpos/{lpo}/review`

Notes:

- Other LPO routes exist in route definitions but are not currently implemented in the controller.

List query support:

- Filters:
  - `filter[id]`, `filter[procurement_request_id]`, `filter[assigned_technician_id]`
  - `filter[expense_category_id]`, `filter[created_at]`, `filter[delivery_at]`, `filter[delivered_at]`, `filter[rating]`
- Sort:
  - `sort=id,created_at,amount,delivery_at`
- Include:
  - `include=items`
- Fields: not supported

### Review submitted LPO document

`PUT /api/v1/app/{company}/property-management/procurement/lpos/{lpo}/review`

Purpose:

- Used by App users to approve or reject a vendor-submitted LPO delivery document.

Request body (intended contract):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `status` | Yes | string | `approved`, `rejected` |
| `comments` | Conditional | string | Recommended and typically required when `status=rejected` |

Suggested responses:

- Approve success:
  - `message`: `LPO document approved successfully.`
  - `lpo.status.value`: expected transition to delivered/approved workflow state
- Reject success:
  - `message`: `LPO document rejected successfully.`
  - `lpo.status.value`: expected transition to review/rejected workflow state

Current backend state:

- Route exists, but the handler is currently a placeholder in `routes/Api/v1/app/property-management.php`.
- Final payload validation and exact response structure should be confirmed once controller/action is implemented.

## Contracts

Endpoints:

- `GET /contracts`
- `POST /contracts`
- `GET /contracts/{contract}`
- `PUT/PATCH /contracts/{contract}`
- `DELETE /contracts/{contract}`
- `PATCH /contracts/{contract}/status`
- `POST /contracts/{contract}/generate-bill-for-next-period`

List query support:

- Filters:
  - `filter[id]`, `filter[title]`, `filter[status]`, `filter[start_at]`, `filter[end_at]`, `filter[created_at]`
  - `filter[facility_id]`, `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[expense_sub_type_id]`, `filter[asset_id]`
- Sort:
  - `sort=id,title,status,start_at,end_at,created_at`
- Include:
  - `include=bills`

Create/Update payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `vendor_id` | Yes | integer | Must exist in `users.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `start_at` | Yes | date (`YYYY-MM-DD`) | - |
| `end_at` | Yes | date (`YYYY-MM-DD`) | Inclusive last day of the contract. `period_in_days` is derived from `start_at`/`end_at` by the backend and stored automatically |
| `type` | Yes | string | `fixed`, `variable` |
| `amount` | Yes | integer | - |
| `expense_type_id` | Yes | integer | Must exist in `facility_expense_types.id` |
| `expense_category_id` | Yes | integer | Must exist in `expense_categories.id` |
| `title` | No | string | Optional |
| `notes` | No | string | Optional |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `expense_sub_type_id` | No | integer | Must exist in `expense_sub_types.id` |
| `asset_id` | No | integer | Must exist in `assets.id` |
| `agreement_upload_id` | No | integer | Must exist in `uploads.id` |
| `has_tax` | No | boolean | Default `false` |
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `creator_id` | No | integer | Must exist in `users.id` |
| `approver_id` | No | integer | Must exist in `users.id` |

`status` is **no longer accepted** on create or update. A contract's status belongs to its
approval chain - see [Approval](#approval) - and the only moves left in a person's hands are
between `active` and `suspended`, through the status endpoint below.

(Previously `status` was accepted here and defaulted to `pending`, which meant an update that
left it out quietly sent a live contract back to pending. It is now ignored entirely.)

### Status update

`PATCH /api/v1/app/{company}/property-management/procurement/contracts/{contract}/status`

Authorized against `update-status-facility-contract` (it previously checked
`update-facility-contract`).

| Field | Required | Type | Allowed Values |
|---|---|---|---|
| `status` | Yes | string | `active`, `suspended` |

Allowed transitions - everything else returns **422**:

| From | To | |
|---|---|---|
| `active` | `suspended` | allowed |
| `suspended` | `active` | allowed |
| anything | `pending` | refused - the chain raises a contract into pending |
| anything | `active` | refused - approving the chain is what makes a contract active |
| anything | `inactive` | refused - rejecting the chain is what makes a contract inactive |
| anything | `expired` | refused - derived from `end_at`, never stored |

`expired` is never written to the database. An `active` contract whose `end_at` has passed
reads as `expired` in reports, computed from the date.

### Approval

`FacilityContract` is a registered approvable model (`config('approvals.models')`) and is
driven by the shared approval framework. Act on a contract's chain through
`POST /access-management/approval-steps/{approvalStep}` - see
[Approval Steps](../access-management/approvals.md). Configure the chain itself under
[Approval Templates](../access-management/approval-templates.md).

Only `POST /contracts` raises a chain. Contracts created by the system - the utility and
service contract sync that runs off a facility's setup, and the EPMAS importers - are in force
the moment they are written and never enter one.

Every contract response from the staff endpoints carries an `approval_steps` array in the same
shape as every other approvable resource. The **vendor**-facing contract endpoints do not: a
supplier does not see the company's internal approvers.

#### Condition facts

Fields a template step's `conditions` may be written against. Operators and the rule shape are
documented once in
[Approval Templates → Step conditions](../access-management/approval-templates.md#step-conditions).

| Fact | Meaning |
|---|---|
| `amount` | The contracted figure - the usual threshold to escalate on |
| `type` | `fixed` or `variable` |
| `billing_cycle` / `billing_period` | How often a fixed contract bills |
| `facility_id` | The property the contract is for |
| `vendor_id` | The supplier |
| `expense_type_id` / `expense_category_id` | What the spend is classified as |
| `currency_id` | The contract currency |
| `has_tax` / `tax_id` | Whether the contract is taxed, and at what rate |

#### Status transitions

| Action | The contract |
|---|---|
| Created, company has no active template | `active` immediately, no chain |
| Created, all steps fall away on conditions | `active` immediately, no chain |
| Created by a holder of a bypass role | `active` immediately, no chain |
| Created, a chain applies | `pending`, chain raised, first step prompted |
| `approve`, not the last step | stays `pending`, next step prompted |
| `approve`, the last step | `active`, `FacilityContractApprovedEvent` fires |
| `review` | stays `pending`, the previous step is reopened |
| `reject` | `inactive` |
| `PATCH {contract}/status` | `active` ↔ `suspended` only |

A `pending` contract is not yet in force: it does not bill, and
`POST {contract}/generate-bill-for-next-period` refuses it, as does the `contracts:generate-bills`
scheduler. The facility's expenditure budget follows the same line - a contract only counts
towards the contracted floor once its chain has made it `active`, and drops back out if it is
rejected.

### Generate bill for next period

`POST /api/v1/app/{company}/property-management/procurement/contracts/{contract}/generate-bill-for-next-period`

Purpose:

- Manually generate the upcoming period's bill for a contract ahead of the scheduled
  billing date (`next_billing_at`), when operationally necessary. Runs the same generation
  as the `contracts:generate-bills` scheduler.

Request body:

- None.

Eligibility (returns `422` validation error otherwise):

- Contract `type` must be `fixed` with `amount > 0`.
- Contract `status` must be `active`.
- Contract must have a billing schedule (`next_billing_at` set).
- Unlike the scheduler, the "not yet due" check is **skipped** — `next_billing_at` may be in the future.

Behavior:

- The bill `notes` describe the billing **period** being generated for (derived from `next_billing_at` and
  the billing cycle), not the date it was generated.
- `next_billing_at` is advanced by one billing cycle when the bill is created.

Success response:

- `message`: `Bill for next period generated successfully.`
- `bill`: the created `FacilityBill`.

## Inventories and Purchase Items

Inventories:

- Implemented endpoint:
  - `GET /inventories`
- Note:
  - Other inventory routes exist but are currently not implemented in the controller.

Inventory list query support:

- Filters:
  - `filter[facility_id]`, `filter[inventory_category_id]`, `filter[name]`
  - `filter[current_stock]`, `filter[storage_room_id]`, `filter[shelf_id]`

Purchase items:

- Implemented endpoints:
  - `GET /inventories/purchase-items`
  - `POST /inventories/purchase-items`
  - `GET /inventories/purchase-items/{purchase_item}`
  - `PUT/PATCH /inventories/purchase-items/{purchase_item}`
  - `DELETE /inventories/purchase-items/{purchase_item}`
  - `PUT/PATCH /inventories/purchase-items/{item}/prices/{price}`

Purchase item create payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `type` | Yes | string | `service`, `product` |
| `name` | Yes | string | Must be unique |
| `inventory_category_id` | Yes | integer | Must exist in `inventory_categories.id` |
| `stock_keeping_unit_id` | No | integer | Must exist in `stock_keeping_units.id` |
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `is_regulated` | No | boolean | Default `false` |
| `base_price` | No | number | Default `0` |
| `vendors` | No | array | Optional |

Purchase item update payload:

- Same shape and validation rules as create payload.
- Success response message: `Purchase Item updated successfully.`
- Error message (server error): `Error updating Purchase Item`

Purchase item delete response:

- Success response message: `Purchase Item deleted successfully.`
- Error message (server error): `Error deleting Purchase Item`

Purchase item price update payload:

| Field | Required | Type |
|---|---|---|
| `price` | Yes | integer |

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Render `4xx` messages and validation errors.
- Show generic fallback for `5xx`.
