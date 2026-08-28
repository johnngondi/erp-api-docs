# Letter of Offer (LOO) — Data Model & Contracts

Domain: `Property Management > Lease Management`

A Letter of Offer is the document a landlord issues to a prospective or
continuing tenant: an **offer** and the **agreement** that follows it, both
drafted from a reusable template.

> **Status: schema and tag resolution.** This describes the data model built in
> Milestone 8 / 14.2 **Ticket 1** and the resolution service built in **Ticket 2**.
> There are still no LOO endpoints — generation, editing, export, send, signature
> and promotion land in Ticket 4, and the approval chain in Ticket 3. Read this to
> know what the shapes will be; do not build against endpoints here, because there
> are none.

## The idea in one paragraph

A **template** holds clause text with `{{tag}}` placeholders in it. Generating a
LOO **copies** that text onto the LOO, so the two are independent from that
moment: editing a LOO never touches its template, and editing a template never
changes a LOO already generated from it. The `{{tags}}` stay in the text as
literal tokens — they are never substituted into it. Resolved values live in a
separate per-LOO store, which is what lets a reviewer correct one value without
touching the clause it appears in. Rendering merges the two at request time.

## Type and status

`type` — `App\Enums\LooType`. Decides where the LOO is prepared from.

| `type` | Prepared from | Colour |
|---|---|---|
| `new lease` | a `LeaseApplication` | `success` |
| `renewal` | the existing `Lease` | `info` |
| `addendum` | the existing `Lease` | `secondary` |

`status` — `App\Enums\LooStatus`. Serialised as `{ "value", "color" }` like every
other status in the API.

| `status` | Meaning | Colour | Editable | Tenant may see |
|---|---|---|---|---|
| `draft` | being written | `secondary` | Yes | No |
| `pending_approval` | in the approval chain | `warning` | Yes | No |
| `approved` | chain cleared, may be exported | `info` | No | Yes |
| `sent` | delivered to the tenant | `primary` | No | Yes |
| `accepted` | signed by the tenant | `success` | No | Yes |
| `declined` | tenant declined | `danger` | No | No |
| `expired` | lapsed unanswered | `danger` | No | No |

"Tenant may see" is enforced server-side by a `visibleToTenant()` scope — nothing
short of a cleared approval chain leaves the staff surface. Export and send are
gated on the same rule.

## Templates

A template carries two independent content blobs, `offer_content` and
`agreement_content`, both `{{tag}}`-bearing HTML.

Scoping is two-dimensional, and `NULL` means "no restriction" on either axis:

| Field | `NULL` means | Set means |
|---|---|---|
| `facility_ids` (json array) | usable at every facility of the company | only those facility ids |
| `facility_type_id` | usable for any space category | only that facility type |

The eligible set for a generation is `active AND facility-usable AND type-usable`.
A company may mark one template `is_default` per `facility_type_id`; setting a new
default clears the previous one.

## The tag registry

`loo_tags` is a **system-wide** catalogue (not per-company) of the tokens a
template author can drag into clause text. 59 tags across nine categories.

| Field | Type | Notes |
|---|---|---|
| `key` | string, unique | The token without braces: `tenant_name` → `{{tenant_name}}` |
| `label` | string | Sidebar label |
| `description` | string \| null | |
| `category` | enum | See below |
| `icon` | string \| null | Icon name |
| `maps_to` | string \| null | Source path for a LOO prepared from an **application** |
| `lease_maps_to` | string \| null | Source path when prepared from a **lease**; `NULL` means `maps_to` serves both |
| `default_format` | string \| null | Seeds the editor's FORMAT picker |
| `is_computed` | boolean | Derived rather than read straight off a column |

Categories (`App\Enums\LooTagCategory`), with how many tags each holds:

| `category` | Label | Tags |
|---|---|---|
| `header` | Header | 2 |
| `parties` | Parties | 5 |
| `property` | Property | 7 |
| `lease_terms` | Lease Terms | 12 |
| `financials` | Financials | 14 |
| `legal` | Legal | 7 |
| `execution` | Execution & Signatures | 5 |
| `banking` | Banking | 3 |
| `system` | System | 4 |

`system` tags (`id`, `type`, `loo_preparation_type`, `loo_preparation_id`) are
internal reference fields, seeded for completeness. They are not expected to be
used in real templates and can reasonably be hidden or collapsed in the sidebar.

### Two source paths per tag

Most tags read the LOO itself or the facility and resolve identically whichever
record the LOO came from — those carry `lease_maps_to: null`. Ten tags differ,
because a renewal has no application to read:

| `key` | `maps_to` (new lease) | `lease_maps_to` (renewal / addendum) |
|---|---|---|
| `tenant_name` | `lease_application.applicant_name` | `lease.user.name` |
| `tenant_address` | `lease_application.applicant_physical_address` | `lease.user.leaseApplications.applicant_physical_address` |
| `guarantors` | `lease_application.guarantors` | `lease.user.leaseApplications.guarantors` |
| `start_date` | `lease_application.proposed_start_date` | `lease.start_at` |
| `facility_type` | `lease_application.facilityType.title` | `lease.facilityType.title` |
| `facility_residential_unit` | `lease_application.residentialUnitTypes` | `lease.leaseItems.facilitySpace.facilityResidentialUnit` |
| `lease_term` | *(computed)* | `lease.period_in_years` |
| `bank` | `lease_application.bankBranch` | `lease.user.leaseApplications.bankBranch` |
| `bank_account_name` | `lease_application.bank_account_name` | `lease.user.leaseApplications.bank_account_name` |
| `bank_account_number` | `lease_application.bank_account_number` | `lease.user.leaseApplications.bank_account_number` |

The tenant of a renewal is **the user holding the lease**. Name and start date
come straight off the lease and its user. Address, guarantors and bank details
have no home on `users`, so they are read back through that user's own
application — see [Open questions](#open-questions).

Not in the registry, deliberately: `fee_note_file_id` is a file attachment rather
than something printed into text, and `lawyer_id` no longer exists as a field.

## Resolved tag values

`loo_tag_values` is the per-LOO store the resolution service writes to.

| Field | Type | Notes |
|---|---|---|
| `tag_key` | string | Soft reference to `loo_tags.key` — the value survives a tag being retired |
| `resolved_value` | text \| null | |
| `is_overridden` | boolean | Set when a reviewer corrected the value by hand |

One row per `(loo_id, tag_key)`. `is_overridden` is what tells a re-resolution
pass the difference between a stale computed value and a deliberate correction.

## Tag resolution

`App\Services\PropertyManagement\Loo\TagResolutionService` turns the registry
into the values above. It is called once at generation, and again whenever the
data underneath a LOO moves.

```php
$service->resolve($loo);                       // read only
$service->resolveAndStore($loo);               // generation
$service->refreshSpaceDerived($loo);           // after a space is added or removed
$service->pendingConfirmation($loo->type);     // what it will not resolve, and why
```

**Where a value comes from** is the LOO's `loo_preparation_type`. A `maps_to` is a
dot path read against four roots — `loo.`, `facility.`, `lease_application.` and
`lease.` — and `LooTag::pathFor()` picks `maps_to` or `lease_maps_to` for you. A
path rooted at the source the LOO was *not* prepared from resolves to `null`
rather than erroring.

**Formats.** Four names are reserved; anything else is read as a PHP date format,
which is what lets the editor's FORMAT picker offer `jS F Y`, `d/m/Y` and the rest
without a fixed list. A per-request override (`['start_date' => 'd/m/Y']`) beats
the tag's `default_format`.

| `format` | Renders as |
|---|---|
| `currency` | `KES 125,000.00` — the **property's** reporting currency, two decimals |
| `yes_no` | `Yes` / `No` |
| `list` | prose, not bullets: `A, B and C` |
| `raw` / `null` | the value as it stands; whole floats lose their decimals, integers never gain separators |
| anything else | a PHP date format |

**The premises tags read `loo_spaces`.** `floor_unit` (`Ground Floor - Shop 4`),
`premises_size`, `service_charge` and `rent_first_period` are all derived from the
spaces the LOO currently grants and the components priced on them — never from
what the applicant originally asked for. `service_charge` is therefore the *live*
figure off `loo_space_components`, as against `monthly_service_charge`, which is
the stored, hand-editable one and has its own tag.

**Overrides survive.** An ordinary re-resolution skips any row with
`is_overridden`, so a reviewer's correction is not undone. `refreshSpaceDerived()`
forces by default and clears the flag, because a recompute re-derives those
figures from the LOO's current spaces — a hand-adjusted figure it replaces has
stopped being current. Pass `force: false` to leave corrections standing.

### What it will not resolve

Tags whose source has not been agreed are reported as **unresolved with a reason**
rather than filled from the nearest plausible column. They still get a
`loo_tag_values` row, with a `null` value, so the editor shows them as fields a
reviewer can fill by hand.

| `key` | Unresolved on | Why |
|---|---|---|
| `rent_review` | both | Nothing on an application or a lease records an escalation basis |
| `landlord_address` | both | Neither `users` nor `landlord_default_accounts` carries an address |
| `lease_term` | `new lease` | An application records no proposed duration; a renewal reads `lease.period_in_years` |
| `tenant_address`, `guarantors`, `bank`, `bank_account_name`, `bank_account_number` | `renewal` / `addendum` | Their `lease_maps_to` routes back through the lease holder's own applications, and which application wins is undecided |

The last row is matched against the registry rather than hard-coded, so the day a
confirmed `lease_maps_to` is seeded those tags start resolving with no code change.

`TagResolution` keeps the two apart deliberately: `values()` are tags that
resolved — a `null` there means the source is genuinely empty — and `unresolved()`
are tags nobody has decided the source for. Collapsing them would make a pending
decision read as an empty field.


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

**Financials** — `monthly_service_charge`, `quarterly_service_charge`,
`annual_service_charge`, `service_charge_security_deposit`,
`rent_security_deposit`, `utility_security_deposit`, `deposit_held`,
`cost_per_parking_slot` (all money, default `0`); `rent_breakdown` (json);
`apply_fitout_period_service_charge` (default `true`),
`rent_and_service_charge_distinct` (default `false`), `has_vat` (default `true`);
`parking_slots_count`, `parking_security_deposit_months` (default `0`).

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

## Open questions

Flagged rather than guessed, and still open before the resolution service is
finished:

- **`tenant_address`, `guarantors` and bank details on a renewal.** `users` has
  no address column and no guarantors of its own, so these are read back through
  the lease holder's own application. If a lease holder has more than one
  application, which one wins is undecided — and an address on `users` may be the
  better answer.
- **`landlord_address`.** Mapped to `facility.landlord.address`, which has nothing
  behind it — `users` has no address column and neither does
  `landlord_default_accounts`. Left unresolved until a source is agreed.
- **`rent_review` and `lease_term` on a new lease.** No column on a lease
  application answers either. Both are left unresolved and filled by hand.
- **How long the first rent period runs.** `rent_first_period` totals the granted
  spaces' rent components, which are monthly figures, so it resolves to one
  month's rent. A landlord billing quarterly wants three — nothing on a LOO states
  the period, and the figure is hand-editable either way.
- **"Suggesting" mode and the "Agent" toolbar button** in the editor designs.
  Neither has any backend scoped. Both need a call on whether they are in this
  milestone, a later one, or frontend-only.

## Related

- [Lease Applications API](../lease-applications.md) — `proposed_start_date` and
  the applicant bank details that feed the tags above
- [Leases API](../leases.md) — what a `new lease` LOO is promoted into
