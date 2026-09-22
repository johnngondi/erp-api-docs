# Remittances API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/remittances`

## Endpoints

- `GET /remittances`
- `GET /remittances/create` (preview options/calculation before create)
- `GET /remittances/create-bulk` (bulk preview by period; optional landlord filter)
- `POST /remittances/create-bulk` (bulk create remittances)
- `POST /remittances`
- `GET /remittances/{remittance}`
- `DELETE /remittances/{remittance}`
- `PATCH /remittances/{remittance}/cancel`

Notes:

- `PUT/PATCH /remittances/{remittance}` route exists but is currently not implemented in controller.
- **`PATCH /remittances/{remittance}/approve` has been removed.** Remittances are approved
  through the shared approval chain now - see [Approval](#approval) below.

## List Remittances

`GET /api/v1/app/{company}/property-management/finance/remittances`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs)
  - `filter[facility_id]`, `filter[landlord_id]`, `filter[status]`
  - `filter[created_at]`, `filter[period_from]`, `filter[period_to]`
- Sort:
  - `sort=id,period_from,period_to,total_income,total_expenses,remittable_amount,created_at,updated_at`
- Include:
  - `include=receipts,expenses`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[status]`: `pending`, `unpaid`, `paid`, `cancelled` (from `RemittanceStatus` enum)

`pending` now means "waiting on an approver". A company with no approval template for
remittances never produces one - see [Approval](#approval).

Sample list response (`FacilityRemittanceResource`):

```json
{
  "data": [
    {
      "id": 910,
      "notes": "May landlord remittance",
      "period_from": {
        "raw": "2026-05-01T00:00:00.000000Z",
        "formatted": "01 May, 2026",
        "diff": "1 week ago"
      },
      "period_to": {
        "raw": "2026-05-07T23:59:59.000000Z",
        "formatted": "07 May, 2026",
        "diff": "2 days ago"
      },
      "is_advance": false,
      "total_income": "120000.00",
      "total_expenses": "15000.00",
      "remittable_amount": "105000.00",
      "landlord": { "id": 30, "name": "Jane Landlord" },
      "facility": { "id": 22, "name": "Riverside Plaza" },
      "currency": { "id": 1, "code": "KES" },
      "status": { "value": "unpaid", "color": "warning" },
      "created": {
        "raw": "2026-05-09T10:18:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      },
      "updated": {
        "raw": "2026-05-09T10:18:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      }
    }
  ]
}
```

## Create/preview payload

| Field | Required | Type |
|---|---|---|
| `facility_id` | Yes | integer (`facilities.id`) |
| `period_from` | Yes | date |
| `period_to` | Yes | date |

## Bulk Remittance Preview

`GET /api/v1/app/{company}/property-management/finance/remittances/create-bulk`

Query params:

| Field | Required | Type | Notes |
|---|---|---|---|
| `period_from` | Yes | date | Start of remittance period |
| `period_to` | Yes | date | End of remittance period. Must be `>= period_from` |
| `landlord_id` | No | integer | Optional filter (`users.id`) |

Response shape (`DataResource`):

- Returns only summary values per facility.
- Does not include detailed collections or expense line items.

Sample response:

```json
{
  "data": {
    "message": "Bulk remittance preview generated successfully.",
    "facilities": [
      {
        "landlord": { "id": 30, "name": "Jane Landlord" },
        "facility": { "id": 22, "name": "Riverside Plaza" },
        "total_collections": 120000,
        "management_fee_pretax": 10000,
        "management_fee": 11600,
        "total_expenses": 15000,
        "remittable_amount": 93400
      }
    ]
  }
}
```

## Bulk Remittance Create

`POST /api/v1/app/{company}/property-management/finance/remittances/create-bulk`

Route name: `property-management.finance.remittances.store-bulk`

Payload:

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_ids` | Yes | array[int] | Must be non-empty, distinct, and all ids must exist in `facilities.id` |
| `period_from` | Yes | date | Start of remittance period |
| `period_to` | Yes | date | End of remittance period. Must be `>= period_from` |

Sample payload:

```json
{
  "facility_ids": [22, 23, 24],
  "period_from": "2026-05-01",
  "period_to": "2026-05-31"
}
```

Sample response:

```json
{
  "data": {
    "message": "Bulk remittances created successfully.",
    "remittances": {
      "data": [
        {
          "id": 910,
          "notes": "FacilityRemittance for Riverside Plaza for Period 2026-05-01 - 2026-05-31",
          "total_income": "120000.00",
          "total_expenses": "26600.00",
          "remittable_amount": "93400.00",
          "status": { "value": "pending", "color": "secondary" },
          "landlord": { "id": 30, "name": "Jane Landlord" },
          "facility": { "id": 22, "name": "Riverside Plaza" }
        }
      ]
    }
  }
}
```

Bulk create raises a chain per remittance, exactly as the single create does. The `pending`
above is what a company with an approval template sees; without one each remittance comes back
`unpaid`. See [Approval](#approval).

## Approval

`FacilityRemittance` is a registered approvable model (`config('approvals.models')`) and is
driven by the shared approval framework, not by an endpoint of its own. Act on a remittance's
chain through `POST /access-management/approval-steps/{approvalStep}` - see
[Approval Steps](../../access-management/approvals.md). Configure the chain itself under
[Approval Templates](../../access-management/approval-templates.md).

The chain is started once, by `POST /remittances` (and by the bulk create), after the receipts
and expenses the remittance covers have been attached - a step's conditions are written against
those figures, so they have to be there before the chain is shaped.

### Where the chain shows up

Every remittance response carries an `approval_steps` array in the same shape as every other
approvable resource: `step_order`, `role`, `actors`, `status`, `comment`, `is_current`,
`can_act`, `allowed_to_edit`, `can_edit`, `attempt`. An empty array means no chain was raised.
The landlord-portal remittance endpoints do **not** include it - a landlord does not see the
company's internal approvers.

### Condition facts

Fields a template step's `conditions` may be written against. Operators and the rule shape are
documented once in
[Approval Templates → Step conditions](../../access-management/approval-templates.md#step-conditions).

| Fact | Meaning |
|---|---|
| `remittable_amount` | What the landlord is owed - the usual threshold to escalate on |
| `total_income` | Collections in the period |
| `total_expenses` | Expenses deducted in the period |
| `is_advance` | `true` for an advance remittance, paid ahead of collection |
| `facility_id` | The property being remitted for |
| `landlord_id` | Who is being remitted |
| `currency_id` | The remittance currency |

### Status transitions

| Action | The remittance |
|---|---|
| Created, company has no active template | `unpaid` immediately, no chain |
| Created, all steps fall away on conditions | `unpaid` immediately, no chain |
| Created by a holder of a bypass role | `unpaid` immediately, no chain |
| Created, a chain applies | `pending`, chain raised, first step prompted |
| `approve`, not the last step | stays `pending`, next step prompted |
| `approve`, the last step | `unpaid`, `FacilityRemittanceApprovedEvent` fires |
| `review` | stays `pending`, the previous step is reopened |
| `reject` | `cancelled`, and everything it held is released (below) |
| Payment voucher raised against it | `paid` |
| Payment voucher cancelled | back to `unpaid` |

`unpaid` is what "approved" means for a remittance: it is due for payment, and a payment
voucher is what settles it.

### What a rejection releases

A rejection does not only set the status. The remittance lets go of everything it had claimed,
so the next remittance for that period can pick it up:

- the management-fee expense it raised is cancelled (and its bill with it);
- every receipt it covered is unlinked from it;
- every expense it deducted is unlinked from it.

Cancelling a remittance by hand does exactly the same - both run the one release step, so the
two cannot drift apart.

### Cancelling while a chain is open

`PATCH /remittances/{remittance}/cancel` is refused with **403** while the remittance is still
waiting on an approver. A remittance under approval is stopped by rejecting it, which records
who refused it and why; cancelling would go around the people holding it. Once the chain has
finished - or where there was never one - cancelling an `unpaid` remittance works as before.

The endpoint is authorized against `cancel-facility-remittance` (it previously checked
`update-facility-remittance`).

### Re-submission

Rejecting terminates the chain and cancels the remittance. A cancelled remittance is not
edited and re-submitted - raise a new one for the period, which opens its own chain. The
receipts and expenses the rejected one held are free again, so the new one sees them.
