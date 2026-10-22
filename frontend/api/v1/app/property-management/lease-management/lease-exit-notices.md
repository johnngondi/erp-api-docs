# Lease Exit Notices API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/exit-notices`

These endpoints allow staff users to manage all lease exit notices across leases.

## Endpoints

- `GET /exit-notices`
- `GET /exit-notices/quote` - a tenant's leases on a property, with their figures (row 148)
- `POST /exit-notices`
- `GET /exit-notices/{tenantExitNotice}`
- `PUT/PATCH /exit-notices/{tenantExitNotice}`
- `DELETE /exit-notices/{tenantExitNotice}`

## List Exit Notices

`GET /api/v1/app/{company}/property-management/lease-management/exit-notices`

Supported query params:

- Filters:
  - `filter[search]`
  - `filter[id]`
  - `filter[lease_id]` (any lease the notice covers, not only its first - row 148)
  - `filter[initiated_by_id]`
  - `filter[exit_date]`
  - `filter[status]`
  - `filter[created_at]`
  - `filter[tenant_id]` (mapped to `lease.user_id`)
  - `filter[facility_id]` (mapped to `lease.facility_id`)
  - `filter[lease.user_id]`
  - `filter[lease.facility_id]`
- Sort:
  - `sort=id,lease_id,exit_date,created_at,updated_at,deposit_held,arrears,repair_damages_cost`
- Pagination:
  - `per_page`
  - `page`

Examples:

- `GET /api/v1/app/12/property-management/lease-management/exit-notices?filter[lease.user_id]=77`
- `GET /api/v1/app/12/property-management/lease-management/exit-notices?filter[search]=vacating&sort=-created_at`

## One notice per tenant (row 148)

A notice is raised for **a tenant on a property** and covers the lease picked, or all their leases
there. Each lease is an **item** with its own reason, damages and documents; the notice has one exit
date, one "clear arrears from deposit" switch and **one approval** - rejecting it rejects all its
leases. On approval each lease is closed exactly as before (terminated, arrears cleared if the switch
is on, its deposit refund bill raised, its deposits marked refunded). The money flow is unchanged.

The screen: pick the property, then the tenant (`GET /users/tenants?filter[facility_id]=…`), the exit
date and optionally one lease; then an **Overall** tab and one tab per lease.

### Quote: the tenant's leases and their figures

`GET /exit-notices/quote?facility_id=…&tenant_id=…[&lease_id=…][&auto_credit_arrears=1]`

Writes nothing. Lists the tenant's active and suspended leases on the property (or the one
`lease_id`):

```json
{
  "data": {
    "leases": [
      { "lease_id": 91, "spaces": ["A-3-07"], "status": { "value": "active", "color": "success" },
        "deposit_held": 30000, "arrears": 5000, "arrears_cleared": 0, "open_exit_notice_id": null }
    ],
    "overall": { "leases_count": 1, "deposit_held": 30000, "arrears": 5000, "arrears_cleared": 0 }
  }
}
```

- `open_exit_notice_id` is set when the lease is already on a notice awaiting approval: it cannot go
  on another, so show it disabled.
- `arrears_cleared` is what the deposit would clear if `auto_credit_arrears` is on.
- **What the tenant gets** on a lease: `deposit_held - arrears - repair_damages_cost`, never below 0.
  Work it out on the form as damages are entered; the notice returns it once saved.

### Initiate for a tenant

`POST /exit-notices` with `leases`:

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_id` | Yes | integer | The property |
| `tenant_id` | Yes | integer | The tenant (`users.id`) |
| `exit_date` | Yes | date | One exit date for the notice |
| `auto_credit_arrears` | No | boolean | One switch for the notice, applied lease by lease |
| `leases` | Yes | array | The lease picked, or all the tenant's leases on the property; at least one |
| `leases.*.lease_id` | Yes | integer | Each once; the tenant's, on this property, active or suspended, not on another open notice |
| `leases.*.reason_for_exit` | Yes | string | |
| `leases.*.repair_damages_cost` | No | number | Min 0 |
| `leases.*.supporting_upload_id` | No | integer | |
| `leases.*.repairs_cost_quote_upload_id` | No | integer | |

A lease that fails a check answers `422` on `leases.{i}.lease_id` with the reason (not this tenant's
lease on this property; ended; already on exit notice #N awaiting approval).

### The notice's figures

Every notice response carries:

- `items[]`: one per lease - `lease_id`, `lease`, `deposit_held`, `arrears`, `arrears_cleared`,
  `repair_damages_cost`, `tenant_gets`, `reason_for_exit`, `supporting_upload(_id)`,
  `repairs_cost_quote_upload(_id)`. `id` is `null` on a notice raised before row 148, which reads as
  one item made from the notice itself.
- `overall`: `leases_count`, `deposit_held`, `arrears`, `arrears_cleared`, `repair_damages_cost`,
  `tenant_gets` - the Overall tab.

The notice's own `deposit_held`, `arrears`, `repair_damages_cost` and `reason_for_exit` are its items'
totals (reasons joined); `lease` is its first lease. A notice with one lease still carries that lease's
documents on itself, as before.

## Initiate Exit Notice (one lease)

The single-lease form below still works and makes a notice with one item.

`POST /api/v1/app/{company}/property-management/lease-management/exit-notices`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `lease_id` | Yes | integer | Must exist in `leases.id` |
| `exit_date` | Yes | date string | Format: `YYYY-MM-DD` |
| `reason_for_exit` | Yes | string | Exit reason |
| `supporting_upload_id` | No | integer | Must exist in `uploads.id` |
| `auto_credit_arrears` | No | boolean | Defaults to `false`. See "Auto-credit arrears" below |
| `repair_damages_cost` | No | number | Min `0`. Repair or damage cost deducted from the refund |
| `repairs_cost_quote_upload_id` | No | integer | Must exist in `uploads.id`. The quote backing the repair cost |

Notes:

- `deposit_held` and `arrears` are computed automatically. They are the lease's
  `total_deposit_amount` and `current_arrears` at the moment of initiation, so a form can preview
  them from `GET leases/{lease}` before submitting.
- The app shows amount to refund as `deposit_held` - `arrears` - `repair_damages_cost`, floored
  at `0`. That is the tenant's share, which management pays out manually (by cheque). Processing
  posts the whole deposit not offset against arrears (the deposit less any arrears credited when
  `auto_credit_arrears` is `true`) to landlord expenses as one deposit refund bill. Repairs are part
  of that expense, so `repair_damages_cost` is not deducted from the bill.
- The tenant portal does not accept the repair fields on create. Staff set them.
- Anyone who can view the lease can initiate its exit notice: the `view-lease` permission plus an
  allocation to the lease's property. `update-lease` is not required. A lease on a property the
  user is not allocated to returns `403` with `You are not assigned to this property.`

### Auto-credit arrears

When `auto_credit_arrears` is `true`, processing the exit notice settles the
tenant's outstanding arrears out of the deposit held before refunding it:

1. The amount credited is `min(total outstanding invoice balances, deposit held)`
   — the deposit funds the credit, so it can never credit more than is held.
2. Credit notes are raised against the tenant's unpaid / partially paid invoices
   (oldest first), each capped at the invoice balance, until the deposit-funded
   budget is used up or the invoices run out.
3. The deposit refund bill/expense is then posted for the **remaining** deposit
   (deposit held minus the amount credited). When the arrears consume the whole
   deposit, no refund is posted.

When `false` (the default), behaviour is unchanged: the full deposit is refunded
and arrears are handled separately.

## Update Exit Notice

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/exit-notices/{tenantExitNotice}`

Request body may include:

- `exit_date`
- `reason_for_exit`
- `supporting_upload_id`
- `repair_damages_cost`
- `repairs_cost_quote_upload_id`
- `auto_credit_arrears`

`repair_damages_cost` and `repairs_cost_quote_upload_id` are intended for staff-side evaluation updates.

**Per lease (row 148):** send a lease's reason, damages and documents under `leases`:

```json
{ "exit_date": "2026-11-30", "leases": [ { "lease_id": 92, "repair_damages_cost": 2500, "reason_for_exit": "Updated" } ] }
```

Each `leases.*.lease_id` must be on the notice (`422` otherwise). The top-level `reason_for_exit`,
`supporting_upload_id`, `repair_damages_cost` and `repairs_cost_quote_upload_id` still work on a notice
with one lease; on a notice with several they answer `422` on `leases` - send them per lease. Every
edit re-reads each lease's deposit held and arrears exactly as create does: deposit held is the lease's
full deposit, whether or not its deposit invoice is paid (an unpaid deposit invoice shows in the
arrears, so it cancels out), and arrears are the lease's balance (billed less collected).

## Delete Exit Notice

`DELETE /api/v1/app/{company}/property-management/lease-management/exit-notices/{tenantExitNotice}`

Deletes the notice, its items and associated approval artifacts, and releases every lease it covered.
