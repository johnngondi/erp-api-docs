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
| `expense_category_id` | No | integer | Must exist in `expense_categories.id`. The request's own category - any category may be chosen, whatever the expense type's default. When omitted, the expense type's default category is used |
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

Endpoints:

- `GET /lpos`
- `POST /lpos` — raise a **direct** LPO (see below)
- `GET /lpos/{lpo}`
- `PUT /lpos/{lpo}/review`

`PUT/PATCH /lpos/{lpo}` and `DELETE /lpos/{lpo}` are registered by the resource route but not implemented.

### Workflow LPOs and direct LPOs

An LPO comes into being in one of two ways:

| | Workflow LPO | Direct LPO |
|---|---|---|
| Raised by | The procurement request workflow, once a bid (work) or option (purchase) is selected | A staff user, by hand, through `POST /lpos` |
| `is_direct` | `false` | `true` |
| `procurement_request_id` / `procurementRequest` | Set | `null` |
| Items | Copied from the winning bid / selected option | Entered on the form |
| Winning bid document | On the request's bid | `winningBidUpload` |
| Comparable quotes | The request's other bids | `comparables` |
| Approval | Already approved through the request's steps; issued as `lpo` | Goes through the LPO approval template, if one is active (see below) |

Both kinds carry their own `title`, `facility`, `type`, `expenseType` and `expenseSubType`, so lists, filters,
property scoping, bills and the printed LPO treat them the same way.

### List query support

`GET /api/v1/app/{company}/property-management/procurement/lpos`

- Filters:
  - `filter[search]` — free text over id, title, notes, supplier, request title and property name
  - **Exact**: `filter[facility_id]`, `filter[is_direct]` (`1`/`0`), `filter[type]` (`work`, `purchase`)
  - `filter[vendor_id]`, `filter[expense_category_id]`, `filter[status]`, `filter[delivered_at]`, `filter[rating]`,
    `filter[amount]`, `filter[total]`, `filter[created_at]`
- Sort:
  - `sort=id,created_at,delivered_at,amount,total,rating,status`
- Include:
  - `include=items`
- Fields: not supported
- Pagination: `per_page`, `page`

Only LPOs on properties the user is allocated to are listed. `filter[facility_id]` reads the LPO's own
property, so direct LPOs are included.

Example:

`GET /api/v1/app/12/property-management/procurement/lpos?filter[is_direct]=1&filter[type]=work&sort=-created_at`

### Create a direct LPO

`POST /api/v1/app/{company}/property-management/procurement/lpos`

Authorization:

- Requires the `review-document-facility-procurement-lpo` permission (policy `create`).
- The user must be allocated to `facility_id`, otherwise `403`.

Request body (`CreateDirectLpoData`):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | Must exist in `facilities.id`. The property the order is for |
| `type` | Yes | string | `work`, `purchase` |
| `title` | No | string | Max 255. Shown on the supplier's jobcard task and the printed LPO |
| `vendor_id` | Yes | integer | Must exist in `users.id`. The supplier |
| `currency_id` | Yes | integer | Must exist in `currencies.id` |
| `expense_type_id` | Yes | integer | Must exist in `facility_expense_types.id` |
| `expense_sub_type_id` | Yes | integer | Must exist in `facility_expense_sub_types.id` and belong to `expense_type_id` |
| `expense_category_id` | No | integer | Must exist in `expense_categories.id`. The LPO's own category - any category may be chosen (e.g. a Repairs & Maintenance order charged to rent rather than service charge). When omitted, the expense type's default category is used |
| `delivery_at` | Yes | string/date | When the work or goods are due |
| `notes` | No | string | - |
| `assigned_technician_id` | No | integer | Must exist in `users.id` |
| `winning_bid_upload_id` | Yes | integer | Must exist in `uploads.id`. The supplier's winning quotation |
| `comparables` | No | array of integer | Each must exist in `uploads.id` and differ from `winning_bid_upload_id`. The other quotations the winning bid was compared against |
| `items` | Yes | array | At least one item |

`items[]` object — the same shape a supplier sends when submitting a bid:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `type` | Yes | string | `product`, `service` |
| `purchase_item_id` | Yes | integer **or** string | An existing `facility_purchase_items.id`, **or** the name of a new item (see below) |
| `notes` | No | string | - |
| `quantity` | Yes | number | Greater than `0` |
| `stock_keeping_unit_id` | Yes | integer | Must exist in `stock_keeping_units.id` |
| `cost` | Yes | number | Unit cost, minimum `0` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |
| `discount_amount` | No | number | Discount on the line, minimum `0`. Default `0` |

**Selecting or adding an item.** Send an integer `purchase_item_id` to pick an existing purchase item. Send a
string to add one: if a purchase item with exactly that name already exists it is reused, otherwise a new
purchase item is created (in the first inventory category, with the line's type, unit, tax and cost as its
base price, and the supplier linked to it). The line's `title` is the purchase item's name.

**Line arithmetic** (the same as bids):

- `amount = quantity × cost`
- `amount_after_discount = amount − discount_amount`
- `tax = rate × amount_after_discount` when the tax is discount-deductible, otherwise `rate × amount`
- `total = amount_after_discount + tax`

If the supplier is **not VAT-registered**, every line is stored with the zero-rate tax and `tax = 0`,
exactly as a workflow LPO is. The LPO's `amount`, `discount_amount`, `amount_after_discount`, `tax` and
`total` are the sums of its lines.

**Uploads.**

- `winning_bid_upload_id` is stored on the LPO and returned as `winningBidUpload`.
- `comparables` are attached to the LPO and returned as `comparables`. An upload belongs to one record at a
  time, so attaching one another record holds moves it. An upload created by a different user, or an id that
  does not exist, is rejected with `422` on the `comparables` key.

**Approval.** The LPO is created as `lpo`. If the company has an active approval template for LPOs, and it
forms a chain for this user, the LPO drops to `pending` and waits for that chain:

- While `pending` the supplier cannot see it, and no jobcard task is raised for them.
- On final approval it moves to `lpo` and the supplier gets the "Upload jobcard" task.
- On rejection it moves to `cancelled`.

With no template, a bypass role, or no applicable steps, it stays `lpo` and the supplier's jobcard task is
raised at once.

Example request:

```json
{
  "facility_id": 4,
  "type": "work",
  "title": "Replace borehole pump",
  "vendor_id": 118,
  "currency_id": 1,
  "expense_type_id": 7,
  "expense_sub_type_id": 21,
  "delivery_at": "2026-10-10",
  "notes": "Access through the service gate.",
  "assigned_technician_id": 52,
  "winning_bid_upload_id": 9031,
  "comparables": [9032, 9033],
  "items": [
    {
      "type": "product",
      "purchase_item_id": 311,
      "notes": "1.5HP submersible",
      "quantity": 1,
      "stock_keeping_unit_id": 1,
      "cost": 85000,
      "tax_id": 2,
      "discount_amount": 5000
    },
    {
      "type": "service",
      "purchase_item_id": "Pump installation labour",
      "quantity": 1,
      "stock_keeping_unit_id": 3,
      "cost": 15000,
      "tax_id": 2
    }
  ]
}
```

Success response (`DataResource`):

- `message`: `LPO created successfully.`
- `lpo`: the created LPO, with `facility`, `vendor`, `currency`, `expenseType`, `expenseSubType`,
  `expenseCategory`, `assignedTechnician`, `winningBidUpload`, `comparables` and `items` loaded.

```json
{
  "message": "LPO created successfully.",
  "lpo": {
    "id": 412,
    "is_direct": true,
    "type": "work",
    "title": "Replace borehole pump",
    "notes": "Access through the service gate.",
    "amount": "100000.00000",
    "discount_amount": "5000.00000",
    "amount_after_discount": "95000.00000",
    "tax": "15200.00000",
    "total": "110200.00000",
    "status": { "value": "pending", "color": "warning" },
    "delivery_at": { "raw": "2026-10-10T00:00:00.000000Z", "formatted": "10 Oct, 2026", "diff": "2 weeks from now" },
    "facility": { "id": 4, "name": "Kilimani Heights" },
    "vendor": { "id": 118, "name": "Aqua Pumps Ltd" },
    "winningBidUpload": { "id": 9031, "title": "Aqua Pumps quote.pdf" },
    "comparables": [
      { "id": 9032, "title": "Rift Water quote.pdf" },
      { "id": 9033, "title": "Borehole Masters quote.pdf" }
    ],
    "procurementRequest": null,
    "items": [
      { "id": 1201, "title": "Submersible pump", "type": "product", "quantity": "1.000", "cost": "85000.00000", "discount_amount": "5000.00000", "tax": "12800.00000", "total": "92800.00000" },
      { "id": 1202, "title": "Pump installation labour", "type": "service", "quantity": "1.000", "cost": "15000.00000", "discount_amount": "0.00000", "tax": "2400.00000", "total": "17400.00000" }
    ]
  }
}
```

Errors:

- `403` — missing permission, or not allocated to `facility_id`.
- `422` — validation, keyed by field (`items.0.cost`, `comparables.1`, `expense_sub_type_id`, ...).

### Notifications when an LPO is issued

An LPO is *issued* when it reaches `lpo`: a workflow LPO on the step that generates it, a direct LPO on
creation (no chain) or on its final approval.

| Who | Notification `type` | When | Channels |
|---|---|---|---|
| The LPO's supplier | `App\Notifications\PropertyManagement\Procurement\LpoIssuedNotification` | Every issued LPO | database, mail, SMS |
| Every other supplier who quoted on the request | `App\Notifications\PropertyManagement\Procurement\BidRejectedNotification` | Workflow **work** LPOs only | database, mail, SMS |
| The procurement request's creator | `App\Notifications\PropertyManagement\Procurement\LpoRaisedNotification` | Workflow LPOs only (a direct LPO's creator raised it themselves) | database, mail, SMS |

Mail and SMS go only where the recipient has an address / phone number.

When a workflow work LPO is generated, every other bid on the request that is not already `rejected` is set
to `rejected`, with `rejection_reason` "Another quotation was selected for this request." The winning bid
is left as it is.

`data` payloads (besides `title`, `message`, `resource_type`, `resource_id`, `resource_url`):

- `LpoIssuedNotification` / `LpoRaisedNotification`: `lpo_id`, `procurement_request_id` (null for direct),
  `facility_name`, `total`, `delivery_at`. `resource_url` opens the LPO in the recipient's portal.
- `BidRejectedNotification`: `bid_id`, `procurement_request_id`, `request_title`, `rejection_reason`.
  `resource_url` opens the supplier's quote.

### Show an LPO

`GET /api/v1/app/{company}/property-management/procurement/lpos/{lpo}`

Loads `currency`, `documentUpload`, `facility`, `expenseType`, `expenseSubType`, `expenseCategory`, `vendor`,
`assignedTechnician`, `winningBidUpload`, `comparables`, `procurementRequest` (null for a direct LPO) and
`items` with `purchaseItem`, `stockKeepingUnit` and `taxType`.

### Response fields

`FacilityProcurementLpoResource`:

| Field | Notes |
|---|---|
| `id` | - |
| `is_direct` | `true` for an LPO raised through `POST /lpos` |
| `type` | `work`, `purchase` |
| `title` | Every LPO: a direct LPO's as entered; a workflow LPO's is copied from its procurement request |
| `notes`, `amount`, `discount_amount`, `amount_after_discount`, `tax`, `total`, `expense_category_id` | - |
| `delivery_at`, `delivered_at`, `created` | `raw`, `formatted`, `diff` |
| `status` | `{ value, color }` — `pending`, `lpo`, `delivered`, `cancelled` |
| `document_number`, `documentUpload` | The supplier's jobcard / delivery note, set when they submit it |
| `work_advice`, `quality_rating`, `speed_rating`, `communication_rating`, `rating`, `comments` | Set on review |
| `facility` | The property |
| `expenseType`, `expenseSubType`, `expenseCategory`, `currency`, `vendor`, `assignedTechnician` | When loaded |
| `winningBidUpload` | Direct LPOs: the winning quotation |
| `comparables` | Direct LPOs: the comparable quotations (array of uploads) |
| `procurementRequest` | Workflow LPOs only |
| `items` | When loaded |
| `permissions` | Instance permissions |

### Review submitted LPO document

`PUT /api/v1/app/{company}/property-management/procurement/lpos/{lpo}/review`

Purpose:

- Used by App users to accept a vendor-submitted jobcard / delivery document. It marks the LPO `delivered`
  and generates the supplier bill for it (once per LPO).

Request body (`CloseLpoData`; `document_number` and `document_upload_id` are taken from the LPO, and
`delivered_at` is set to now):

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `work_advice` | No | string | - |
| `quality_rating` | No | integer | Default `0` |
| `speed_rating` | No | integer | Default `0` |
| `communication_rating` | No | integer | Default `0`. `rating` is the average of the three |
| `comments` | No | string | - |

The bill takes its property and expense type / sub-type from the LPO itself, so direct and workflow LPOs
bill the same way.

Success response:

- `message`: `Document accepted!`
- `lpo`: the updated LPO with `documentUpload`

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
  - `filter[search]` — free text over the contract's id, title, supplier name and property name
  - **Exact** (`=`): `filter[id]`, `filter[status]`, `filter[facility_id]`, `filter[vendor_id]`,
    `filter[expense_type_id]`, `filter[expense_category_id]`, `filter[expense_sub_type_id]`, `filter[asset_id]`
  - **Partial** (`LIKE`): `filter[title]`, `filter[start_at]`, `filter[end_at]`, `filter[created_at]`
  - Aliases: `filter[property_id]` → `facility_id`, `filter[supplier_id]` → `vendor_id`
- Sort:
  - `sort=id,title,status,start_at,end_at,created_at`
- Include:
  - `include=bills`

> **Exact versus partial matters here.** Ids and `status` are exact: they were previously declared as
> bare strings, which Spatie turns into partial filters, so `filter[facility_id]=6` matched every
> property whose id contained a 6, and `filter[status]=active` also returned `inactive` contracts.
>
> `title` and the dates stay partial on purpose — a title is a text match, and a partial date lets
> `filter[created_at]=2026-09` mean "that month".
>
> The UI's *property* and *supplier* are the schema's *facility* and *vendor*. Both spellings are
> accepted, so either can be used; an unknown filter name is a `400`, not an empty list.

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
| `uploads` | No | array of integer | Supporting documents — addendums, extensions. Each must exist in `uploads.id`. See below |
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


### Supporting documents

`uploads` is a list of `uploads.id` values — **addendums, contract extensions and anything else the
contract picks up over its life**. It sits beside the single signed document above, which is
unchanged: that column still holds the agreement itself, and is never part of this list.

| Behaviour | Result |
|---|---|
| `uploads: [1, 2]` | Those two are attached |
| `uploads: [1]` on a contract holding 1 and 2 | **2 is released** — the list is the complete set, not an addition |
| `uploads` **omitted entirely** | Attachments are left untouched — safe for a partial update |
| `uploads: []` | All are released |

Released means **unowned, not deleted**: the file still exists and can be attached elsewhere.

An upload may belong to one record at a time, so attaching one that another contract holds moves it.
An upload created by a different user is rejected with `422` on the `uploads` key, as is an id that
does not exist.

Documents keep the name they were uploaded with — there is no document type to set.

The response exposes them as `uploads`, an array of the standard upload resource.

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
