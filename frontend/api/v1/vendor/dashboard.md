# Vendor Dashboard API

Base route:

`/api/v1/vendor`

## Endpoint

- `GET /api/v1/vendor`

Every figure is the signed-in supplier's own. A supplier staff login sees its supplier's figures.

## Query Parameters

| Param | Type | Required | Description |
|---|---|---|---|
| `months` | integer | No | Months covered by the revenue chart: `3`, `6` or `12`. Defaults to `6`. Any other value is a `422`. |

## Response

Example response:

```json
{
  "data": {
    "stats": {
      "open_jobs": 3,
      "active_quotes": 2,
      "active_contracts": 4,
      "pending_bills": 5
    },
    "contracts_value": [
      { "currency": { "id": 1, "code": "KES" }, "amount": 2500000 }
    ],
    "outstanding_bills": [
      { "currency": { "id": 1, "code": "KES" }, "amount": 184000.5 }
    ],
    "jobs_status": {
      "available": 3,
      "quoted": 2,
      "missed": 1,
      "total": 6
    },
    "revenue": {
      "months": 6,
      "period": { "from": "2026-05-01", "to": "2026-10-31" },
      "series": [
        {
          "currency": { "id": 1, "code": "KES" },
          "total": 905000,
          "data": [
            { "label": "May 2026", "value": 120000 },
            { "label": "Jun 2026", "value": 0 },
            { "label": "Jul 2026", "value": 250000 },
            { "label": "Aug 2026", "value": 180000 },
            { "label": "Sep 2026", "value": 215000 },
            { "label": "Oct 2026", "value": 140000 }
          ]
        }
      ]
    },
    "latest_open_jobs": [
      {
        "id": 41,
        "type": "work",
        "title": "Replace lobby lighting",
        "priority": { "value": "high", "color": "danger" },
        "is_emergency": false,
        "facility": { "id": 3, "name": "Westlands Plaza" },
        "expenseType": { "id": 7, "name": "Electrical" },
        "status": { "value": "order", "color": "info" },
        "deadline": { "raw": "2026-10-12T15:00:00.000000Z", "formatted": "12 Oct, 2026", "diff": "6 days from now" },
        "quote": null,
        "permissions": { "view": true, "bid": true }
      }
    ],
    "permissions": ["view-facility-procurement-request"]
  }
}
```

## Field Rules

Jobs are the work procurement requests the supplier was invited to. A job counts as live while it
is in `order` or `quotes-review`; the Open Jobs page lists the ones in `order`.

| Field | Meaning |
|---|---|
| `stats.open_jobs` | Live jobs still taking quotes (status `order`, deadline empty or in the future) that the supplier has not quoted. Same number as `jobs_status.available`. |
| `stats.active_quotes` | The supplier's quotes in `bid` or `recommended` on a live job, so still awaiting a decision. |
| `stats.active_contracts` | The supplier's contracts in `active`. |
| `stats.pending_bills` | The supplier's bills in `unpaid` or `partially-paid`. Credit notes are not counted. |
| `contracts_value` | Sum of `amount` on the active contracts, one entry per currency. |
| `outstanding_bills` | Sum of `balance` on `unpaid` and `partially-paid` bills, one entry per currency. Credit notes in those statuses are negative and reduce it. |
| `jobs_status.available` | As `stats.open_jobs`. |
| `jobs_status.quoted` | Live jobs the supplier has quoted. |
| `jobs_status.missed` | Live jobs the supplier did not quote before quoting closed (deadline passed, or the job moved to `quotes-review`). |
| `jobs_status.total` | `available + quoted + missed`. |
| `revenue` | Bill totals per calendar month for the last `months` months, the current month included. Counts bills in `unpaid`, `partially-paid` or `paid`, dated by `invoice_date` (or `created_at` when there is no invoice date). Credit notes net against the month they fall in. Pending and cancelled bills are left out. |
| `revenue.series` | One series per currency the supplier billed in during the period, largest `total` first. Empty when nothing was billed. |
| `latest_open_jobs` | Up to five jobs counted in `stats.open_jobs`, newest first, in the Open Jobs list shape. |
| `permissions` | The supplier's dashboard permissions, cached by the frontend. |

Money is in currency units. A `currency` is `null` for records that carry no currency, and the
frontend shows those in the primary currency.
