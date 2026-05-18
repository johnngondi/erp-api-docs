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
- `PATCH /remittances/{remittance}/approve`
- `PATCH /remittances/{remittance}/cancel`

Notes:

- `PUT/PATCH /remittances/{remittance}` route exists but is currently not implemented in controller.

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
  - `include=collections,expenses`
- Pagination:
  - `per_page`, `page`

Enum filter options:

- `filter[status]`: `pending`, `unpaid`, `paid`, `cancelled` (from `RemittanceStatus` enum)

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
          "status": { "value": "pending", "color": "info" },
          "landlord": { "id": 30, "name": "Jane Landlord" },
          "facility": { "id": 22, "name": "Riverside Plaza" }
        }
      ]
    }
  }
}
```
