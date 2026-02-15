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
  - `filter[expense_type_id]`, `filter[expense_sub_type_id]`, `filter[created_at]`, `filter[priority]`, `filter[type]`
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
| `vendors` | Conditional | array | Required if `override=true` |
| `deadline` | No | string/date | Optional |
| `has_preferred_vendor` | No | boolean | Default `false` |
| `override` | No | boolean | Default `false` |

## LPOs

Endpoints currently implemented:

- `GET /lpos`
- `GET /lpos/{lpo}`

Notes:

- Other LPO routes exist in route definitions but are not currently implemented in the controller.

List query support:

- Filters:
  - `filter[id]`, `filter[procurement_request_id]`, `filter[assigned_technician_id]`
  - `filter[created_at]`, `filter[delivery_at]`, `filter[delivered_at]`, `filter[rating]`
- Sort:
  - `sort=id,created_at,amount,delivery_at`
- Include:
  - `include=items`
- Fields: not supported

## Contracts

Endpoints:

- `GET /contracts`
- `POST /contracts`
- `GET /contracts/{contract}`
- `PUT/PATCH /contracts/{contract}`
- `DELETE /contracts/{contract}`
- `PUT /contracts/{contract}/status`

List query support:

- Filters:
  - `filter[id]`, `filter[title]`, `filter[status]`, `filter[start_at]`, `filter[end_at]`, `filter[created_at]`
  - `filter[facility_id]`, `filter[expense_type_id]`, `filter[expense_sub_type_id]`, `filter[asset_id]`
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
| `period_in_days` | Yes | integer | Used to compute `end_at` |
| `type` | Yes | string | `fixed`, `on-demand` |
| `amount` | Yes | integer | - |
| `title` | No | string | Optional |
| `notes` | No | string | Optional |
| `currency_id` | No | integer | Must exist in `currencies.id` |
| `expense_type_id` | No | integer | Must exist in `expense_types.id` |
| `expense_sub_type_id` | No | integer | Must exist in `expense_sub_types.id` |
| `asset_id` | No | integer | Must exist in `assets.id` |
| `agreement_upload_id` | No | integer | Must exist in `uploads.id` |
| `has_tax` | No | boolean | Default `false` |
| `tax_id` | No | integer | Must exist in `taxes.id` |
| `creator_id` | No | integer | Must exist in `users.id` |
| `approver_id` | No | integer | Must exist in `users.id` |
| `status` | No | string | `pending`, `active`, `suspended`, `expired`, `inactive` |

Status update payload:

| Field | Required | Type | Allowed Values |
|---|---|---|---|
| `status` | Yes | string | `pending`, `active`, `suspended`, `expired`, `inactive` |

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
  - `PUT/PATCH /inventories/purchase-items/{item}/prices/{price}`
- Note:
  - `PUT/PATCH /inventories/purchase-items/{purchase_item}` and `DELETE /.../{purchase_item}` exist in routes but are currently not implemented.

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

Purchase item price update payload:

| Field | Required | Type |
|---|---|---|
| `price` | Yes | integer |

## Errors

Use shared behavior in `docs/frontend/app/README.md`:

- Render `4xx` messages and validation errors.
- Show generic fallback for `5xx`.
