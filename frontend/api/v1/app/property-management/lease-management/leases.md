# Leases API

Domain: `Property Management > Lease Management`

Base route:

`/api/v1/app/{company}/property-management/lease-management/leases`

## Main Endpoints

- `GET /leases` (list)
- `POST /leases` (create)
- `GET /leases/{lease}` (show)
- `PUT/PATCH /leases/{lease}` (update)
- `DELETE /leases/{lease}` (delete)
- `PATCH /leases/{lease}/activate`
- `PATCH /leases/{lease}/suspend`
- `PATCH /leases/{lease}/terminate`
- `POST /leases/{lease}/renew`: start a renewal of a lease that is ending. See
  [Renew a lease](#renew-a-lease)
- `POST /leases/{lease}/generate-invoice-for-next-period`
- `GET /leases/{lease}/invoice-components`: what a manual invoice on the lease can bill. See
  [Invoices](invoices.md#components-a-manual-invoice-can-bill)

## List Leases

`GET /api/v1/app/{company}/property-management/lease-management/leases`

Supported query params:

- Filters:
  - `filter[facility_id]` — **exact**; comma-separated ids match any. Lease pickers narrow to one
    property with it
  - `filter[user_id]`
  - `filter[status]`
  - `filter[created_at]`
  - `filter[start_at]`
  - `filter[end_at]`
  - `filter[next_due_at]`
  - `filter[billing_cycle]`
- Sort:
  - `sort=user_id,status,created_at,start_at,end_at,next_due_at,billing_cycle`
  - Prefix with `-` for descending (example: `sort=-start_at`)
- Include: not supported
- Select fields: not supported
- Pagination:
  - `per_page` (optional)
  - `page` (optional)

Examples:

- `GET /api/v1/app/12/property-management/lease-management/leases?filter[status]=active&sort=-start_at`
- `GET /api/v1/app/12/property-management/lease-management/leases?filter[user_id]=77&per_page=10&page=2`

## Create Lease

`POST /api/v1/app/{company}/property-management/lease-management/leases`

Request body:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `user_id` | Yes | integer | Must exist in `users.id` |
| `start_at` | Yes | date (`YYYY-MM-DD`) | - |
| `period_in_years` | Yes | integer | - |
| `facility_id` | Yes | integer | Must exist in `facilities.id` |
| `period_in_months` | Yes | integer | - |
| `billing_cycle` | Yes | string | `biennial`, `annually`, `biannual`, `quarterly`, `monthly`, `weekly`, `daily`, `hourly` |
| `next_due_at` | Yes | date (`YYYY-MM-DD`) | - |
| `status` | No | string | `pending`, `active`, `suspended`, `terminated`, `rejected`. Not worth sending on create — see approval below |
| `currency_id` | No | integer | Must exist in `currencies.id` (defaults to `1` when omitted) |

Example request:

```json
{
  "user_id": 77,
  "start_at": "2026-01-01",
  "period_in_years": 1,
  "facility_id": 14,
  "period_in_months": 0,
  "billing_cycle": "monthly",
  "next_due_at": "2026-03-01",
  "status": "active",
  "currency_id": 1
}
```

Example response:

```json
{
  "message": "Lease created successfully",
  "lease": {
    "id": 101,
    "user": { "id": 77, "name": "Jane Tenant" },
    "facility": { "id": 14, "name": "Parkview Residency" },
    "billing_cycle": "monthly",
    "status": { "value": "active", "color": "success" },
    "start_at": { "raw": "2026-01-01", "formatted": "01 Jan, 2026" },
    "next_due_at": { "raw": "2026-03-01", "formatted": "01 Mar, 2026" }
  }
}
```


## Creation approval

A new lease goes through an approval chain before it becomes active, when the company has a
template configured for it.

| Status | Meaning |
| --- | --- |
| `pending` | Raised, waiting on its approval step. Not yet a working lease. |
| `active` | Approved, or created when no approval template is configured. |
| `rejected` | Refused. Terminal. |
| `suspended`, `terminated` | As before, set by hand later in the lease's life. |

**Do not send a status when creating.** The server decides: it writes `pending` and opens the
chain when a template applies, and leaves the lease `active` when none does. A status in the
payload is ignored for this purpose.

**A pending lease is deliberately not a working lease.** It is not invoiced, its space is not
marked occupied, and it does not appear in the dashboard counts or the tenant reports until it is
approved. If a lease seems to have vanished after creation, check whether it is awaiting approval
rather than assuming it failed to save.

**Approval and rejection run through the usual endpoints** — see
[approvals](../../access-management/approvals.md). The lease exposes `approval_steps` like any
other approvable record, so the same screens work for it.

**A lease promoted from a Letter of Offer skips this.** The Loo was approved on the way in, and
asking for a second approval of the same agreement would stall every promotion. Those leases are
created `active` with no chain.

**Nothing changes until a template exists.** With no active template for `Lease`, leases are
created `active` exactly as before, so this can be deployed ahead of the configuration.

## Update Lease

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}`

Use the same payload shape as create.

## Lease Status Actions

No request body required:

- Activate: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/activate`
- Suspend: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/suspend`
- Terminate: `PATCH /api/v1/app/{company}/property-management/lease-management/leases/{lease}/terminate`

Response shape:

```json
{
  "message": "Lease suspended successfully",
  "lease": {
    "id": 101,
    "status": { "value": "suspended", "color": "secondary" }
  }
}
```

## Renew a lease

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/renew`

Starts a renewal of a lease that is ending. It creates a **renewal lease application**
(`is_renewal: true`, `lease_id`) on the tenant's behalf and puts it in front of the
reviewer, exactly as if the tenant had applied from their portal. From there the ordinary
application flow applies: review → approve → generate the `renewal` offer → send → sign.
Promoting a signed renewal offer into the extended lease is still not built.

**Who.** Anyone who can view the lease: `view-lease` on an allocated property. There is no
separate renew permission.

**When.** The lease must be `active` with an `end_at` no more than 12 months away. A lease
whose `end_at` has already passed but is still `active` qualifies. Refused while the lease
already has a renewal in progress: a renewal application in `pending` or `review`, an
`approved` one whose offer has not been spent, or a live offer drafted straight off the
lease through `POST leases/{lease}/loos`.

Request body. Every field is optional:

| Field | Type | Default |
|---|---|---|
| `proposed_start_date` | date | the day after `end_at` |
| `proposed_period_in_years` | integer `0`–`99` | the lease's `period_in_years` |
| `proposed_period_in_months` | integer `0`–`11` | the lease's `period_in_months` |

A term of zero years and zero months is refused with `422` on `proposed_period_in_years`.

What the application is pre-filled with:

- **From the lease:** the tenant (`user_id`), the property and its type, the proposed start
  and term above, and `billing_cycle` where the lease's cycle is one an application accepts.
- **Spaces:** every lease item becomes an allocated application space with its priced
  components (`cost_per_space_unit`, `amount`, `tax_id`) copied across. Staff may reprice or
  change them while the application is open, as on any application.
- **Applicant profile and guarantors:** copied from the tenant's *source application*. That
  is the application whose offer was promoted into this lease, else the tenant's latest
  application on this property, else their latest application anywhere. Guarantor documents
  and verification results are not copied. With no application on record at all, the profile
  falls back to the tenant's account (name, email, phone, address, postal address, tax PIN)
  and the property's city and country; anything still unknown is written blank and is left
  for staff to complete through the [staff edit](lease-applications.md#staff-edit).
- **Not copied:** lease escalations (add them on the application), documents and vetting.

The tenant is told the application was received, the same notification they get when they
apply themselves, so they can read and complete it from their portal.

Response `201`:

```json
{ "data": {
  "message": "Renewal application created successfully.",
  "application": {
    "id": 913,
    "is_renewal": true,
    "lease_id": 101,
    "status": { "value": "pending", "color": "secondary" },
    "...": "..."
  }
} }
```

`application` is the full staff application resource, the same shape as
`GET lease-applications/{application}` (see [Lease Applications](lease-applications.md)),
with its spaces, guarantors and `lease` loaded.

Errors: `403` carrying the reason when the lease is not renewable or the caller is not
allocated to its property; `422` keyed `lease` if another renewal was started in the
meantime, or if the property (and, failing that, the tenant) has no city or country on
record, since an application cannot be written without them.

The refusal reasons, word for word: *Only an active lease can be renewed.* · *This lease
has no end date to renew from.* · *A lease can be renewed only within 12 months of its end
date.* · *This lease already has a renewal application in progress.* · *This lease already
has an offer in progress.* The zero-term `422` reads *A renewal needs a term of at least one
month.*

### Renewal state on the lease

`GET leases/{lease}` carries:

```json
{
  "permissions": {
    "view": true, "update": true, "delete": false,
    "activate": false, "terminate": true, "suspend": true,
    "renew": false
  },
  "open_renewal_application": { "id": 913, "status": "pending" }
}
```

- `permissions.renew` is `true` only when the call above would succeed: the caller may view
  the lease, it is active and ending within 12 months, and nothing is renewing it already.
  Show the **Renew** button on it, and nothing else.
- `open_renewal_application` is the renewal application in progress (`{ id, status }`), or
  `null`. When `renew` is `false` because of it, link to that application instead. It is on
  the detail only, not on the list.

The other `permissions` keys are the lease's own abilities (`view`, `update`, `delete`,
`activate`, `terminate`, `suspend`) and were always there; they are listed here because this
is the first place the key is documented.

## Generate Invoice For Next Period

`POST /api/v1/app/{company}/property-management/lease-management/leases/{lease}/generate-invoice-for-next-period`

Purpose:

- Manually generate the upcoming period's invoice for a lease ahead of the scheduled due date
  (`next_due_at`), when operationally necessary. Runs the same generation as the `invoices:generate`
  scheduler.

Request body:

- None.

Eligibility (returns `422` validation error otherwise):

- Lease `status` must be `active`.
- Lease must have a billing schedule (`next_due_at` set).
- At least one auto-billed component must be priced above zero. Otherwise the lease has nothing to
  bill: `422` with `lease`: `This lease has nothing to bill: every component is priced at zero.`
  No invoice is created and `next_due_at` does not move.
- Unlike the scheduler, the "next due" date check is **skipped** — `next_due_at` may be in the future.

Behavior:

- The invoice `notes` describe the billing **period** being generated for (derived from `next_due_at` and
  the billing cycle), not the date it was generated.
- Every invoice line carries a value. A component priced at zero stays on the lease but is never
  invoiced, so the invoice holds only the priced components.
- `next_due_at` is advanced to the start of the next period when the invoice is created.

Success response:

- `message`: `Invoice for next period generated successfully.`
- `invoice`: the created `FacilityInvoice`.

## Lease Items and Components

### Lease items endpoints

- `GET /leases/{lease}/items`
- `POST /leases/{lease}/items`
- `GET /leases/{lease}/items/{leaseItem}`
- `PUT/PATCH /leases/{lease}/items/{leaseItem}` (currently disabled by backend and returns validation error)
- `DELETE /leases/{lease}/items/{leaseItem}`

Create lease item payload:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `facility_space_id` | Yes | integer | Must exist in `facility_spaces.id`, must belong to lease facility, and must not be occupied |
| `components` | Yes | array | Array of lease item component objects |

Component object inside `components`:

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_component_id` | Yes | integer | Must exist in `lease_components.id` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |
| `cost_per_sqft` | Yes | integer | - |
| `hs_code` | No | string | Optional |

### Lease item components endpoints

- `GET /leases/{lease}/items/{leaseItem}/components`
- `POST /leases/{lease}/items/{leaseItem}/components`
- `GET /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`
- `PUT/PATCH /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`
- `DELETE /leases/{lease}/items/{leaseItem}/components/{leaseItemComponent}`

`POST` accepts either **one component** at the top level, or **several** under a `components`
array. Both are validated the same way; the bulk form previously accepted anything.

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `lease_component_id` | Yes | integer | Must exist in `lease_components.id` |
| `tax_id` | Yes | integer | Must exist in `taxes.id` |
| `cost_per_sqft` | Yes | number | `cost_per_month` is computed from it and the space size |
| `hs_code` | No | string \| null | Optional |

Errors on the bulk form are keyed by position, so the caller can mark the offending row:
`components.0.lease_component_id`, `components.2.tax_id`. An empty `components: []` is refused with
`components: ["At least one component is required."]`.

List component query params:

- Filters: `filter[lease_item_id]`, `filter[lease_component_id]`, `filter[cost_per_sqft]`, `filter[cost_per_month]`, `filter[tax_id]`, `filter[hs_code]`
- Sort: `sort=lease_item_id,lease_component_id,cost_per_sqft,cost_per_month,tax_id,hs_code`
- Include: not supported
- Select fields: supported
  - `fields[lease_item_components]=id,lease_item_id,lease_component_id,cost_per_sqft,cost_per_month,tax_id,hs_code`

Example list with field selection:

`GET /api/v1/app/12/property-management/lease-management/leases/101/items/55/components?fields[lease_item_components]=id,lease_component_id,cost_per_month&sort=-cost_per_month`

### Sending documents with the lease

`POST /leases` and `PUT/PATCH /leases/{lease}` accept an `uploads` array of upload ids. Upload each
file through `POST /api/v1/settings/file-management/uploads` first, then send the ids.

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `uploads` | No | array of integers | Upload ids. Each must exist in `uploads.id` |

They become **attachments** on the Documents tab below, downloadable and removable.

**This replaces `signed_agreement_upload_id`, `signed_loo_upload_id` and `other_documents`.** Those
three were never read - the API accepted them, returned success and discarded the files, so a
signed agreement uploaded through the Create Lease wizard was lost. Send `uploads` instead.

On update the list is the **complete set**: a document already on the lease and missing from it is
released. Omitting `uploads` entirely leaves the lease's documents alone, which is what an edit
form that does not show the Documents tab should do. The tab's own Add button appends through the
documents endpoint instead - do not send `uploads` from there.

### Lease documents endpoints

`GET /leases/{lease}` also returns `uploads` - the files attached to the lease itself, in the same
shape every other model with documents uses. On the list, it is opt-in with `?include=uploads`.
That key holds **only** attachments; the offer letter, the signed agreement and the rest come from
the documents endpoint below, which gathers them from the Loo and the application.


- `GET /leases/{lease}/documents`
- `POST /leases/{lease}/documents`
- `DELETE /leases/{lease}/documents/{document}`

Permissions: reading needs `view-lease`, adding and removing need `update-lease`. There are no
separate document permissions.

The list is not paginated - a lease's document count is small and the tab shows all of it.

Document object:

| Field | Type | Notes |
|---|---|---|
| `id` | integer | The **upload** id. This is what `DELETE` takes, and what identifies the file |
| `title` | string \| null | - |
| `file_name` | string | - |
| `type` | string | MIME type |
| `extension` | string | - |
| `size` | integer | Bytes |
| `source_url` | string | Where to download it. Points at the existing upload preview route |
| `source` | object | `{ value, label, color }` - where the document came from, for a chip |
| `context` | string \| null | What it belongs to, where that helps tell two apart: a Loo's reference, or an application document's type |
| `removable` | boolean | Whether `DELETE` will accept it. Only `attachment` is ever `true` |

`source.value` is one of:

| Value | Label | Comes from |
|---|---|---|
| `attachment` | Attachment | A file added to the lease through this endpoint |
| `offer-letter` | Offer letter | The Loo that was promoted into this lease |
| `signed-offer` | Signed offer | The tenant's signature on that Loo |
| `legal-fee-note` | Legal fee note | That Loo's fee note |
| `lease-agreement` | Signed lease agreement | The application the Loo was prepared from |
| `application-document` | Application document | The applicant's supporting documents. **Staff only** |

A lease owns almost none of its own paperwork, so most rows are gathered from the Loo that created
the lease, from the application behind it, and from any renewal or addendum drafted from the lease.
Those are read-only here: removing one would mean unpicking the record it actually belongs to.

#### Adding a document

Upload the file first through `POST /api/v1/settings/file-management/uploads`, then claim it:

`POST /leases/{lease}/documents`

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `uploads` | Yes | array | At least one upload id |
| `uploads.*` | Yes | integer | Must exist in `uploads.id` |

Send only the file you just uploaded - the endpoint **appends**. Documents already on the lease are
untouched, so there is no need to post the whole list back.

An upload is refused (`422`, keyed `uploads`) when it already belongs to another record, or when it
was uploaded by somebody else. Re-sending one already on this lease is accepted and changes
nothing.

The response carries the full document list back, so the tab can re-render from it.

#### Removing a document

`DELETE /leases/{lease}/documents/{document}` where `{document}` is the upload id.

The file is **released, not deleted** - it stays in `uploads` and simply stops belonging to the
lease. Only a row whose `removable` is `true` is accepted; anything else returns `422` keyed
`upload`.

## Related Docs

- `docs/frontend/app/property-management/lease-management/lease-deposits.md`
- `docs/frontend/app/property-management/lease-management/lease-escalations.md`
- `docs/frontend/app/property-management/lease-management/lease-opening-balances.md`

## Frontend Error Handling

Apply the shared rules in `docs/frontend/app/README.md`:

- Show backend `message` and field `errors` for `4xx`.
- Show generic fallback message for `5xx`.
