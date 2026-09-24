# Property Management Dashboard API

Base route:

`/api/v1/app/{company}/property-management`

## Endpoint

- `GET /api/v1/app/{company}/property-management`

Returns the billing-health dashboard: three money cards, the collections chart, the LPO status
donut, the expiring-leases list, the open-tickets list and the acting user's dashboard permissions.

## Scope

Every widget reads the same set of properties, resolved once:

1. the properties the acting user is **allocated to** (see
   `docs/backend/property-allocation-scoping.md`; a user allocated to nothing sees empty widgets),
2. narrowed by the `landlord_id`, `facility_id` filters.

`filters.property_count` in the response says how many properties survived, so the UI can show
"No data for the selected filter scope" when it is `0`.

## Query Parameters

All are optional. Omit them all for the default view: every allocated property, **All Time**.

| Param | Type | Description |
|---|---|---|
| `landlord_id` | integer (`users.id`) | Only properties owned by this landlord. |
| `facility_id` | integer (`facilities.id`) | Only this property. |
| `period_from` | date (`Y-m-d`) | Start of the period. Omit for an open start. |
| `period_to` | date (`Y-m-d`) | End of the period. Omit for an open end. |

Period rules:

- Neither bound given = **All Time**. Cards have no previous period to compare against, so their
  `previous` and `change_percent` are `null`.
- One bound given = open on the other side. Same comparison rule as All Time.
- Both bounds given = a bounded period. The **previous period** is the same number of days
  immediately before `period_from`; the cards compare against it.
- If `period_from` is after `period_to` the two are swapped.

An unknown `landlord_id`, `facility_id` or `company_department_id`, or an unparsable date, returns
`422` with the usual validation payload.

## Response

```json
{
  "data": {
    "filters": {
      "landlord_id": 12,
      "facility_id": null,
      "period": { "from": "2026-05-01", "to": "2026-05-31" },
      "previous_period": { "from": "2026-03-31", "to": "2026-04-30" },
      "property_count": 3
    },
    "currency": { "id": 1, "code": "KES", "name": "Kenyan Shilling" },
    "cards": {
      "billings":    { "value": 4910000, "previous": 4500000, "change_percent": 9.1 },
      "collections": { "value": 4310000, "previous": 4310000, "change_percent": 0 },
      "arrears":     { "value": 600000,  "previous": 800000,  "change_percent": -25 }
    },
    "collections_report": {
      "period": { "from": "2026-05-01", "to": "2026-05-31" },
      "granularity": "weekly",
      "min_value": 145000,
      "max_value": 580000,
      "data": [
        { "date": "2026-04-27", "label": "27 Apr", "value": 580000, "collected": 580000, "billed": 610000 },
        { "date": "2026-05-04", "label": "04 May", "value": 290000, "collected": 290000, "billed": 300000 }
      ]
    },
    "lpo_status": {
      "total": 9,
      "items": [
        { "key": "new",         "label": "New",         "color": "success", "count": 3, "percent": 33.3 },
        { "key": "in_progress", "label": "In Progress", "color": "primary", "count": 3, "percent": 33.3 },
        { "key": "late",        "label": "Late",        "color": "danger",  "count": 3, "percent": 33.3 }
      ]
    },
    "expiring_leases": {
      "total": 1,
      "period": { "from": "2026-05-01", "to": "2026-05-31" },
      "items": [
        {
          "id": 88,
          "tenant": { "id": 41, "name": "Acme Traders Ltd" },
          "property": { "id": 7, "name": "Maple Heights" },
          "end_at": "2026-05-20",
          "days_left": 26,
          "status": { "value": "active", "color": "success" }
        }
      ]
    },
    "open_tickets": {
      "total": 4,
      "items": [
        {
          "id": 36,
          "reference": "TK-036",
          "title": "HVAC noisy in office wing",
          "property": { "id": 7, "name": "Maple Heights" },
          "owner": { "id": 19, "name": "Brian" },
          "priority": { "value": "normal", "color": "primary" },
          "status": { "value": "open", "color": "primary" },
          "created": { "raw": "2026-05-02T08:14:00.000000Z", "formatted": "02 May, 2026", "diff": "3 weeks ago" }
        }
      ]
    },
    "permissions": ["view-facility", "create-ticket"]
  }
}
```

## What each widget counts

Money figures share their definitions with the reports, so a card never disagrees with the report
behind it.

### Cards

| Card | Definition |
|---|---|
| `billings` | Sum of `lease_billings.total` dated in the period, for leases on the scoped properties. Same as the Collections Summary report's "billed". |
| `collections` | Sum of **confirmed** `lease_collections.total` dated in the period. A cancelled receipt is not money collected. Same as the reports' "collected". |
| `arrears` | `total - paid` over every issued invoice raised **on or before the end of the period** (today for an open end). Pending and cancelled invoices are not owed. Same as the Ageing Report and Debtors Listing. |

`change_percent` is `(value - previous) / |previous| * 100`, rounded to one decimal. It is `null`
when there is no previous period or the previous value is zero; render that as `+0.0%` or hide the
comparison line.

`currency` is the one reporting currency the scoped properties share. It is `null` when they use
different reporting currencies or no property is in scope; in that case show the figures without
a currency symbol.

### Collections report

Collected (`value`, also `collected`) against `billed`, bucketed over the period:

| Period length | `granularity` | Bucket | `label` |
|---|---|---|---|
| under 30 days | `daily` | one day | `25 Jan` |
| 30 to 89 days | `weekly` | Monday-to-Sunday week, `date` is the Monday | `27 Apr` |
| 90 days or more | `monthly` | calendar month, `date` is the 1st | `Apr 2026` |

An open start runs from the first billing or collection on record for the scoped properties (or the
start of the current month when there is none); an open end runs to today. Every bucket in the span
is present, zero-filled, so the line never has gaps.

### LPO status

Open purchase orders raised in the period, on the scoped properties:

| Key | Meaning |
|---|---|
| `new` | status `pending`: raised, awaiting approval. |
| `in_progress` | status `lpo`: issued to the vendor and not yet due (`delivery_at` is today or later). |
| `late` | status `lpo` whose `delivery_at` has passed with no delivery recorded. |

Delivered and cancelled orders are closed, so they are not in the donut or in `total`. `percent`
is each count over `total`, one decimal; all three are `0` when `total` is `0`.

### Expiring leases

Active leases whose `end_at` falls in the window, soonest first, at most 10 rows (`total` is the
full count). The window is the period; an open start is today and an open end is 90 days after the
start, so All Time reads as "the next 90 days". `days_left` is negative once the end date has passed
(only possible with an explicit past period).

### Open tickets

Tickets with status `open` raised in the period on the scoped properties, newest first, at most 10
rows (`total` is the full count). `reference` is `TK-` plus the id zero-padded to three digits.
`owner` is the user who raised the ticket.

## Frontend Notes

- Treat the payload as read-only dashboard data.
- Send the filter values as plain query parameters, e.g.
  `?landlord_id=12&company_department_id=3&period_from=2026-05-01&period_to=2026-05-31`. The
  "All Time" choice simply omits both period params.
- Every list widget carries its own `total` and `items`; render "See All" from `total`.
- If any widget key is missing, hide that widget rather than failing the whole page.

## Errors

Use shared error handling behavior from `docs/frontend/app/README.md`:

- Show `4xx` messages to users (`422` carries per-field validation messages).
- Show generic message for `5xx`.
