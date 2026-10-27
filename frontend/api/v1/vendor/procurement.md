# Vendor Procurement API

Base route:

`/api/v1/vendor/procurement`

Auth:

- Requires `auth:sanctum` and vendor portal access.
- Vendor-scoped data is enforced in queries/policies.

## Dates

Quote, LPO and site visit `delivery_at` come back with `raw` as a **plain date**, `"2026-10-01"` (row 142). They used to be UTC
timestamps - `"2026-09-30T21:00:00.000000Z"` for 1 Oct in Nairobi - and a form that sent `raw` back
saved the day before. The server now reads any date it is sent as the day it means in
Nairobi, but send `YYYY-MM-DD`.

## RFQ Open Jobs

Invited suppliers who have not quoted on an open request receive a daily quotation reminder (working days, 09:00) until they quote or the deadline passes; see the `QuoteSubmissionReminderNotification` type in [notifications.md](notifications.md).

**Cancelled invitations (rebase).** Staff can rebase a request: the request for quotation is
cancelled, every supplier asked is told why (`RfqCancelledNotification`), and any quote sent is
**shelved** (status `shelved`, with the reason). A cancelled invitation no longer lists or opens the
job and cannot quote, send a site visit report or assign a technician (`403`). When the request is
sent out again, the suppliers who had quoted are asked again (`RfqReopenedNotification`) and may send
a new quote; their shelved quote stays as history. Other suppliers may be newly invited
(`RfqInvitationNotification`). When too few quotes came in by the deadline, staff may extend it
(`RfqDeadlineExtendedNotification` to every supplier already asked); quoting reopens until the new
deadline.

**Default supplier first.** When a property or category names you as its default supplier, a work
request may come to you alone first, for a limited window (the request's `deadline`). Quote within
it, or [decline](#decline-an-invitation) if you cannot do the job, and the request goes to other
suppliers at once. If you do neither, it goes to them when the window ends; you stay invited and
may still quote until the new deadline.

**Declined invitations.** A supplier who declines still sees the job (`permissions.view`) but can no
longer quote, send a site visit report or assign a technician (`403`), and gets no more quote
reminders or deadline notices for it.

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
- `filter[not_quoted]` (`true`: only jobs the vendor has not quoted on; jobs the vendor declined are left out too)
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
- `quote` (the vendor's live quote summary if present; a quote shelved by a rebase is not shown here, so a supplier asked again can quote anew)
- `facility`
- `asset`
- `owner`
- `expenseType`
- `expenseSubType`
- `status` (enum object)
- `deadline` (`raw`, `formatted`, `diff`)
- `created` (`raw`, `formatted`, `diff`)
- `technician` (`VendorUserResource` or `null`): the technician the vendor assigned
- `declined` (`{ raw, formatted, reason }` or `null`): set once the vendor declined to quote; `reason` may be `null`
- `permissions` (`{ view, bid, sendSiteVisitReport, assignTechnician, decline }`): what the vendor may do on the job. After declining, `decline`, `bid`, `sendSiteVisitReport` and `assignTechnician` are `false` and `view` stays `true`

### Show open job

`GET /rfq/open-jobs/{openJob}`

Notes:

- Returns `404` if request is not an open job (`status != order` or `type != work`), or the vendor's invitation to it was cancelled by a rebase.
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

### Decline an invitation

`POST /procurement/rfq/open-jobs/{openJob}/decline`

The vendor turns down the invitation to quote on a work request.

Request body:

| Field | Type | Required | Notes |
|---|---|---|---|
| `reason` | string | No | Max 1000 characters. Shown to staff on the request's suppliers list (`decline_reason`) |

Allowed (`permissions.decline`) only while:

- the job is open for quotation (`status=order`) and is a `work` request;
- the vendor's invitation is open: not cancelled by a rebase and not already declined;
- the vendor has no live quote on it (a vendor who quoted withdraws the quote instead).

Otherwise **403**.

What happens:

- the invitation records `declined_at` and the reason; the job's `declined` is set;
- the vendor can still view the job but can no longer quote, send a site visit report or assign a
  technician, and gets no more quote reminders or deadline notices for it;
- `filter[not_quoted]=true` no longer lists the job;
- when the vendor is the request's default supplier and the request is still in its default-supplier
  stage, the request goes to the other suppliers at once.

Success response (`DataResource`):

```json
{
  "data": {
    "message": "You have declined to quote on this request.",
    "open_job": {
      "id": 42,
      "title": "Fix the borehole pump",
      "declined": {
        "raw": "2026-10-12T09:30:00.000000Z",
        "formatted": "12 Oct, 2026 09:30",
        "reason": "We do not service this pump model."
      },
      "permissions": {
        "view": true,
        "bid": false,
        "sendSiteVisitReport": false,
        "assignTechnician": false,
        "decline": false
      },
      "...": "the rest of OpenJobResource"
    }
  }
}
```

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
- `status` (enum object) — `bid`, `recommended`, `selected`, `rejected`, `shelved` (`{ value: 'shelved', color: 'warning' }`)
- `rejection_reason` — set when the quote lost to another supplier's (`Another quotation was selected for this request.`)
- `round` — the request's quote round the quote was sent in (1, then one more after each rebase)
- `shelved` — `{ at: { raw, formatted }, reason }` when a rebase set the quote aside (the reason is the one the supplier was sent), else `null`
- `permissions` — `update` and `delete` are `false` on a shelved quote: it can no longer be changed or withdrawn
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

- One live quote per supplier per job: a second one returns **422** (`You have already submitted a quotation for this job.`). A quote shelved by a rebase does not count, so a supplier asked again after a rebase can quote anew.
- A vendor whose invitation a rebase cancelled gets **403**; after the deadline (or while staff decide on a quote shortfall) the job is closed: **422** (`Quotation submission is closed for this request.`).
- The quote is stamped with the request's current quote `round`, and the invitation records `quoted_at`.
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

Only the vendor's own LPOs are listed, and never a `pending` one (an order still awaiting approval has
not been placed).

Query params:

- `filter[search]`
- `filter[facility_id]`, `filter[is_direct]` (`1`/`0`), `filter[type]` (`work`, `purchase`)
- `filter[expense_category_id]`, `filter[status]`, `filter[delivered_at]`, `filter[rating]`
- `filter[amount]`, `filter[total]`, `filter[created_at]`
- `sort=id,created_at,delivered_at,amount,total,rating,status`
- `include=items` (optional)
- `per_page` (optional)
- `page` (optional)

Response item shape (`ProcurementLpoResource`):

- `id`
- `is_direct` — `true` when staff raised the LPO directly, without a procurement request
- `type` — `work`, `purchase`
- `title` — every LPO has one: a direct LPO's as entered, a workflow LPO's copied from its procurement request
- `notes`
- `amount`
- `discount_amount`
- `amount_after_discount`
- `tax`
- `total`
- `expense_category_id`
- `delivery_at` (`raw`, `formatted`, `diff`)
- `delivered_at` (`raw`, `formatted`, `diff`)
- `facility` — the property
- `procurementRequest` — `null` for a direct LPO
- `expenseCategory`
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
- `facility`
- `procurementRequest` (`null` for a direct LPO)
- `items.purchaseItem`
- `items.stockKeepingUnit`
- `items.taxType`

A `pending` LPO returns `404`.

### Notifications

Suppliers are notified in the portal inbox. The LPO notification also goes by mail and SMS where they
have an address / phone number; a rejected quote is in-app only:

| Notification `type` | When | `resource_url` opens |
|---|---|---|
| `App\Notifications\PropertyManagement\Procurement\LpoIssuedNotification` | An LPO is issued to you — straight away, or once a pending LPO is approved | The LPO |
| `App\Notifications\PropertyManagement\Procurement\BidRejectedNotification` | Another supplier's quote won the request you quoted on | Your quote |

Alongside the LPO notification, an "Upload jobcard" pending task is raised for the LPO.

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

Submit behavior:

- Backend always sets `delivered_at` to current time.
- Backend computes overall `rating` as average of quality/speed/communication.
- Your "Upload jobcard" task is cleared, and the LPO's creator (the staff user who raised it, or who
  raised its procurement request) gets a "Review document and close LPO" task. Submitting again
  replaces that task, so the creator never holds two.

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
- `filter[expense_category_id]`
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
- `expenseCategory`
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
- `expenseCategory`
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
- For create/update payload shape in app routes, include optional `expense_category_id` (must exist in `expense_categories.id`); backend derives it from `expense_type_id` when omitted.
