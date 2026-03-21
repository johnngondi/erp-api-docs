# Vendor Procurement API

Base route:

`/api/v1/vendor/procurement`

Auth:

- Requires `auth:sanctum` and vendor portal access.
- Vendor-scoped data is enforced in queries/policies.

## RFQ Open Jobs

### List open jobs

`GET /rfq/open-jobs`

Query params:

- `filter[id]`
- `filter[title]`
- `filter[facility_id]`
- `filter[asset_id]`
- `filter[expense_type_id]`
- `filter[expense_sub_type_id]`
- `filter[created_at]`
- `filter[priority]`
- `sort=id,title,created_at,priority`
- `per_page` (optional)
- `page` (optional)

Response item shape (`OpenJobResource`):

- `id`
- `type`
- `title`
- `description`
- `is_emergency`
- `require_site_visit`
- `site_visit_instructions`
- `priority` (enum object: `{ value, color }`)
- `quote` (current vendor quote summary if present)
- `facility`
- `asset`
- `owner`
- `expenseType`
- `expenseSubType`
- `status` (enum object)
- `deadline` (`raw`, `formatted`, `diff`)
- `created` (`raw`, `formatted`, `diff`)

### Show open job

`GET /rfq/open-jobs/{openJob}`

Notes:

- Returns `404` if request is not an open job (`status != order` or `type != work`).
- Includes `uploads` in addition to list fields.

### Assign technician

`POST /procurement/rfq/open-jobs/{openJob}/assign-technician`

Request body:

| Field | Type | Required | Notes |
|---|---|---|---|
| `technician_id` | integer | Yes | Must exist in `vendor_users.id` |

Rules:

- Open job must be open for quotation (`status=order`).
- Assignment supports any open-job type (`work` and `purchase`).
- Selected `technician_id` must belong to authenticated vendor.
- Selected vendor user must have `is_technician=true`.

Success response (`DataResource`):

- `message`: `Technician assigned successfully.`
- `open_job_id`
- `technician` (`VendorUserResource`)

## RFQ Quotes

### List quotes

`GET /rfq/quotes`

Query params:

- `filter[id]`
- `filter[procurement_request_id]`
- `filter[notes]`
- `filter[amount]`
- `filter[created_at]`
- `filter[status]`
- `sort=id,procurement_request_id,notes,amount,created_at,status`
- `per_page` (optional)
- `page` (optional)

Response item shape (`JobBidResource`):

- `id`
- `notes`
- `has_tax`
- `job`
- `creator`
- `currency`
- `amount`
- `discount_type`
- `discount_value`
- `discount_amount`
- `amount_after_discount`
- `tax`
- `total`
- `delivery` (`raw`, `formatted`, `diff`)
- `status` (enum object)
- `created` (`raw`, `formatted`, `diff`)

### Create quote

`POST /rfq/open-jobs/{openJob}/quotes`

Authorization:

- Requires ability to `bid` on the open job.

Request body (`ProcurementRequestBidData`):

| Field | Type | Required | Notes |
|---|---|---|---|
| `delivery_at` | string (date/datetime) | Yes | Expected by backend as delivery date |
| `notes` | string | Yes | Quote notes |
| `currency_id` | integer | Yes | Quote currency |
| `items` | array | Yes | Array of quote line items |
| `use_site_visit_report` | boolean | No | Defaults to `false` |
| `site_visit_report_id` | integer/null | No | Optional reference |
| `upload_id` | integer/null | No | Optional upload reference |
| `default_tax_id` | integer/null | No | Optional default tax id |

Item object (`ProcurementRequestBidDataItemData`):

| Field | Type | Required | Notes |
|---|---|---|---|
| `type` | string | Yes | Item type |
| `purchase_item_id` | integer or string | Yes | Existing item id or free-text item name |
| `notes` | string | Yes | Item notes |
| `quantity` | integer | Yes | Quantity |
| `stock_keeping_unit_id` | integer | Yes | Must exist in `stock_keeping_units.id` |
| `cost` | integer | Yes | Unit cost |
| `tax_id` | integer | Yes | Must exist in `taxes.id` |
| `discount_amount` | integer | Yes | Per-item discount amount |

Create behavior:

- If `purchase_item_id` is not integer, backend creates/finds purchase item by name.
- Backend computes `amount`, `amount_after_discount`, `tax`, `total` per item.
- Quote totals are aggregated from item totals.

Success response (`DataResource`):

- `message`: `Quotation created successfully.`
- `procurementRequest`: created quote as `JobBidResource`

### Show quote

`GET /rfq/quotes/{bid}`

Includes:

- `currency`
- `items`
- `items.stockKeepingUnit`
- `items.taxType`
- `creator`
- `procurementRequest`
- `procurementRequest.facility`

### Update/Delete quote

- `PUT/PATCH /rfq/quotes/{bid}`
- `DELETE /rfq/quotes/{bid}`

Notes:

- Routes exist, but controller methods are currently placeholders.

## RFQ Site Visit Reports

### List reports

`GET /rfq/site-visit-reports`

Query params:

- `filter[id]`
- `filter[procurement_request_id]`
- `filter[notes]`
- `filter[created_at]`
- `sort=id,procurement_request_id,notes,created_at`
- `per_page` (optional)
- `page` (optional)

Response item shape (`SiteVisitReportResource`):

- `id`
- `notes`
- `job`
- `technician`
- `delivery` (`raw`, `formatted`, `diff`)
- `status` (enum object)
- `created` (`raw`, `formatted`, `diff`)

### Create report

`POST /rfq/open-jobs/{openJob}/site-visit-reports`

Authorization:

- Requires ability to `sendSiteVisitReport` on the open job.

Request body (`SiteVisitReportData`):

| Field | Type | Required | Notes |
|---|---|---|---|
| `delivery_at` | string (date/datetime) | Yes | Report delivery date |
| `notes` | string | Yes | Report notes |
| `items` | array | Yes | Array of line items |
| `uploads` | array/null | No | Optional uploads payload |

Item object (`SiteVisitReportDataItemData`):

| Field | Type | Required | Notes |
|---|---|---|---|
| `type` | string | Yes | Item type |
| `purchase_item_id` | integer or string | Yes | Existing item id or free-text item name |
| `notes` | string | Yes | Item notes |
| `quantity` | integer | Yes | Quantity |
| `stock_keeping_unit_id` | integer | Yes | Must exist in `stock_keeping_units.id` |

Create behavior:

- If `purchase_item_id` is not integer, backend creates/finds purchase item by name.
- `technician_id` is set to authenticated user.

Success response (`DataResource`):

- `message`: `Site Visit Report created successfully.`
- `report`: created report as `SiteVisitReportResource`

### Show report

`GET /rfq/site-visit-reports/{report}`

Includes:

- `items`
- `items.stockKeepingUnit`
- `technician`
- `procurementRequest`
- `procurementRequest.facility`

### Update/Delete report

- `PUT/PATCH /rfq/site-visit-reports/{report}`
- `DELETE /rfq/site-visit-reports/{report}`

Notes:

- Routes exist, but controller methods are currently placeholders.

## LPOs

### List LPOs

`GET /lpos`

Query params:

- `filter[id]`
- `filter[procurement_request_id]`
- `filter[assigned_technician_id]`
- `filter[created_at]`
- `filter[delivery_at]`
- `filter[delivered_at]`
- `filter[rating]`
- `sort=id,created_at,amount,delivery_at`
- `include=items` (optional)
- `per_page` (optional)
- `page` (optional)

Response item shape (`ProcurementLpoResource`):

- `id`
- `notes`
- `amount`
- `discount_amount`
- `amount_after_discount`
- `tax`
- `total`
- `delivery_at` (`raw`, `formatted`, `diff`)
- `delivered_at` (`raw`, `formatted`, `diff`)
- `procurementRequest`
- `currency`
- `assignedTechnician`
- `document_number`
- `documentUpload`
- `rating`
- `comments`
- `status` (enum object)
- `created` (`raw`, `formatted`, `diff`)
- `items` (when included)

### Show LPO

`GET /lpos/{lpo}`

Includes loaded by controller:

- `currency`
- `documentUpload`
- `procurementRequest`
- `items.purchaseItem`
- `items.stockKeepingUnit`
- `items.taxType`
- `procurementRequest.facility`

### Submit LPO document

`PATCH /lpos/{lpo}/submit-document`

Authorization:

- Requires ability to `submitDoc` on the LPO.

Request body (`CloseLpoData`):

| Field | Type | Required | Notes |
|---|---|---|---|
| `document_number` | string | Yes | Delivery/work document reference |
| `document_upload_id` | integer | Yes | Must exist in `uploads.id` |
| `work_advice` | string/null | No | Optional |
| `quality_rating` | integer | No | Defaults to `0` |
| `speed_rating` | integer | No | Defaults to `0` |
| `communication_rating` | integer | No | Defaults to `0` |
| `comments` | string/null | No | Optional |

Submit behavior:

- Backend always sets `delivered_at` to current time.
- Backend computes overall `rating` as average of quality/speed/communication.

Success response (`DataResource`):

- `message`: `LPO Document Submitted successfully.`
- `lpo`: updated LPO with `documentUpload`

## Contracts

### List contracts

`GET /contracts`

Query params:

- `filter[id]`
- `filter[title]`
- `filter[status]`
- `filter[start_at]`
- `filter[end_at]`
- `filter[created_at]`
- `filter[facility_id]`
- `filter[expense_type_id]`
- `filter[expense_sub_type_id]`
- `filter[asset_id]`
- `sort=id,title,status,start_at,end_at,created_at`
- `include=bills` (optional)
- `per_page` (optional)
- `page` (optional)

Response item shape (`ContractResource`):

- `id`
- `title`
- `notes`
- `has_tax`
- `start_at` (`raw`, `formatted`, `diff`)
- `period_in_days`
- `end_at` (`raw`, `formatted`, `diff`)
- `type`
- `amount`
- `currency`
- `vendor`
- `facility`
- `expenseType`
- `expenseSubType`
- `asset`
- `agreementUpload`
- `tax`
- `withholdings`
- `status` (enum object)
- `created` (`raw`, `formatted`, `diff`)
- `bills` (when included)

### Show contract

`GET /contracts/{contract}`

Includes loaded by controller:

- `expenseType`
- `expenseSubType`
- `asset`
- `facility`
- `agreementUpload`
- `vendor`
- `creator`
- `approver`
- `tax`
- `bills`

### Create/Update/Delete contract

- `POST /contracts`
- `PUT/PATCH /contracts/{contract}`
- `DELETE /contracts/{contract}`

Notes:

- Routes exist on vendor portal.
- Current vendor controller implements read endpoints only (`index`, `show`).
