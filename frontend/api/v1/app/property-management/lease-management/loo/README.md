# Letter of Offer (LOO) — Data Model & Contracts

Domain: `Property Management > Lease Management`

A Letter of Offer is the document a landlord issues to a prospective or
continuing tenant: an **offer** and the **agreement** that follows it, both
drafted from a reusable template.

> **Status: complete and callable.** The data model (**Ticket 1**), the
> resolution service (**Ticket 2**), the escalation-driven rent schedule, the
> approval wiring (**Ticket 3** — which folded the LOO into the generic approval
> framework rather than giving it a stack of its own) and the full workflow
> surface (**Ticket 4** — generation, editing, spaces, export, send, signature
> and promotion) are all built. Endpoints are documented in
> [loo.md](./loo.md#endpoints) and [loo-templates.md](./loo-templates.md#endpoints).

## The three documents

| File | Covers |
|---|---|
| **README.md** (this file) | The shared contract: type and status, the tag registry, resolved tag values, tag resolution |
| [loo-templates.md](./loo-templates.md) | `LooTemplate` — clause text, scoping, defaults |
| [loo.md](./loo.md) | The generated LOO — granted spaces, rent schedule, `our_ref`, fields, approval |

## Where the endpoints live

All under `api/v1/app/{company}/property-management/`.

| Surface | Base | Documented in |
|---|---|---|
| Generating an offer | `lease-management/lease-applications/{application}/loos`, `lease-management/leases/{lease}/loos` | [loo.md](./loo.md#endpoints) |
| The offer itself | `lease-management/loos/{loo}` | [loo.md](./loo.md#endpoints) |
| Granted spaces | `lease-management/loos/{loo}/spaces` | [loo.md](./loo.md#endpoints) |
| Tag registry | `lease-management/loo-tags` | [below](#the-tag-registry) |
| Clause templates | `settings/loo-templates` | [loo-templates.md](./loo-templates.md#endpoints) |
| Approval templates | `access-management/approval-templates` | [approval-templates.md](../../../access-management/approval-templates.md) |
| Tenant portal | `api/v1/tenant/loos` | [loo.md](./loo.md#the-tenant-portal) |

**A generated LOO is addressed on its own, not under the record it came from.**
Generation is the only step that needs to know the source, and it has two
endpoints for that reason — an application for a `new lease`, a lease for a
`renewal` or `addendum`. Everything after it is a `{loo}`. To list the offers on
one application or one lease, filter the index on `loo_preparation_type` +
`loo_preparation_id`.

**There is no `loo-approval-templates` endpoint, and that is deliberate.** A LOO
goes through the generic approval framework, so its approval templates are the
ordinary ones, with `App\Models\Loo` as the model type and the per-step
`conditions` carrying the bypass rules. A second CRUD surface would be the
parallel stack Ticket 3 deleted, coming back.

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

Templates are covered in [loo-templates.md](./loo-templates.md); the generated
document in [loo.md](./loo.md).

## The tag registry

`loo_tags` is a **system-wide** catalogue (not per-company) of the tokens a
template author can drag into clause text. 62 tags across nine categories.

```http
GET lease-management/loo-tags
    ?filter[category]=financials&filter[label]=rent
```

Unpaginated — it is a fixed catalogue the editor's sidebar renders whole, and
paging it would only make the client reassemble something the server already has
entire. Each entry carries `token` (`{{tenant_name}}`) alongside `key`, so the
brace convention lives in one place. Authorised by `view-loo-template`: whoever
may read the templates may read the vocabulary they are written in.

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
| `financials` | Financials | 17 |
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
| `lease_term` | `loo.lease_term_years` | `lease.period_in_years` |
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
| `rent_review` | both | The escalation *schedule* is recorded, but nothing records what the clause should say about the review basis |
| `landlord_address` | both | Neither `users` nor `landlord_default_accounts` carries an address |
| `tenant_address`, `guarantors`, `bank`, `bank_account_name`, `bank_account_number` | `renewal` / `addendum` | Their `lease_maps_to` routes back through the lease holder's own applications, and which application wins is undecided |

The last row is matched against the registry rather than hard-coded, so the day a
confirmed `lease_maps_to` is seeded those tags start resolving with no code change.

`TagResolution` keeps the two apart deliberately: `values()` are tags that
resolved — a `null` there means the source is genuinely empty — and `unresolved()`
are tags nobody has decided the source for. Collapsing them would make a pending
decision read as an empty field.


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
- **`rent_review`.** No column on a lease application or a lease records an
  escalation or rent-review *basis* in prose. The schedule itself is now recorded
  (`lease_application_escalations`), but what the clause should say about it is not,
  so the tag is left unresolved and filled by hand.
- **"Suggesting" mode and the "Agent" toolbar button** in the editor designs.
  Neither has any backend scoped, and Ticket 4 deliberately did not build one —
  storing suggested edits and an AI clause-drafting endpoint are both real
  features, not polish. Both still need a call on whether they are in this
  milestone, a later one, or frontend-only.
- **What moves a LOO to `expired`.** The status exists and the tenant-visibility
  rules account for it, but nothing sets it: no column records how long an offer
  stands, and no job lapses one. An offer validity period is the missing input.

## Related

- [LOO templates](./loo-templates.md) — the clause text a LOO is drafted from
- [LOO document](./loo.md) — granted spaces, the rent schedule, and approval
- [Lease Applications API](../lease-applications.md) — `proposed_start_date` and
  the applicant bank details that feed the tags above
- [Leases API](../leases.md) — what a `new lease` LOO is promoted into
