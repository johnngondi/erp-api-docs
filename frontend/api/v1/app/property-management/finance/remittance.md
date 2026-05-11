# Remittances API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/remittances`

## Endpoints

- `GET /remittances`
- `GET /remittances/create` (preview options/calculation before create)
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
