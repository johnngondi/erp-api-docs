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
| `lease_component_id` | Yes | integer | A component at least one of the lease's units carries. See [Only a component the lease has](#only-a-component-the-lease-has) |
| `next_due` | Yes | date (`YYYY-MM-DD`) | The next date this escalation applies — the **only** date that drives it |
| `end_at` | No | date (`YYYY-MM-DD`) | The date this escalation stops. Omit for one that runs indefinitely. Must be after `next_due` |
| `cycle` | Yes | string | `days`, `weeks`, `months`, `years` |
| `period` | Yes | integer | Number of cycle units. At least 1 |
| `rate` | Yes | decimal | Percent. Fractions are kept, so `7.5` means 7.5% |

There is **no start date**. `start_at` is no longer accepted or returned: an escalation applies on
`next_due`, then `next_due` moves on by `period` × `cycle`. A `start_at` sent anyway is ignored.

### Only a component the lease has

An escalation names a component, not a unit, and raises **every unit on the lease that carries
it**: Parking on one of three units raises that one unit, Rent on all three raises all three. A
component none of the lease's units carries would raise nothing, so it is refused (row 145):

```json
{
  "message": "This lease has no unit with Parking.",
  "errors": { "lease_component_id": ["This lease has no unit with Parking."] }
}
```

On an edit the check runs only when `lease_component_id` changes, so an escalation whose unit was
since removed from the lease can still have its rate and dates corrected.

### When an escalation stops

It applies on `next_due` only while `next_due` is on or before **both**:

- its own `end_at` (when set), and
- the lease's `end_at` (when set).

Once `next_due` falls after either, the escalation is finished: the component **keeps the rent it
escalated to** and stops rising, and nothing further applies. An escalation that came due before its
stop date still applies even if the nightly run reaches it late.

### Several escalations on one component

A component may carry more than one escalation — "5% in year one, 10% per year afterwards" is two
rows against the same component — **provided their periods do not overlap**.

This is not bookkeeping. Every escalation whose `next_due` has passed is applied, and each
multiplies the component's monthly cost by its own rate. Two overlapping escalations on one
component therefore both fire and compound: 5% and 10% together give 15.5% rather than 10%, every
cycle, and it never shows up as a line on an invoice.

An escalation with no `end_at` runs indefinitely, so nothing can follow it until one is set.

Two escalations overlap when one's `next_due` → `end_at` window meets the other's.

| Error key | When |
|---|---|
| `lease_component_id` | None of the lease's units carries the component |
| `end_at` | The stop date is not after `next_due` |
| `next_due` | The window overlaps another escalation on the same component |

Example request:

```json
{
  "lease_component_id": 9,
  "next_due": "2026-03-01",
  "end_at": "2027-02-28",
  "cycle": "months",
  "period": 12,
  "rate": 5
}
```

Example response:

```json
{
  "message": "Lease escalation created successfully.",
  "lease_escalation": {
    "id": 61,
    "end_at": { "raw": "2027-02-28", "formatted": "28 Feb, 2027" },
    "cycle": "months",
    "period": 12,
    "rate": 5,
    "next_due": { "raw": "2026-03-01", "formatted": "01 Mar, 2026", "diff": "5 months ago" }
  }
}
```

`end_at.raw` and `next_due.raw` are plain dates (`YYYY-MM-DD`), not UTC timestamps.
Send them back as they are: a timestamp like `2026-02-28T21:00:00Z` is midnight on 1 March in
Nairobi, and taking its date part moves the escalation a day earlier on every save.

## Update Lease Escalation

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations/{leaseEscalation}`

Use the same payload shape as create. Prefill the edit form from the `raw` dates above.

## Delete Lease Escalation

`DELETE /api/v1/app/{company}/property-management/lease-management/leases/{lease}/escalations/{leaseEscalation}`

## How an escalation shows on an invoice

When an escalation takes effect partway through a billing period, the generated invoice still
carries **one line per component**, billed at the average rate across the period:

| Field | Value |
|---|---|
| `quantity` | The months in the period, e.g. `3` for a quarter |
| `amount` | Each stretch's monthly rate × the months it covers, added together |
| `cost` | `amount ÷ quantity` |
| `notes` | The period label, e.g. `Jan - Mar 2027` |

Example: rent is 100,000 a month, rising to 110,000 on 1 February, billed quarterly Jan–Mar.

```json
{
  "notes": "Jan - Mar 2027",
  "quantity": 3,
  "cost": 106666.66667,
  "amount": 320000
}
```

`amount` is exact. `cost` is rounded, so `cost × quantity` printed to two decimals can be a cent
off — always show and total `amount`, never recompute it.

This applies to every component with an escalation configured, not only rent. An escalation in
the middle of a month is shared by days, so one on 15 February in the example above bills 315,000.
A period with no escalation in it is billed exactly as before.

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
