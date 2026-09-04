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
submitted deliberately — `shouldAutoInitiateApproval()` returns `false`, and
`POST loos/{loo}/submit` calls `ApprovalService::initiate()`.

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
an application yields a `new lease`, a lease a `renewal` or an `addendum`. This
is the **only** step that needs to know the source — everything after it is a
`{loo}`. Body: `{ "loo_template_id": 4, "type": "renewal" }`, both optional
(`type` is ignored on the application route, which can only produce a
`new lease`).

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
untrue. One exception: moving `lease_term_years`/`lease_term_months` rebuilds the
rent schedule — a schedule has no meaning without an end date — *unless* the same
request also carried `rent_breakdown`, in which case yours stands.

`legal_fee_updated_at` and `legal_fee_recorder_id` are stamped by the server
whenever a fee moves, and are not accepted in the payload.

Only while `status` is `draft` or `pending_approval`. An offer in a tenant's
hands is fixed.

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

Two columns are never recomputed, because nothing on the offer can derive them:
`utility_security_deposit` (utilities are not priced through `loo_spaces` at all)
and `deposit_held` (what a tenant has actually paid — a receipt's answer, not a
rate card's).

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

Gated on the approval chain. A file, once produced, gets forwarded.

The response is the file. `X-Loo-Unresolved-Tags` names any token the render left
standing or blank — reported rather than refused, since a blank is often correct
(an offer with no guarantors prints nothing for them) and only the reader can
tell that from an omission. In the rendered output an **unknown** token is left
visible as `{{token}}`; a tag that resolved to nothing renders as empty.

Resolved values are HTML-escaped on the way in. The clause text around them is
markup a template author wrote and passes through; a value read out of an
application is data and does not.

### Send

```json
{ "document": "offer", "email": "agent@example.test", "note": "As discussed." }
```

All three optional. Exports the document, files it as an `Upload` owned by the
offer, records `sent_at`/`sent_to`/`sent_by_id`, moves `status` to `sent`, and
notifies the tenant (database, plus mail where there is an address). An `email`
overrides the address on file — an agent, a company secretary — and is recorded
either way, so what happened is on the document rather than in someone's mailbox.

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
lands — an offer the landlord's own chain sent back goes to `draft`.

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
column on it says how often that is billed; it defaults to `monthly`. `start_at`
overrides the proposed start, for the ordinary case of a tenancy agreed in March
and signed in April. `currency_id` defaults to the property's reporting currency,
and is left null when the property records none rather than guessing one.

The offer records `promoted_lease_id`/`promoted_at`, and the application picks up
the signed copy as its `signed_agreement_upload_id`. The application's *status*
does not move — it already sits at `approved`, since an offer cannot be drafted
against one that does not.

Refused when the offer is not `accepted`, when it has already been promoted, and
when it grants no spaces.

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
| `delete-loo` | app | Deleting a draft |
| `export-loo` | app | Rendering **either document** — offer or agreement — to PDF or HTML |
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
