# The LOO Document — Data Model & Contracts

Domain: `Property Management > Lease Management`

What a generated Letter of Offer holds, what it costs, and the approval chain it
climbs. For the clause templates it is drafted from see
[loo-templates.md](./loo-templates.md); for the tag registry and the status
lifecycle see [README.md](./README.md).

> **Status: complete and callable.** Schema, rent schedule, approval, the review
> conversation and the full workflow surface — generation, editing, spaces, export,
> send, signature and promotion — are all built. See [Endpoints](#endpoints).
>
> One thing is deliberately **not** built: promoting a `renewal` or `addendum`.
> That path amends the lease it was prepared from, and exactly what it amends was
> never scoped, so the endpoint refuses it by name rather than guessing. Nothing
> above the action assumes the source is an application — see
> [Renewals and addenda](#renewals-and-addenda).

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

A tax's `value` is a **fraction**, as everywhere else in billing: `0.16` is 16%. A
component's monthly tax is `amount × value`. Until 7 October 2026 the schedule divided by
100 again and charged 16% VAT as 0.16%. Offers drafted before then carry the
under-taxed schedule until they are recomputed (any edit to a draft or pending offer
does it, or `php artisan loos:recompute-rent-schedules`) or, once sent, [rebased](#rebase).

Escalation is **compounding** — 10% twice is ×1.21, not ×1.2 — matching what
`ProcessLeaseEscalationJob` does to a live lease. Where two components escalate on
different cycles the term is cut on the **union** of their dates, so a period boundary
exists wherever any figure changes.

### Stored shape

`loos.rent_breakdown`, one entry per period. Each row carries a `label`/`amount` pair
as well as its structured figures. `{{rent_breakdown}}` prints the `components` of every
row as a table (see [the rent tags](#the-rent-tags)); the shape itself is unchanged and is
what the editor shows for hand corrections.

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
| `rent_breakdown` | every period, as an HTML table: one row per component, per month / per quarter / per annum |

All are tax-inclusive and cover rent, service charge, parking and signage together.

**`rent_breakdown` is a table, not prose.** For each period it prints a header row with
the period label, then one row per component (across all granted spaces together, never
per space) with the component's figure per month, per quarter (×3) and per annum (×12),
tax included, escalated as the period states, and a **Total** row from the period's own
`monthly` / `quarterly` / `annual`. A hand-corrected period that carries no `components`
prints only its Total row. The markup is:

```html
<table class="rent-breakdown">
  <thead><tr><th>Component</th><th>Per month</th><th>Per quarter</th><th>Per annum</th></tr></thead>
  <tbody>
    <tr class="period"><th colspan="4">Year 1 (1st October 2026 - 30th September 2027)</th></tr>
    <tr><td>Rent</td><td>KES 348,000.00</td><td>KES 1,044,000.00</td><td>KES 4,176,000.00</td></tr>
    <tr><td>Service Charge</td><td>KES 60,000.00</td><td>KES 180,000.00</td><td>KES 720,000.00</td></tr>
    <tr class="total"><td>Total</td><td>KES 408,000.00</td><td>KES 1,224,000.00</td><td>KES 4,896,000.00</td></tr>
    <tr class="period"><th colspan="4">Year 2 (…)</th></tr>
    ...
  </tbody>
</table>
```

The tag's `default_format` is `html` (see the formats table in the [README](README.md)):
the editor inserts its `resolved_value` as markup, and the PDF styles `table.rent-breakdown`
(collapsed borders, right-aligned figures, bold period and total rows). A reviewer who
corrects a figure sends the table back through `tag_values`; it is sanitised to table,
paragraph and emphasis elements and the `class`/`colspan` attributes, so a script or a link
pasted into it never reaches the document. `rent_breakdown_monthly` is unchanged and still
prints prose.
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

## Deposits

Three figures, all **net of tax**, all re-derived whenever the granted spaces change:

- **`rent_deposit_months`** starts as the property's `deposit_months` (the facility setting,
  default 3), copied at generation. A reviewer may change it on the offer, and the two
  deposits follow (see [Editing](#editing)).
- **`rent_security_deposit`** is `rent_deposit_months` × the rent per month of the **final
  period** of the rent schedule: the escalated figure in the last row of `rent_breakdown`,
  summed over the rent components across every granted space. The deposit covers the
  tenancy at the rent it ends on, not the rent it starts on.
- **`service_charge_security_deposit`** is the same months × the final period's service
  charge per month.

When the schedule cannot be built yet (no term or no start date) both fall back to the
first-period component totals. `utility_security_deposit` is never derived: utilities are
not priced through `loo_spaces`.

**`deposit_held`**, on a `renewal` offer, is prefilled with what the landlord already holds
on the lease being renewed: the sum of that lease's deposits that have not been refunded
(`total_deposit_amount` on the lease, the same figure the lease page shows), billed or not.
It is re-read whenever the spaces change, so a hand edit to it is replaced then, like every
other derived deposit figure. On a `new lease` or an `addendum` the column is left as staff
enter it. The lease it was read from, with its deposit rows, is on the offer response as
[`renewed_lease`](#renewed-lease).

**`{{deposit_to_pay}}`** is the top-up: `rent_security_deposit` +
`service_charge_security_deposit` + `utility_security_deposit` − `deposit_held`, floored at
zero. A renewal whose held deposit already covers the new figures prints `KES 0.00`.

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

## An offer needs something to cover

Generation is refused with `422` and a `spaces` key when the record it is drafted from
covers no units:

```json
{ "errors": { "spaces": ["An offer can only be drafted once spaces have been allocated."] } }
```

`permissions.generateLoo` carries the same rule, so the action disappears rather than
failing when pressed. It is now `generate-loo` **and** an approved application **and** a
non-empty allocation.

**Expect this on applications approved before the rule existed.** Approval now requires an
allocation, so only older records can reach it — and an approved application cannot be
reopened, so Generate LOO stays unavailable on them for good. That is the rule working,
not a fault. Show the message as it comes; there is no action the user can take.

A renewal is checked against the items on the lease being renewed, since that is what it
copies.

## Who an offer is for, and what it was generated from

Two fields on the **staff** list and the staff single offer, for the columns beside the
reference:

```json
"loo_template": { "id": 4, "name": "Standard Retail Offer" },
"prepared_for": { "type": "lease_application", "id": 24, "name": "Safaricom PLC" }
```

`loo_template` is the template the offer was generated from. Deliberately a link rather
than the full template — that resource carries `offer_content` and `agreement_content`,
so a page of 25 offers would ship 25 copies of two clause documents to print one line.
Call the templates endpoint when somebody opens one.

`prepared_for` is the counterparty, read off whatever the offer was prepared from:

| `type` | Taken from | Who that is |
|---|---|---|
| `lease_application` | the application's `applicant_name` | the applicant |
| `lease` | the lease's tenant | the tenant |

**A `new lease` offer has no tenant**, which is why a column reading only tenants comes up
empty on most of the list. The counterparty is still an applicant until the lease is
raised. `type` is sent so the column can be labelled per row rather than guessed, and `id`
gives you something to link to.

`name` falls back to the linked user when an application carries no name of its own.

**Neither field appears on the tenant portal**, which serves the same resource. The
template an offer was built from is internal.

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
`rent_deposit_months` (the property's `deposit_months` at generation; see
[Deposits](#deposits)), `pet_license_revocation_notice_days` (`30`),
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
| `reject` | → `rejected` (final) |

A LOO **does not** enter a chain when it is created. It is drafted first and
submitted deliberately — `shouldAutoInitiateApproval()` returns `false`, and
`POST loos/{loo}/submit` calls `ApprovalService::initiate()`.

Rejection is **terminal**: the LOO moves to `rejected` and stays there. It can be
viewed and deleted, and nothing else — `update`, `submit`, `manageSpaces`,
`comment`, `export`, `send` and the rest all report `false` and answer `403`. Its
comment threads stay readable. A rejected offer is spent, like a declined or
expired one, so it does not block generating a fresh LOO for the same source; the
corrected offer is that new LOO, with its own chain.

Send, and everything the tenant portal is allowed to see, are gated on
`approvalChainCleared()`, which reads the **status** rather than the steps — a LOO
approved because no step applied to it has no chain at all, and is no less
approved for that. [Export](#export) is not: a draft or an offer in its chain can be
downloaded too, marked DRAFT.

## Comments

The review conversation on an offer: a reviewer raises a point, whoever sent the
offer round answers it, and the thread is closed when it has been dealt with.

Deliberately **not** `approval_steps.comment`, which already exists and stays what
it is — the single reason attached to a review or rejection decision, and one the
framework nulls out on approve. A discussion needs what a decision does not: many
remarks against one document, answers to them, and a state saying whether the
point was ever settled.

### Who may take part

**Anyone in the offer's approval chain** — whoever submitted it, plus anyone
holding the role of any of its steps. That is the whole rule, and it is why the
comment surface adds **no permission of its own**: no permission can express
"the people being asked to approve this document". Reading falls back to
`view-loo`, so a manager can follow a review they are not personally part of
without being able to join it.

Membership is role-based rather than actor-based, so a reviewer three steps down
the chain — whose turn has not come and who therefore has no `actors` entry yet —
can still comment. It also spans every `attempt`: somebody who reviewed the offer
before it was rejected does not stop being party to the conversation when it comes
round again.

**An offer that has never been submitted has no chain, so nobody can comment on
it** — not even the person who drafted it. The conversation opens when the
document goes round.

> **None of this reaches the tenant.** `viewComments` deliberately does not route
> through `view()`, whose second branch hands an approved offer to the tenant it is
> addressed to. The review thread is where staff say the deposit is too low; it
> must not become visible the moment the chain clears. There is no comment
> endpoint on the tenant portal at all.

### What a comment is about

A comment can be anchored to the passage it concerns. `comment_in` names which
clause body the reader was in, and `highlighted_content` holds the text they
highlighted inside it.

| Field | Type | Notes |
|---|---|---|
| `comment_in` | enum \| null | `offer_content` or `agreement_content` — the `loos` columns of the same names. Serialised as `{ value, color }` |
| `highlighted_content` | text \| null | The highlighted passage, verbatim |

**Both are optional.** The highlight-then-comment flow is the usual one, but a
reviewer also has document-level remarks — "this offer is missing a break clause"
— that are about no particular sentence, and requiring an anchor would make those
lie about one. A passage does require a clause, though: `highlighted_content`
without `comment_in` is a validation error, while `comment_in` alone is a fine
remark about that clause generally.

**The passage is stored as text, not as character offsets.** Clause bodies stay
editable right through approval, and an offset would silently come to point at
different words after the first edit — the comment would still render, against the
wrong sentence. The quoted text stays truthful even when the paragraph around it
moves: find it by searching the clause, and where the search fails, say the
passage is no longer in the document rather than highlighting something nobody
meant. The same reasoning `document_upload_id` is held under.

The anchor is fixed once a comment is raised. `PATCH` changes the body and nothing
else — a remark that could be re-pointed after people had answered it would leave
their answers attached to a question nobody asked.

### Threads

**A reply is a comment.** It can be replied to in turn, to any depth — there is no
one-level limit. What makes a comment a reply is `parent_id` and nothing else.

| Field | Notes |
|---|---|
| `parent_id` | The comment being answered. `null` on a thread opener |
| `root_id` | The comment that opened the thread. `null` on a root, which is also what identifies one |
| `depth` | `0` on a root, `parent.depth + 1` below it — indent by this without rebuilding the tree |

`root_id` is carried denormalised so a whole conversation is one indexed read
rather than a walk down a level at a time. The index endpoint uses it to return
fully nested threads in two queries regardless of depth.

A reply **inherits** two things from its thread rather than restating them:

- **its anchor** — the passage under discussion belongs to the conversation, not to
  each message in it. Sending `comment_in` or `highlighted_content` on a reply is
  a validation error, not a silent discard;
- **its `attempt`** — an answer posted during attempt 2 to a point raised in
  attempt 1 belongs to that first exchange. Filing it under the current attempt
  would split one conversation across two rounds and neither half would read.

A new remark, by contrast, takes the attempt the offer is on now — so after a
rejection the old threads stay readable at `attempt: 1` beside the fresh ones.

### Resolving

**Resolution belongs to the thread, not to a message in it.** Only the comment
that opened one can be resolved, and closing it closes the conversation beneath
it; a resolved reply with unanswered children hanging off it would be a state
nobody could read. Resolving a reply is a `422`.

Two people may close a thread: **the person who raised it**, and **whoever
submitted the offer for the attempt it was raised under** (`initiated_by_id`).
Not everyone in the chain — that would let the reviewer a question was aimed at
close it without answering. Reopening is held to the same two.

A resolved thread is frozen against edits: its author cannot reword or withdraw a
comment inside it until it is reopened.

### Editing and withdrawing

Only the author, and only while the thread is open. Withdrawing a comment
soft-deletes it **and every answer beneath it** — an answer whose question has
gone is unreadable — while leaving its ancestors and its siblings standing.
Deleting a root removes the whole conversation.

## Delivery, the answer, and what it became

Set by the workflow, never by an edit — they are deliberately absent from the
update payload, so an approval step cannot hand somebody the right to backdate a
signature.

| Field | Written by | Notes |
|---|---|---|
| `sent_at`, `sent_to`, `sent_by_id` | send | `sent_to` is the address the copy actually went to |
| `document_upload_id` | send | The exported PDF **as delivered** |
| `responded_at` | signature | One timestamp for both outcomes; `status` says which |
| `signed_by_id` | signature | Who *recorded* the answer — staff, or the tenant |
| `signatory_name` | signature | Who actually signed, where that differs |
| `signature_upload_id` | signature | The signed copy, on an acceptance only |
| `decline_reason` | signature | On a decline only |
| `promoted_lease_id`, `promoted_at` | promotion | What the offer became |
| `can_promote` | promotion | Whether this offer can be onboarded right now — drives the button |
| `can_withdraw` | withdrawal | Whether this offer can be taken back right now — drives the button |
| `withdrawn_at`, `withdrawal_reason`, `withdrawn_by` | withdrawal | Set when an offer is withdrawn |

`document_upload_id` is the copy that went out, not one that could be
re-rendered. Re-rendering reads the LOO as it stands *now*; what a tenant signed
is what the file says, and the two must not be able to disagree. The tenant
portal serves that stored file.

## Endpoints

Base: `api/v1/app/{company}/property-management/lease-management`

### Generating

| Method | Path | Permission |
|---|---|---|
| `POST` | `lease-applications/{application}/loos` | `generate-loo` |
| `POST` | `leases/{lease}/loos` | `generate-loo` |

Two entry points because the thing being offered against is genuinely different:
an application yields a `new lease` (a `renewal` when the application has
`is_renewal`), a lease a `renewal` or an `addendum`. This
is the **only** step that needs to know the source — everything after it is a
`{loo}`. Body: `{ "loo_template_id": 4, "type": "renewal" }`, both optional
(`type` is ignored on the application route, which produces a `new lease`, or a
`renewal` for a [renewal application](../lease-applications.md#renewal-applications)).
A renewal prepared from an application reads its tags from the application, like a
`new lease`.

**Choosing a template.** Left out, the server resolves it: the default for this property and space
type, or the only eligible one. Where several are eligible and none is default, it refuses with a
`422` keyed `loo_template_id` — **and returns the choices alongside it**, so a picker needs no
second call:

```json
{ "errors": {
    "loo_template_id": ["Several templates cover this property and space type. Choose one, or mark a default."],
    "eligible_templates": [
      { "id": 4,  "name": "Standard Commercial Offer", "is_default": false },
      { "id": 14, "name": "Warehouse Offer",           "is_default": false }
    ]
} }
```

Resend with `loo_template_id` set to one of those ids. A template that is not eligible is refused
separately with *"That template is not available for this property and space type."*

If you would rather list them up front, the eligible set is
`GET .../settings/loo-templates?filter[facility_id]=…&filter[facility_type_id]=…&filter[is_active]=1`
— **all three filters**, since eligibility is active **and** property **and** type. Omitting
`is_active` offers templates the server will reject.

The response carries the offer, its resolved tags, and what the registry
deliberately did not resolve:

```json
{ "data": {
  "message": "Offer generated successfully.",
  "loo": { "...": "..." },
  "tags_content": [ { "tag_key": "tenant_name", "resolved_value": "Acme Traders", "is_overridden": false } ],
  "unresolved_tags": { "rent_review": "No escalation or rent-review basis is recorded…" }
} }
```

Generation refuses, with a validation error, when: the application has not been
approved; the source already carries a live offer (anything but `declined` or
`expired`); or no template — or more than one, with no default — covers the
property and space type.

### The offer

| Method | Path | Permission |
|---|---|---|
| `GET` | `loos` | `view-loo` |
| `GET` | `loos/{loo}` | `view-loo` |
| `PUT`/`PATCH` | `loos/{loo}` | `update-loo`, or an approval step's edit grant |
| `DELETE` | `loos/{loo}` | `delete-loo` |
| `POST` | `loos/{loo}/submit` | `update-loo` |
| `GET` | `loos/{loo}/export` | `export-loo` |
| `POST` | `loos/{loo}/send` | `send-loo` |
| `POST` | `loos/{loo}/signature` | `sign-loo` |
| `POST` | `loos/{loo}/promote` | `create-lease` |

There is **no `POST loos`**: an offer is generated from a source record, never
posted into existence on its own.

Index filters: `status`, `type`, `facility_id`, `loo_template_id`, `our_ref`,
`loo_preparation_type`, `loo_preparation_id`. The last two are how you list the
offers on one application or one lease.

`promote` is gated on `create-lease` rather than a LOO permission — it creates a
lease, and reaching it through an offer should not be a way around the permission
governing that everywhere else.

#### Renewed lease

A `renewal` offer's detail (`GET loos/{loo}`, the `PATCH` response and the generate
response) carries the lease being renewed and the deposits the landlord holds on it, so
whoever prepares the offer sees what [`deposit_held`](#deposits) was read from and can
check it against the figures:

```json
"renewed_lease": {
  "id": 101,
  "status": { "value": "active", "color": "success" },
  "start_at": "2024-01-01T00:00:00.000000Z",
  "end_at": "2026-12-31T00:00:00.000000Z",
  "period_in_years": 3,
  "period_in_months": 0,
  "total_deposit_amount": 315000,
  "deposits": [
    { "id": 55, "component": { "id": 1, "name": "Rent" }, "amount": 240000, "billed": true, "invoice_id": 9012, "refunded_at": null },
    { "id": 56, "component": { "id": 2, "name": "Service Charge" }, "amount": 60000, "billed": true, "invoice_id": 9012, "refunded_at": null },
    { "id": 57, "component": { "id": 7, "name": "Utility Deposit" }, "amount": 15000, "billed": false, "invoice_id": null, "refunded_at": null }
  ]
}
```

`total_deposit_amount` counts the rows with no `refunded_at`. Every row is listed,
refunded ones included, so the panel can show why a figure was left out. The key is
absent (not `null`) on a `new lease` or `addendum` offer, on the list, and on the tenant
portal. The lease itself is the preparation on an offer drafted from the lease, or the
renewal application's `lease` on one drafted from an application.

### Editing

`PATCH loos/{loo}` covers all three editable surfaces in one request, because
correcting a deposit, the paragraph mentioning it and the figure printed inside
that paragraph is one act of review:

```json
{
  "offer_content": "<p>Revised offer for {{tenant_name}}.</p>",
  "rent_deposit_months": 3,
  "legal_fees": 45000,
  "tag_values": [ { "tag_key": "tenant_name", "resolved_value": "Acme Traders Limited" } ]
}
```

Every LOO-side field is editable, **including the auto-computed ones**. A tag
correction sets `is_overridden`, which is what stops an ordinary re-resolution
putting the computed value back.

**It writes what it was sent and recomputes nothing.** An update that quietly
re-derived a figure over the one just typed would make "every field is editable"
untrue. Two exceptions, both because the field edited is an *input* to the figure:

- moving `lease_term_years`/`lease_term_months` rebuilds the rent schedule — a schedule
  has no meaning without an end date — *unless* the same request also carried
  `rent_breakdown`, in which case yours stands;
- moving `rent_deposit_months` re-derives `rent_security_deposit` and
  `service_charge_security_deposit` (months × the final period's rent and service charge,
  see [Deposits](#deposits)) and refreshes the deposit tags that were not hand-corrected —
  *unless* the same request also carried the deposit column in question.

A `tag_values` correction for a tag whose format is `html` (today `rent_breakdown`) is
sanitised before it is stored: only `table`, `thead`, `tbody`, `tr`, `th`, `td`, `p`,
`br`, `strong` and `em`, with `class` and `colspan`, survive. Every other tag's value is
stored as sent and escaped when the document is rendered.

`legal_fee_updated_at` and `legal_fee_recorder_id` are stamped by the server
whenever a fee moves, and are not accepted in the payload.

Only while `status` is `draft` or `pending_approval`. An offer in a tenant's
hands is fixed, and so is a `rejected` one.

### Granted spaces

| Method | Path | Permission |
|---|---|---|
| `GET` | `loos/{loo}/spaces` | `view-loo` |
| `POST` | `loos/{loo}/spaces` | `update-loo` |
| `PUT`/`PATCH` | `loos/{loo}/spaces/{looSpace}` | `update-loo` |
| `DELETE` | `loos/{loo}/spaces/{looSpace}` | `update-loo` |

```json
{
  "facility_space_id": 91,
  "components": [ { "lease_component_id": 3, "cost_per_space_unit": 250, "tax_id": null } ]
}
```

`amount` is never submitted — it is `cost_per_space_unit` × the space's size,
computed server-side, so a client cannot send a total that does not follow from
the rate it sent alongside it. Eligibility and suggested pricing come from the
same `SpaceComponentPricer` the application side uses, so a component billable on
a space at application time is billable on it here and costs the same.

`PATCH` replaces the component set outright — a merge cannot express "this space
no longer carries a service charge". The space itself cannot be moved: withdraw
it and grant the other one, so the new space's own rules are actually applied.

**Every write here re-prices the offer.** `monthly_service_charge` and its
quarterly/annual siblings, `rent_security_deposit`,
`service_charge_security_deposit`, the rent schedule and the space-derived tags
are all re-derived from the spaces as they now stand — so each response returns
the whole `loo` alongside the space, because your copy of those figures is stale.
A hand-adjusted figure **is** replaced, deliberately: the spaces it was typed
against have moved.

The two deposits are months × the **final period's** rent and service charge, and the
schedule is rebuilt first so they read the escalated figure — see [Deposits](#deposits).
On a `renewal`, `deposit_held` is re-read from the lease being renewed at the same time.
Only `utility_security_deposit` is never recomputed: utilities are not priced through
`loo_spaces` at all. On a `new lease` or `addendum`, `deposit_held` is not derived
either.

> **Net of tax.** The recomputed service-charge and deposit columns are net,
> matching the components they come from. The rent *schedule* is tax-inclusive,
> because that is what a tenant actually pays per period. `has_vat` tells the
> clause whether tax is added on top.

### Comments

| Method | Path | Permission |
|---|---|---|
| `GET` | `loos/{loo}/comments` | `view-loo`, **or** membership of the approval chain |
| `POST` | `loos/{loo}/comments` | membership of the approval chain |
| `PUT`/`PATCH` | `loos/{loo}/comments/{looComment}` | the comment's author |
| `DELETE` | `loos/{loo}/comments/{looComment}` | the comment's author |
| `POST` | `loos/{loo}/comments/{looComment}/resolve` | the thread's author, or the submitter |
| `DELETE` | `loos/{loo}/comments/{looComment}/resolve` | the thread's author, or the submitter |

No permission of its own — see [Comments](#comments) for why, and for the rules
behind the right-hand column. There is **no `GET loos/{loo}/comments/{looComment}`**:
a single message out of its conversation is not a thing worth addressing.

Raising a point, anchored to a passage:

```json
{
  "body": "This deposit is two months short of policy.",
  "comment_in": "offer_content",
  "highlighted_content": "a deposit equal to one month's rent"
}
```

Answering one. A reply carries neither `comment_in` nor `highlighted_content` —
it inherits the thread's, and sending either is a `422`:

```json
{ "body": "Corrected to three months in the latest revision.", "parent_id": 41 }
```

`GET` paginates **threads, not messages** — a page of roots, each with its answers
nested beneath it to whatever depth they run. Paginating a flat list would cut
conversations in half at a page boundary. A leaf reports `"replies": []` rather
than omitting the key.

```json
{ "data": [{
  "id": 41,
  "attempt": 1,
  "body": "This deposit is two months short of policy.",
  "comment_in": { "value": "offer_content", "color": "info" },
  "highlighted_content": "a deposit equal to one month's rent",
  "parent_id": null, "root_id": null, "depth": 0, "is_reply": false,
  "author": { "id": 7, "name": "A. Reviewer" },
  "is_resolved": true,
  "resolved_at": { "raw": "2026-09-04T09:12:00Z", "formatted": "04 Sep, 2026", "diff": "2 hours ago" },
  "resolved_by": { "id": 3, "name": "L. Manager" },
  "approval_step": { "id": 88, "step_order": 1, "role": { "id": 12, "name": "Finance" } },
  "replies": [{
    "id": 42, "parent_id": 41, "root_id": 41, "depth": 1, "is_reply": true,
    "body": "Corrected to three months in the latest revision.",
    "comment_in": { "value": "offer_content", "color": "info" },
    "replies": []
  }]
}] }
```

Filters: `attempt`, `comment_in`, `resolved`, `unresolved`. Sorts: `id`,
`created_at`, `resolved_at`; newest thread first by default.

```http
GET loos/{loo}/comments?filter[unresolved]=1&filter[comment_in]=offer_content
```

`approval_step` is where the remark was raised, recorded only when its author was
the person the chain was waiting on at the time. It is context, not authority — a
thread outlives the step it started on.

### Export

```http
GET loos/{loo}/export?document=offer&format=pdf
```

`document` is `offer` (default) or `agreement` — they are separate documents, an
offer is issued and an agreement is executed. `format` is `pdf` (default) or
`html`; both come off one renderer, so the preview pane and the tenant's copy
cannot drift.

The rendered document is the editor content and nothing else: no letterhead, title,
reference line or footer is added around it. Write those into the template (the
`{{our_ref}}` and `{{offer_date}}` tags are there for the reference line).

Allowed on a `draft` and a `pending_approval` offer as well as an approved one
(`approved`, `sent`, `accepted`), so staff can read and circulate the document before it is
approved. Refused (`403`) on a closed offer: `declined`, `expired`, `withdrawn` or `rejected`.

**An offer that has not cleared approval is watermarked.** A large, translucent red "DRAFT"
runs diagonally across every page of the PDF and the HTML. An approved offer carries no mark.

Only an approved offer can be [sent](#send): the file Send attaches and files against the
offer is still refused for anything short of a cleared chain, so a draft never reaches the
tenant.

The response is the file. `X-Loo-Unresolved-Tags` names any token the render left
standing or blank — reported rather than refused, since a blank is often correct
(an offer with no guarantors prints nothing for them) and only the reader can
tell that from an omission. In the rendered output an **unknown** token is left
visible as `{{token}}`; a tag that resolved to nothing renders as empty.

Resolved values are HTML-escaped on the way in. The clause text around them is
markup a template author wrote and passes through; a value read out of an
application is data and does not. The one exception is a tag whose registry format is
`html` (`rent_breakdown`): its value is markup the resolver built, and it is inserted as
such after passing the same sanitiser a reviewer's correction goes through. A `<p>` that
holds nothing but such a token is unwrapped first, since a table cannot sit inside a
paragraph.

### Send

```json
{ "document": "offer", "email": "agent@example.test", "note": "As discussed." }
```

All three optional. Exports the document, files it as an `Upload` owned by the
offer, records `sent_at`/`sent_to`/`sent_by_id`, moves `status` to `sent`, and
notifies the tenant (database, plus mail where there is an address). An `email`
overrides the address on file — an agent, a company secretary — and is recorded
either way, so what happened is on the document rather than in someone's mailbox.

The email carries the filed PDF as an attachment, the same file the tenant portal's
download serves.

Re-sending a `sent` offer is allowed; a lost email is a real thing. Once the
tenant has answered it is refused, because a second copy arriving after an answer
invites a second answer.

### Signature

```json
{ "accepted": true, "signatory_name": "J. Mwangi, Director", "signature_upload_id": 55 }
```
```json
{ "accepted": false, "decline_reason": "Rent above budget" }
```

Only from `sent` — an approved offer nobody has been given cannot have been
answered, and an answered one is not re-answered. A tenant who changes their mind
needs a fresh offer; the first answer is part of the record.

`signature_upload_id` is refused on a decline, and `decline_reason` on an
acceptance.

`declined` means **the tenant** declined. It is not where a rejected approval
lands — an offer the landlord's own chain refused goes to `rejected`.

### Promote

```json
{ "billing_cycle": "quarterly", "start_at": "2027-04-01", "currency_id": 2 }
```

All optional. Creates the `Lease` with `lease_items` and
`lease_item_components` copied from the granted spaces, and `lease_escalations`
copied from `lease_application_escalations` — that table was written in the shape
of `lease_escalations` for exactly this hop. Two figures change name and nothing
else does: `cost_per_space_unit` → `cost_per_sqft`, `amount` → `cost_per_month`,
both still net of tax.

`billing_cycle` is asked for because a LOO prices a tenancy per period but no
column on it says how often that is billed. Left out, it defaults to the
application's own `billing_cycle`, and to `monthly` when the application has none. `start_at`
overrides the proposed start, for the ordinary case of a tenancy agreed in March
and signed in April. `currency_id` defaults to the property's reporting currency,
and is left null when the property records none rather than guessing one.

The offer records `promoted_lease_id`/`promoted_at`, and the application picks up
the signed copy as its `signed_agreement_upload_id`. The application's *status*
does not move — it already sits at `approved`, since an offer cannot be drafted
against one that does not.

**An offer may be onboarded from `sent` as well as `accepted`** — as soon as it has gone to the
tenant, without waiting for their answer. Anything earlier is refused: a `draft`, one awaiting
approval, or one `approved` but not yet sent has not reached the tenant, so a lease raised from it
would belong to an offer nobody has read.

**An onboarded offer can no longer be declined.** Recording a decline against one that has already
become a lease is refused with a `422` keyed `loo`. Without that, a decline left the lease active -
billing, holding its space, on the reports - with a declined offer still pointing at it.

**Accepting after onboarding is still allowed, and is the normal path.** It is how the signed offer
is captured, and the lease's Documents tab reads that signature straight off the offer, so a tenant
signing after the lease exists is exactly what should happen.

Refused when the offer has not been sent, when it has already been promoted, and when it grants no
spaces.

**Read `can_promote` to decide whether to offer the button.** It is the same rule the endpoint
enforces — sent or accepted, right type, not already promoted. Do **not** use
`permissions.promote` for this: that checks the caller's permission only, so it is `true` on a
draft and on an offer already turned into a lease.

### Withdraw

`POST /api/v1/app/{company}/property-management/lease-management/loos/{loo}/withdraw`

Takes back an offer that should not have gone out — wrong figures, wrong recipient. Optional body:

```json
{ "reason": "Wrong service charge rate" }
```

Allowed from **`approved`** and **`sent`** only. A `draft` or one awaiting approval can simply be
deleted, so withdraw adds nothing there; an `accepted` offer is an agreement the tenant has given,
which is a different conversation from retracting a mistake.

**Refused on an offer already onboarded into a lease.** The lease is live and nothing here unwinds
it — allowing it would reopen the hole the decline guard closes.

A withdrawn offer is **spent**, so the application is free for a replacement immediately. That is
the point of it: until now the error raised when drafting a replacement said *"Withdraw or delete
it"*, while delete stops working the moment an offer is sent — leaving a decline the tenant never
gave as the only way out.

**Read `can_withdraw` to decide whether to offer the button.** It is the same rule the endpoint
enforces. `permissions.withdraw` is not a substitute: it checks the caller's permission only, so it
is true on a draft and on an offer already withdrawn.

Withdrawal is recorded separately from a decline — `withdrawn_at`, `withdrawn_by`,
`withdrawal_reason` — because one is the landlord's decision and the other the tenant's. Writing a
withdrawal into the decline columns would make the record say the tenant answered when they never
did.

#### Renewals and addenda

`POST loos/{loo}/promote` refuses a `renewal` or an `addendum` with a validation
error naming the type. A renewal amends the lease it was prepared from rather
than creating one beside it, and exactly what it amends — the term, the pricing,
the escalations, whether existing items are replaced or extended — was never
scoped. A guess would have produced a second live lease over the same spaces,
which is precisely what the polymorphic preparation model exists to prevent.

Everything above that point is preparation-agnostic: the route is `{loo}`, the
controller does not know the source, and the action branches on
`type->preparesFromApplication()`. Wiring the update path in later is a branch
inside one method with nothing above it to restructure.

### Rebase

`POST /api/v1/app/{company}/property-management/lease-management/loos/{loo}/rebase`

Throws the offer away and puts its application back where it was before the offer was drafted.
Use it when the offer has to be redone from scratch: the application changed, the figures were
wrong, or the application itself needs another review.

```json
{ "return_to": "approved" }
```

```json
{ "return_to": "reviewer", "reason": "Service charge rate on the application is outdated." }
```

| Field | Required | Type | Notes |
|---|---|---|---|
| `return_to` | Yes | string | `approved` \| `reviewer` |
| `reason` | When `return_to` is `reviewer` | string | Max 1000 characters. Optional for `approved` |

What happens, in one transaction:

1. The offer is **deleted** (soft-deleted, like `DELETE loos/{loo}`). Its pending tasks,
   approval-step tasks and notifications go with it, and the tenant portal stops showing it.
2. Then, by `return_to`:
   - **`approved`**: the application stays `approved`, and the **Generate Letter of Offer**
     task is raised again for the LOO generation role. The next offer is drafted with
     [`POST lease-applications/{application}/loos`](#generating), from the application as it
     stands now.
   - **`reviewer`**: the application is
     [returned to its reviewer](../lease-applications.md#return-an-approved-application):
     it goes back to `pending`, the approver gets a task, and an in-app, email and SMS
     notification with the reason.

The response is the application, not the offer, which no longer exists:

```json
{
  "message": "Offer rebased. The application is waiting for a new offer.",
  "application": { "id": 12, "status": { "value": "approved", "color": "success" }, "...": "..." }
}
```

Refused with `422` (key `loo`) when the offer:

- was prepared from a **lease** (a renewal or addendum). There is no application to return to.
  Delete or withdraw it instead;
- has been **onboarded into a lease**. The lease is live and nothing here unwinds it;
- is already **spent** (`declined`, `expired`, `rejected`, `withdrawn`). It no longer blocks a
  new offer, so there is nothing to rebase. To send the application back to its reviewer, use
  [`PATCH lease-applications/{application}/return`](../lease-applications.md#return-an-approved-application).

Any other status can be rebased: `draft`, `pending_approval`, `approved`, `sent`, and `accepted`
(an acceptance that has not been onboarded).

**Who may rebase:** `rebase-loo` for `return_to: approved`. `return_to: reviewer` also needs
`return-lease-application`, the permission behind returning an application directly.

**Read `can_rebase` to decide whether to offer the button.** It applies the same status rules as
the endpoint. For the caller's rights, read `permissions.rebase` (may rebase at all) and
`permissions.rebaseToReviewer` (may also choose the reviewer option).

## The tenant portal

Base: `api/v1/tenant/loos`

| Method | Path | Permission | |
|---|---|---|---|
| `GET` | `` | — | The tenant's offers |
| `GET` | `/{loo}` | — | One offer, with its resolved tags |
| `GET` | `/{loo}/download` | — | The document **as sent** |
| `POST` | `/{loo}/signature` | `sign-loo` | Accept or decline |

The reads are authorised by **ownership** alone. Two filters run on every query:
the offer's preparation record must belong to this tenant, and the offer must have
cleared its approval chain. Both are applied as scopes on the query, so anything
short of an approved offer is a **404** here, never a 403 that would confirm it
exists.

Signing asks ownership **first** — so somebody else's offer is still a 404, never a
403 — and then `sign-loo`, which is held on the tenant guard as well as the staff
one. The seeded tenant role carries every tenant-scoped permission, so a tenant has
it unless it has been deliberately taken away; one name covers the act whether the
answer arrives through the portal or over the counter.

`download` serves the stored file, never a fresh render, and 404s when nothing
has been filed — an offer the tenant can see but that carries no document has
been approved and not yet sent.

The signature endpoint runs the same action as the staff one, so an acceptance
recorded on the portal and one recorded over the counter are the same record.

**There is no comment endpoint here, deliberately.** [Comments](#comments) are the
internal review conversation — where staff say the deposit is too low and the term
is wrong — and the staff-side `viewComments` check is written so that clearing the
approval chain does not open that thread to the tenant it is addressed to.

## Permissions

Added to `storage/app/seeders/permissions.json` under the `Property Management`
tag.

Ordinary CRUD throughout, with `generate-` standing in for `create-` because an
offer is drafted from a source record rather than posted into existence.

| Permission | Guard | Covers |
|---|---|---|
| `view-loo` | app | Reading offers |
| `generate-loo` | app | Drafting one from a source record |
| `update-loo` | app | Editing clauses, fields and resolved tags; **granting, withdrawing and re-pricing spaces**; submitting for approval |
| `delete-loo` | app | Deleting a draft, a pending or a rejected offer |
| `rebase-loo` | app | [Rebasing](#rebase) an offer: deleting it and putting its application back to `approved` |
| `return-lease-application` | app | With `rebase-loo`, rebasing to the reviewer. On its own, [returning an approved application](../lease-applications.md#return-an-approved-application) |
| `export-loo` | app | Rendering **either document** — offer or agreement — to PDF or HTML, a draft included (watermarked) |
| `send-loo` | app | Delivering to the tenant |
| `sign-loo` | **app + tenant** | Answering an offer: the tenant in the portal, or staff recording one that came back on paper |
| `view-loo-template` | app | Reading templates and the tag registry |
| `create-loo-template` | app | Authoring a template |
| `update-loo-template` | app | Editing one, and restoring a deleted one |
| `delete-loo-template` | app | Deleting one |

**[Comments](#comments) add no permission.** Who may raise and answer a point is
membership of the offer's approval chain, which no permission can express; reading
the thread falls back to `view-loo`. Nothing was added to `permissions.json` for
it, and there is no patch seeder to run.

**Spaces have no permission of their own.** What an offer covers and what it costs
*is* the offer, and somebody trusted to rewrite the deposit clause is not
separately untrusted to price the unit that clause is about — so `update-loo`
covers both. The approval edit grant deliberately does not reach the spaces,
though: a step lends out the right to correct named `editable_fields`, and a grant
covering `rent_deposit_months` must not become a licence to re-price the letting.

**`sign-loo` is held on both guards**, because signing is one act however it
reaches the system. A tenant answers their own offer in the portal; staff record
the same answer when it came back over the counter, and both go through
`LooPolicy::recordSignature()`. It is separate from `send-loo` — putting an offer
in front of a tenant and answering one are different jobs. The tenant portal still
tests **ownership first**, so an offer that is not yours is a 404 before the
permission is ever consulted.

`update-loo` is also the name the approval framework's edit grant checks
(`Approvable::approvalEditBypassPermission()` derives `update-{model}`). Promotion
reuses the existing `create-lease`.

> **Upgrading an existing install.** `manage-loo-template` and `manage-loo-spaces`
> are gone, and `sign-loo` is new. Run
> `php artisan db:seed --class=LooCrudPermissionsPatchSeeder`: it creates the new
> names, **carries the existing grants across** — template authors keep authoring,
> space pricers keep pricing, anyone who could record a signature still can, and
> every tenant role gains `sign-loo` — and only then drops the two retired
> permissions. It is idempotent.

## Related

- [LOO templates](./loo-templates.md) — the clause text a LOO is drafted from
- [Tag reference](./loo-tags.md) — all 62 tags, and how each resolves
- [README](./README.md) — the tag registry, resolution, and the status lifecycle
- [Approval templates](../../../access-management/approval-templates.md) — step conditions and the edit grant
- [Leases API](../leases.md) — what a `new lease` LOO is promoted into
