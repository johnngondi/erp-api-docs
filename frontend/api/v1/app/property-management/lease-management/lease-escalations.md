# Lease Escalations API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations`

## Endpoints

- `GET /leases/{lease}/escalations`
- `POST /leases/{lease}/escalations`
- `GET /leases/{lease}/escalations/{leaseEscalation}`
- `PUT/PATCH /leases/{lease}/escalations/{leaseEscalation}`
- `DELETE /leases/{lease}/escalations/{leaseEscalation}`

## List Lease Escalations

`GET /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations`

Supported query params:

- Filters:
  - `filter[id]`
  - `filter[lease_id]`
  - `filter[lease_component_id]`
  - `filter[start_at]`
  - `filter[end_at]`
  - `filter[cycle]`
  - `filter[period]`
  - `filter[rate]`
  - `filter[next_due]`
  - `filter[created_at]`
- Sort:
  - `sort=id,lease_id,lease_component_id,start_at,end_at,cycle,period,rate,next_due,created_at`
- Include: not supported
- Select fields: not supported
- Pagination: `per_page`, `page`

Example:

`GET /api/v1/app/12/property-management/lease-management/leases/101/escalations?filter[lease_component_id]=9&sort=-start_at`

## Create Lease Escalation

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_component_id` | Yes | integer | Must exist in `lease_components.id` |
| `start_at` | Yes | date (`YYYY-MM-DD`) | The first escalation date, not the tenancy start |
| `end_at` | No | date (`YYYY-MM-DD`) | The date this escalation stops. Omit for one that runs indefinitely. Must be after `start_at` |
| `cycle` | Yes | string | `days`, `weeks`, `months`, `years` |
| `period` | Yes | integer | Number of cycle units. At least 1 |
| `rate` | Yes | decimal | Percent. Fractions are kept, so `7.5` means 7.5% |
| `next_due` | No | date (`YYYY-MM-DD`) | Defaults to `start_at` |

### Several escalations on one component

A component may carry more than one escalation — "5% in year one, 10% per year afterwards" is two
rows against the same component — **provided their periods do not overlap**.

This is not bookkeeping. Every escalation whose `next_due` has passed is applied, and each
multiplies the component's monthly cost by its own rate. Two overlapping escalations on one
component therefore both fire and compound: 5% and 10% together give 15.5% rather than 10%, every
cycle, and it never shows up as a line on an invoice.

An escalation with no `end_at` runs indefinitely, so nothing can follow it until one is set.

Once `end_at` has passed the escalation stops applying and `next_due` stops advancing. The
component **keeps the rent it escalated to** and simply stops rising — it does not revert.

| Error key | When |
|---|---|
| `end_at` | The stop date is not after the start date |
| `start_at` | The period overlaps another escalation on the same component |

Example request:

```json
{
  "lease_component_id": 9,
  "start_at": "2026-03-01",
  "end_at": "2027-02-28",
  "cycle": "months",
  "period": 12,
  "rate": 5,
  "next_due": "2027-03-01"
}
```

Example response:

```json
{
  "message": "Lease escalation created successfully.",
  "lease_escalation": {
    "id": 61,
    "start_at": { "raw": "2026-03-01", "formatted": "01 Mar, 2026", "diff": "5 months ago" },
    "end_at": { "raw": "2027-02-28", "formatted": "28 Feb, 2027" },
    "cycle": "months",
    "period": 12,
    "rate": 5,
    "next_due": { "raw": "2027-03-01", "formatted": "01 Mar, 2027", "diff": "5 months from now" }
  }
}
```

`start_at.raw`, `end_at.raw` and `next_due.raw` are plain dates (`YYYY-MM-DD`), not UTC timestamps.
Send them back as they are: a timestamp like `2026-02-28T21:00:00Z` is midnight on 1 March in
Nairobi, and taking its date part moves the escalation a day earlier on every save.

## Update Lease Escalation

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations/{leaseEscalation}`

Use the same payload shape as create. Prefill the edit form from the `raw` dates above.

## Delete Lease Escalation

`DELETE /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations/{leaseEscalation}`

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
