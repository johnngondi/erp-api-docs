# The LOO Document — Data Model & Contracts

Domain: `Property Management > Lease Management`

What a generated Letter of Offer holds, what it costs, and the approval chain it
climbs. For the clause templates it is drafted from see
[loo-templates.md](./loo-templates.md); for the tag registry and the status
lifecycle see [README.md](./README.md).

> **Status: schema, rent schedule and approval.** There are still no LOO
> endpoints — generation, editing, export, send, signature and promotion land in
> Ticket 4. Read this to know what the shapes will be; do not build against
> endpoints here, because there are none.

## Granted spaces

`loo_spaces` is the authoritative record of what the LOO actually covers — not
the originating request. It is copied from
`lease_application_facility_space` on generation (or from the lease's
`lease_items` for a renewal), and becomes `lease_items` again on promotion.

It holds a plain `facility_space_id`, **not** a polymorphic target. A residential
request is stated as unit types and quantities, but it is resolved to real
`facility_spaces` rows when the application is submitted, so by the time a LOO
exists both commercial and residential grants are already spaces.

`loo_space_components` prices each granted space per component, and mirrors
`lease_application_facility_space_components` column for column:

| Field | Type | Notes |
|---|---|---|
| `lease_component_id` | integer | Rent, service charge, parking fee, … |
| `cost_per_space_unit` | decimal(50,5) | Net of tax |
| `amount` | decimal(50,5) | `cost_per_space_unit` × the space's size, computed server-side |
| `tax_id` | integer \| null | Defaults to the component's own tax, overridable per space |

This is the editable pricing surface: staff adjust components here, on the LOO,
without touching the application they were copied from.

## The rent schedule

What the tenant pays, period by period, from the first period to the end of the term.
A **period** runs from one escalation to the next: the first is everything before any
escalation bites, and each one after it is the same figure escalated by whatever rates
the schedule states.

Built by `App\Services\PropertyManagement\Loo\RentScheduleBuilder` and **stored** on
the LOO by `RecomputeLooRentScheduleAction`, which runs at generation and whenever an
input moves. The tags read the stored schedule, not a fresh computation — what an offer
prints should be what somebody approved, and a reviewer can correct a period by hand.
A LOO that has never had one computed falls back to building it live.

Three inputs, each with an application-side and a lease-side source, so one builder
serves a `new lease` LOO and a renewal:

| | prepared from an application | prepared from a lease |
|---|---|---|
| start | `proposed_start_date` | `lease.start_at` |
| term | LOO term, then `proposed_period_in_*` | LOO term, then the lease's own period |
| escalations | `lease_application_escalations` | `lease_escalations` |

Amounts come off `loo_space_components` and nowhere else — rent, service charge,
parking and signage alike, since all four reach a LOO the same way. Tax is the row's
own `tax_id`: an explicit `NULL` means the space was priced tax-free on purpose, and
inheriting the component default would quietly overrule that.

Escalation is **compounding** — 10% twice is ×1.21, not ×1.2 — matching what
`ProcessLeaseEscalationJob` does to a live lease. Where two components escalate on
different cycles the term is cut on the **union** of their dates, so a period boundary
exists wherever any figure changes.

### Stored shape

`loos.rent_breakdown`, one entry per period. Each row carries a `label`/`amount` pair
as well as its structured figures — that pair is what `{{rent_breakdown}}` renders as
prose, so the list needs no special casing.

```json
[{
  "period": 1,
  "label": "Year 1 (1st October 2026 - 30th September 2027)",
  "amount": 4896000,
  "starts_at": "2026-10-01", "ends_at": "2027-09-30", "months": 12,
  "monthly": 408000, "quarterly": 1224000, "annual": 4896000,
  "net": 4320000, "tax": 576000, "total": 4896000,
  "components": [
    {"lease_component_id": 3, "name": "Rent", "monthly_net": 300000, "monthly_tax": 48000, "monthly_gross": 348000}
  ]
}]
```

`ends_at` is the last day the period covers, not the first day of the next one — a
clause reads "to 30th September", never "to 1st October".

### The rent tags

All five read the stored schedule. The four first-period figures come off row 1.

| Tag | Is |
|---|---|
| `rent_first_period` | everything payable across the whole first period |
| `rent_first_period_monthly` | the same, per month |
| `rent_first_period_quarterly` | per quarter |
| `rent_first_period_annual` | per annum |
| `rent_breakdown` | every period, as prose |

All are tax-inclusive and cover rent, service charge, parking and signage together.
`service_charge` is unchanged and still means something different: service-charge
components only, net of tax, live off `loo_space_components`.

`lease_term` resolves too, now that both sides carry a term — the LOO's own, falling
back to the applicant's proposed period, or the lease's on a renewal. It renders
"6 years", "18 months", "6 years 6 months".

### Parking

`loos` carries no `parking_slots_count` or `cost_per_parking_slot`. A slot is a
`facility_space` flagged `is_parking`, priced through a parking-fee component and
granted through `loo_spaces` — the scalars restated that in a second place that could
disagree with it. Both tags survive, recomputed: the count is how many granted spaces
are parking, and the per-slot cost is the parking-fee total over that count.
`parking_security_deposit_months` stays, because it is a term rather than a price.

## `our_ref`

Assembled at issue time and then held on the LOO:

```
{company initials}/{facility initials}/{month year}/{loo id}/{generated by}
```

for example `PE/TRM/08 2026/41/7`.

Initials are **derived from the two names**, not stored — neither companies nor
facilities carry an initials column. Multi-word names contribute one letter per
word (`Two Rivers Mall` → `TRM`), legal suffixes and connectors are dropped
(`Property Experts Limited` → `PE`), and a single-word name contributes its first
three letters (`Britam` → `BRI`).

Because the reference is stamped onto the record when the LOO is issued, and the
LOO holds its own `facility_id`, a later rename or a re-pointed application
cannot change a reference already in a tenant's hands.

## LOO fields

Every field below is staff-editable, **including the computed ones** — a figure
recomputed from the granted spaces can still be hand-adjusted afterwards. Money
is `decimal(50,5)` in currency units, matching the rest of the system.

**Term** — `lease_term_years`, `lease_term_months`. Copied from the application's
proposed period at generation; the LOO's own value wins wherever it is set.

**Financials** — `monthly_service_charge`, `quarterly_service_charge`,
`annual_service_charge`, `service_charge_security_deposit`,
`rent_security_deposit`, `utility_security_deposit`, `deposit_held` (all money,
default `0`); `rent_breakdown` (json, the stored rent schedule);
`apply_fitout_period_service_charge` (default `true`),
`rent_and_service_charge_distinct` (default `false`), `has_vat` (default `true`);
`parking_security_deposit_months` (default `0`).

**Lease terms** — `quarterly_rent_due_dates` (default
`1st January, 1st April, 1st July and 1st October`), `rent_due_day_of_month`
(`5th`), `late_payment_grace_days` (`14`), `redecoration_notice_days` (`7`),
`rent_deposit_months` (`1`), `pet_license_revocation_notice_days` (`30`),
`termination_notice_months` (`3`), `termination_notice_rent_in_lieu_months` (`3`).

The first three are strings, not numbers: the clause prints them verbatim.

**Execution** — `landlord_execution_deadline_days` (`21`),
`tenant_execution_deadline_days` (`14`), `has_landlord_signature` (`false`),
`signatory_type` (`landlord` | `trustees`, default `landlord`), `trustees_count`
(`0`, only meaningful when the signatory type is `trustees`).

**Legal** — `has_lawyer` (`false`), `submitted_to_lawyer_at`, `legal_fees`,
`disbursement_fee`, `stamp_duty_fee`, `legal_fee_updated_at`,
`legal_fee_recorder_id`, `fee_note_file_id` (an upload; not a tag).


## Approval

A LOO goes through the **generic approval framework**, like every other approvable
resource. It is registered in `config('approvals.models')`, uses the `Approvable`
trait, and is driven by `ApprovalService`.

It used to have a stack of its own, because a LOO step has to be skippable *for
some documents and not others* — "residential under 50,000 a month does not need
the CEO" — and `approval_template_steps` had nowhere to hang that. That gap is now
closed on the shared table (`conditions`), so there is nothing LOO-specific left in
the mechanism. The condition shape, operators and modes are documented once, in
[approval-templates.md](../../../access-management/approval-templates.md#step-conditions).

What *is* LOO-specific is the facts its conditions may read, and how its statuses
move.

### Condition facts

Beyond every scalar column on `loos`, a LOO exposes these derived facts:

| Fact | Meaning |
|---|---|
| `monthly_rent` | what the tenancy costs a month at the start of the term, **tax included** |
| `quarterly_rent`, `annual_rent` | the same figure × 3 and × 12 |
| `facility_type` | the type's title (`Residential`, `Commercial`, …) |
| `facility_type_id`, `facility_id` | ids, for an `in` against a set |
| `loo_type` | `new lease` \| `renewal` \| `addendum` |
| `lease_term_months` | the term the LOO states, falling back to the one proposed |
| `space_count` | how many spaces the LOO grants |

`monthly_rent` is read the same way `{{rent_first_period_monthly}}` is printed:
the **stored** rent schedule's first period wins, falling back to building it,
then to the priced components. A threshold should be measured against the figure
on the document somebody is being asked to approve, not one re-derived from rows
that may have moved since it was drafted.

`facility_type` is read off the record the LOO was prepared from, **not** the
facility. An applicant picks a type inside a mixed-use facility, and the
facility's own type would answer "mixed use" to a rule asking about residential.

A LOO with nothing priced yet has **no** `monthly_rent`, and a null never matches
any operator — so an unfinished document cannot slip under a threshold.

The canonical example, hung on the CEO step:

```json
{
  "mode": "bypass",
  "match": "all",
  "rules": [
    { "field": "monthly_rent",  "operator": "less_than", "value": 50000 },
    { "field": "facility_type", "operator": "equals",    "value": "Residential" }
  ]
}
```

### Status transitions

| Action | The LOO |
|---|---|
| `initiate` | `draft` → `pending_approval` (or straight to `approved` if no step applies) |
| `approve`, not the last step | stays `pending_approval` |
| `approve`, the last step | → `approved`, and `LooApprovedEvent` fires |
| `review` | stays `pending_approval`; the previous step is re-opened |
| `reject` | → `draft` |

A LOO **does not** enter a chain when it is created. It is drafted first and
submitted deliberately — `shouldAutoInitiateApproval()` returns `false`, and the
submit endpoint (Ticket 4) calls `ApprovalService::initiate()`.

Rejection returns the LOO to `draft` rather than to a terminal state: an offer
that has not left the building is rejected so somebody corrects it and sends it
round again. Re-submitting opens a new **`attempt`**, so the rejected chain stays
on the record beside the new one — and the chain is re-evaluated against the LOO
as it now stands, so a figure corrected after a rejection can change which steps
apply.

Export, send, and everything the tenant portal is allowed to see are gated on
`approvalChainCleared()`, which reads the **status** rather than the steps — a LOO
approved because no step applied to it has no chain at all, and is no less
approved for that.

## Related

- [LOO templates](./loo-templates.md) — the clause text a LOO is drafted from
- [README](./README.md) — the tag registry, resolution, and the status lifecycle
- [Approval templates](../../../access-management/approval-templates.md) — step conditions and the edit grant
- [Leases API](../leases.md) — what a `new lease` LOO is promoted into
