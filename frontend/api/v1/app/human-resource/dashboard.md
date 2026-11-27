# Human Resource Dashboards (Frontend)

Two dashboards: the Admin one for HR and supervisors, the Staff one for every employee.

Both return the **full response contract now**. A tile that depends on a later ticket is present
with the value `null` until that ticket fills it. Render a `null` tile as "not available yet" (or
hide it), never as zero.

| Tile | Filled by |
| --- | --- |
| Employees on leave, annual leave left, quick updates | Leave tickets |
| Checked in today, attendance score, today's card | Attendance tickets |
| Performance, performance trend, job posts | Not in this batch (stay `null`) |

## Admin dashboard

`GET /api/v1/app/{company}/human-resource/admin`

Permission: `view-human-resource-dashboard` (`403` without it).

### Query parameters

| Param | Default | Description |
| --- | --- | --- |
| `period` | `this_quarter` | The calendar period Workforce Movement covers. One of the values below. Anything else is a `422` on `period`. |

| `period` | Covers |
| --- | --- |
| `this_month` | 1st of this month → today |
| `this_quarter` | 1st of the current quarter → today |
| `last_quarter` | the whole previous quarter |
| `this_year` | 1 January → today |
| `last_year` | the whole previous year |

Dates are the company's own (its timezone). A period still running ends today; no month after
today is returned.

**Only Workforce Movement follows the period.** `employees` (total, its split and the change this
month) is always as of today, and the leave and attendance tiles will be today's figures.

The old `months` parameter is gone; it is ignored if sent.

### Response

```json
{
  "data": {
    "period": {
      "value": "this_quarter",
      "label": "This quarter",
      "from": "2026-10-01",
      "to": "2026-11-25"
    },
    "employees": {
      "total": 130,
      "active": 118,
      "on_probation": 9,
      "suspended": 2,
      "leaving": 1,
      "starting": 3,
      "change_this_month": {
        "count": 6,
        "percent": 4.8,
        "previous_total": 124
      }
    },
    "performance": null,
    "employees_on_leave": null,
    "checked_in_today": null,
    "job_posts": null,
    "performance_trend": null,
    "workforce_movement": {
      "from": "2026-10-01",
      "to": "2026-11-25",
      "months": [
        { "month": "2026-10", "label": "Oct 2026", "joins": 4, "exits": 2 },
        { "month": "2026-11", "label": "Nov 2026", "joins": 6, "exits": 1 }
      ],
      "total_joins": 10,
      "total_exits": 3
    },
    "quick_updates": null,
    "permissions": ["view-human-resource-dashboard", "view-human-resource-setting", "impersonate-employee"]
  }
}
```

### Fields

- `period`: the period answered, ready to label the selector. `from` and `to` are dates
  (`Y-m-d`); `to` is never after today.
- `employees.total`: headcount today. Everyone who has joined and not yet left. Drafts are never
  counted, and someone hired with a join date still ahead is not counted yet.
- `employees.active`, `on_probation`, `suspended`, `leaving`: the same people split by status, so
  they always add up to `total`. `leaving` is someone terminated whose last working day is still
  ahead.
- `employees.starting`: hired, with a join date still ahead. Not part of `total`.
- `employees.change_this_month`: headcount today against headcount at the end of last month.
  `count` can be negative. `percent` has one decimal (a whole number comes as e.g. `20`) and is
  `null` when there was nobody at the end of last month. Use it for the "+5.0% this month" line.
- `workforce_movement.months`: one row per calendar month in the period, oldest first, including
  months with no movement. `joins` counts employees whose join date falls in the period and the
  month; `exits` counts terminated employees whose last working date (or termination date) does.
  `label` is ready for the axis. `from` and `to` repeat the period.
- `permissions`: the caller's Human Resource permissions, the same list as
  `GET /me/permissions` → `modules["Human Resource"]`.

## Staff dashboard

`GET /api/v1/app/{company}/human-resource/me`

No permission. The caller must be an employee on probation or active (see `README.md`). With the
`X-Acting-Employee` header it answers for the employee acted for.

### Response

```json
{
  "data": {
    "employee": {
      "id": 45,
      "staff_no": "NWR-0045",
      "name": "Jane Wanjiku Mwangi",
      "first_name": "Jane",
      "job_title": "Accountant",
      "status": { "value": "active", "color": "success", "label": "Active" }
    },
    "acting_on_behalf": false,
    "performance": null,
    "attendance_score": null,
    "annual_leave": null,
    "today": null,
    "performance_trend": null,
    "quick_updates": null
  }
}
```

- `employee`: who the staff space is showing. Use `first_name` for "Welcome back, Jane".
- `acting_on_behalf`: `true` when the caller sent `X-Acting-Employee`. Show a banner naming
  `employee.name` while it is `true`.
