# Lease Applications API

Domain: `Property Management > Lease Management`

A lease application is a prospective tenant's request to lease space. Applicants
(tenants) submit and maintain their own applications; staff list, review, and
approve/reject them.

An application identifies one facility and the residential unit types requested
at that facility. The server derives company ownership from the facility, so
staff can retrieve the application under the correct company.

The applicant's documents are **not** part of the application payload. They live
in their own collection, `documents`, and are submitted one at a time through a
dedicated endpoint — see [Application Documents](#application-documents).
Guarantors are a nested resource — see [Guarantors](#guarantors).

## Surfaces

Staff (this domain):

`/api/v1/app/{company}/property-management/lease-management/lease-applications`

- `GET /lease-applications`
- `POST /lease-applications/invite` — send a prospective tenant the link to register and apply (see [Invite an applicant](#invite-an-applicant))
- `GET /lease-applications/{application}`
- `PUT/PATCH /lease-applications/{application}` — staff edit, while the application is open (see [Staff edit](#staff-edit))
- `DELETE /lease-applications/{application}`
- `PATCH /lease-applications/{application}/review`
- `GET /lease-applications/{application}/vetting-results` — staff-only vetting summary
- `POST /lease-applications/{application}/vet` — re-run vetting for one application
- `GET /lease-applications/{application}/spaces` — the spaces allocated to the application
- `POST /lease-applications/{application}/spaces` — allocate a space
- `PUT/PATCH /lease-applications/{application}/spaces/{applicationSpace}` — reprice one
- `DELETE /lease-applications/{application}/spaces/{applicationSpace}` — release one
- `GET /lease-applications/{application}/spaces/{facilitySpace}/component-options` — what a
  space may be billed, and what to charge

Applicant (tenant):

`/api/v1/tenant/lease-applications`

- `GET /lease-applications`
- `POST /lease-applications`
- `GET /lease-applications/{application}`
- `PUT/PATCH /lease-applications/{application}`
- `DELETE /lease-applications/{application}`
- `PUT/PATCH /lease-applications/{application}/documents/{documentType}`
- `GET|POST /lease-applications/{application}/guarantors`
- `GET|PUT|PATCH|DELETE /lease-applications/{application}/guarantors/{guarantor}`
- `PUT/PATCH /lease-applications/{application}/guarantors/{guarantor}/documents/{documentType}`

## List Applications

`GET .../lease-applications`

Supported query params:

- Filters:
  - `filter[search]`
  - `filter[applicant_type]` (`personal` | `business`)
  - `filter[status]` (see `LeaseApplicationStatus`)
  - `filter[user_id]`
  - `filter[facility_type_id]`
  - `filter[city_id]`
  - `filter[applicant_registered_country_id]`
  - `filter[application_submitted_at]`
  - `filter[proposed_start_date]`
  - `filter[reviewed_at]`
  - `filter[parking_required]`
  - `filter[generator_required]`
  - `filter[registration_type]`
  - `filter[is_renewal]`
  - `filter[lease_id]`
  - `filter[referral_source]`
  - `filter[created_at]`
- Includes: `include=guarantors`, `documents`, `documents.upload`, `facility`, `residentialUnitTypes`, `lease`
- Sort: `sort=id,created_at,application_submitted_at,proposed_start_date,reviewed_at`
- Pagination: `per_page`, `page`

## Invite an applicant

`POST .../lease-applications/invite`

Sends a prospective tenant the link to register and start an application. Staff-only;
requires `view-lease-application`. Nothing is created: the applicant has no account
until they register through the link.

| Field | Required | Type | Notes |
|---|---|---|---|
| `first_name` | Yes | string | Max 100 characters |
| `last_name` | Yes | string | Max 100 characters |
| `email` | Yes | string | A valid email address, max 191 characters |
| `phone` | No | string \| null | International format, e.g. `+254712345678` (6–15 digits, optional leading `+`) |

```json
{ "first_name": "Amina", "last_name": "Otieno", "email": "amina@example.com", "phone": "+254712345678" }
```

Response `200`:

```json
{ "data": { "message": "Application link sent to amina@example.com." } }
```

What happens:

- The link goes out by **email**, and by **SMS** and **WhatsApp** when a `phone` is
  given. Mail is sent through the company's mailer.
- The link is the register page with the applicant's details pre-filled:
  `{FRONTEND_URL}/auth/register?apply=lease&first_name=…&last_name=…&email=…&phone=…`.
  The frontend uses `apply=lease` to preselect the tenant portal and, when someone who
  is already signed in opens it, to tell them to use a private window or log out first.
- The email lists the steps: register, start the application and upload the documents,
  and that the financial document must be a monthly or annual income statement, not
  just a balance sheet or a bank statement.

Refused with `403` without `view-lease-application`. Missing or invalid fields are a `422`.

The same link without the applicant's details (`/auth/register?apply=lease`) can be
shared by hand; it needs no endpoint.

## Create Application

`POST /api/v1/tenant/lease-applications`

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `applicant_type` | Yes | string | `personal` or `business` |
| `applicant_name` | Yes | string | |
| `applicant_registered_country_id` | Yes | integer | Must exist in `countries.id` |
| `applicant_physical_address` | Yes | string | |
| `applicant_postal_address` | Yes | string | |
| `applicant_contact_email` | Yes | email | |
| `applicant_contact_phone` | Yes | string | |
| `applicant_tax_pin` | Yes | string | |
| `facility_type_id` | Yes | integer | Must exist in `facility_types.id` |
| `facility_id` | Yes | integer | Must exist in `facilities.id`. The server derives the application's company from this facility. |
| `city_id` | Yes | integer | Must exist in `cities.id` |
| `registration_type` | Yes | string | `national_id`, `business_license` or `passport` |
| `registration_number` | Yes | string | |
| `tax_pin` | Yes | string | |
| `region` | No | string | |
| `proposed_start_date` | No | date (`YYYY-MM-DD`) | When the applicant wants the tenancy to begin. Their own choice, and what the offer is priced and dated against. |
| `proposed_period_in_years` | No | integer | How long a tenancy the applicant is asking for. Pairs with the months below. |
| `proposed_period_in_months` | No | integer | The remainder of the proposed term, 0-11. |
| `bank_account_name` | No | string | Name on the applicant's own bank account. |
| `bank_account_number` | No | string | The applicant's own account number. |
| `bank_branch_id` | No | integer | Must exist in `bank_branches.id`. Resolved for display as `{branch.name}, {branch.bank.name}`. |
| `parking_required` | No | boolean | Defaults to `true` |
| `generator_required` | No | boolean | Defaults to `true` |
| `occupants` | No | integer | `0`–`100` |
| `space_size` | No | integer | `0`–`1,000,000` |
| `residential_unit_types` | Yes, except on a renewal | array | One or more requested unit types, each with a facility-level `FacilityResidentialUnit` `id` and a positive integer `quantity`. Each unit type must belong directly to `facility_id`. |
| `applicant_postal_code` | No | string \| null | The postal code that goes with `applicant_postal_address`. Printed by `{{tenant_postal_code}}`. |
| `is_renewal` | No | boolean | Defaults to `false`. `true` asks to renew an existing lease — see [Renewal applications](#renewal-applications). |
| `lease_id` | Required when `is_renewal` is `true` | integer | The lease being renewed. Must exist, belong to the same company as `facility_id`, and be held by the applicant. Must be absent or `null` when `is_renewal` is `false`. |
| `fit_out_period_months` | No | integer | `0`–`120`. Months the tenant has to fit out before rent starts. Feeds the fit-out tags on the offer. |
| `nature_of_business` | No | string | What the applicant will use the premises for. Printed by `{{nature_of_business}}`. |
| `billing_cycle` | No | string | `monthly`, `quarterly`, `biannual` or `annually`. How often the tenancy will be billed. Copied onto the lease when the offer is promoted. |
| `industry` | No | string | The applicant's industry, as free text. |
| `referral_source` | No | string | How the applicant heard of the property: `staff`, `agent`, `social_media`, `website`, `billboard` or `other` |
| `referrer_name` | No | string | Who referred them, where there was a person |
| `referrer_email` | No | email | |
| `referral_notes` | No | string | |
| `date_of_birth` | No | date (`YYYY-MM-DD`) | Personal applicants |
| `gender` | No | string | `male`, `female` or `other` |
| `marital_status` | No | string | `single`, `married`, `divorced`, `widowed` or `other` |
| `children_count` | No | integer | `0`–`50` |
| `occupation` | No | string | |
| `position` | No | string | The applicant's position at work |
| `citizenship` | No | string | |
| `current_residence` | No | string | Where the applicant lives now |
| `previous_landlord_name` | No | string | The applicant's current or most recent landlord |
| `previous_landlord_address` | No | string | |
| `previous_landlord_city` | No | string | |
| `previous_landlord_email` | No | email | |
| `previous_landlord_phone` | No | string | |
| `previous_landlord_contact_person` | No | string | |
| `status` | No | string | Defaults to `pending` |
| `comments` | No | string | |
| `application_submitted_at` | No | date | Defaults to now |

`created_at` is returned on the show and list responses in the usual
`{ raw, formatted, diff }` shape — when the record was made, as against
`application_submitted_at`, which is when the applicant submitted it.

`reviewed_at` comes back in the same shape, beside `reviewed_by` — when the decision
was taken. **Null on anything decided before 6 Oct 2026:** the column has existed as
long as the table but only the EPMAS import ever wrote it, so applications reviewed
through the app recorded the reviewer and not the moment. Nothing can recover it for
those, so keep the date hidden when it is null rather than falling back to
`updated_at`, which moves whenever anything touches the row.

Example:

```json
{
  "applicant_type": "business",
  "applicant_name": "Acme Trading Ltd",
  "applicant_registered_country_id": 1,
  "applicant_physical_address": "1 Market Street",
  "applicant_postal_address": "P.O. Box 100",
  "applicant_contact_email": "leases@acme.test",
  "applicant_contact_phone": "+254700000000",
  "applicant_tax_pin": "A001234567B",
  "facility_type_id": 2,
  "facility_id": 12,
  "city_id": 1,
  "registration_type": "business_license",
  "registration_number": "BRN-12345",
  "tax_pin": "A001234567B",
  "proposed_start_date": "2027-03-01",
  "bank_account_name": "Acme Trading Ltd",
  "bank_account_number": "0123456789",
  "bank_branch_id": 44,
  "residential_unit_types": [
    {
      "id": 345,
      "quantity": 5
    }
  ]
}
```

Every residential unit type must belong directly to facility `12`. The server
stores the requested quantity on the application/unit-type relationship.

On submission the server also allocates real units against that request where it
can — see [Application Spaces](#application-spaces). The request itself is never
altered by that, so a request for five units that only found three free still
reads as five here.

The application no longer carries any document upload fields. Create the
application first, then submit each document to
[its own endpoint](#submit-a-document).

## Update Application

`PUT/PATCH /api/v1/tenant/lease-applications/{application}`

Accepts the same body as create. `residential_unit_types` replaces the current
unit-type requests as a full sync, and the application company is refreshed
from the selected `facility_id`. Documents are not part of this payload.

### Staff edit

`PUT/PATCH /api/v1/app/{company}/property-management/lease-management/lease-applications/{application}`

Staff correct an application on the applicant's behalf with the same body as the
applicant's update; the same rules apply to each field. `status` is never read from
this payload. One difference: `residential_unit_types` is optional here for every
application. Leaving it out keeps the recorded unit types and the space allocation as
they are, which is what an application with allocated spaces but no unit types needs. Use [Review Application](#review-application) to decide one.

**Who and when.** It needs `update-lease-application`, the application's property must
be allocated to the caller, and the application must be **open** (`pending` or
`review`). An `approved` or `rejected` application returns `403`. The application's
`permissions.staffUpdate` flag says whether the caller may edit it right now; show
the Edit action only when it is `true`. It is the same window
[spaces](#when-the-allocation-may-be-changed) and [documents](#submit-a-document)
already follow: the decision was made against what the application said.

**Effect.** As with the applicant's own edit, saving sends the application back to
`pending` for the reviewer to look at again. Any earlier review comment is cleared, and
the review task is raised again for the reviewer role (see
[Review Application](#review-application)). Every change is recorded in the activity log.

Response `200`: `{ "message": "Lease Application updated successfully.", "data": { ...the application } }`

## Renewal applications

A renewal application asks to renew a lease the applicant already holds. It is an
ordinary application with two extra fields:

| Field | Type | Notes |
|---|---|---|
| `is_renewal` | boolean | `true` for a renewal |
| `lease_id` | integer \| null | The lease being renewed. Required when `is_renewal` is `true` |

`lease_id` must belong to the same company as the application's `facility_id`, and the
lease must be held by the applicant (`lease.user_id` = the application's `user_id`).
Otherwise the request fails with `422` on `lease_id`.

`residential_unit_types` is optional on a renewal: the units being renewed are the
lease's, and the property may have nothing listed for application. A renewal sent
without unit types gets no automatic space allocation; staff allocate its spaces. The
staff edit may resend `is_renewal` and `lease_id` unchanged.

A renewal is reviewed exactly like any other application. The Letter of Offer
generated from it has `type` `renewal` rather than `new lease`, and is prepared from
the application, so every applicant-side tag prints from the application as it does for
a new lease.

The response carries `is_renewal`, `lease_id`, and `lease` (`{ id, ... }`). `lease` is always loaded on `GET .../{application}` (both surfaces) and is opt-in on the list with `include=lease`.

> **Not yet:** a signed renewal offer cannot be promoted. `POST .../loos/{loo}/promote`
> still refuses anything but a `new lease` offer, so a renewal stops at `accepted`.
> Promotion for renewals is a separate piece of work.

## Applicant profile

Besides the fields the offer prints, an application can carry the applicant's
personal details, their current or previous landlord, and how they were referred. All
of these are optional, all are written on create and update, and all are returned on
**both** surfaces. Nothing in the workflow depends on them.

| Group | Fields |
|---|---|
| Personal | `date_of_birth`, `gender`, `marital_status`, `children_count`, `occupation`, `position`, `citizenship`, `current_residence` |
| Business | `nature_of_business`, `industry` |
| Previous landlord | `previous_landlord_name`, `_address`, `_city`, `_email`, `_phone`, `_contact_person` |
| Referral | `referral_source`, `referrer_name`, `referrer_email`, `referral_notes` |
| Tenancy | `fit_out_period_months`, `billing_cycle` |

Every one of them accepts `null` on create and update, which clears it.

`gender`, `marital_status`, `referral_source` and `billing_cycle` come back as
`{ "value", "color" }`; `date_of_birth` as `raw` / `formatted` / `diff`.

```json
{
  "nature_of_business": "Retail pharmacy",
  "industry": "Healthcare",
  "fit_out_period_months": 3,
  "billing_cycle": { "value": "quarterly", "color": "info" },
  "referral_source": { "value": "agent", "color": "info" },
  "referrer_name": "Jane Agent",
  "previous_landlord_name": "Acme Properties",
  "date_of_birth": null,
  "gender": null
}
```

## Proposed start date

`proposed_start_date` is the calendar day the applicant asks the tenancy to begin.
It is stored as a `DATE` — no time of day — and is returned in the usual shape:

```json
"proposed_start_date": {
  "raw": "2027-03-01",
  "formatted": "01 Mar, 2027",
  "diff": "6 months from now"
}
```

Visible on **both** surfaces, written on one: the applicant chooses it on their
own form (create and update), and staff read it back — and can sort and filter
the application list by it — when scheduling and pricing the offer. It is
optional, so it can be `null`.

It is not validated as a future date. Staff editing an older application should
not be forced to move a start date that has since passed.

Downstream, this is the source of the Letter of Offer's `{{start_date}}` tag for
`new lease` offers — see the [LOO workflow](loo/README.md).

## Proposed term

`proposed_period_in_years` and `proposed_period_in_months` are how long a tenancy the
applicant is asking for. Both optional, both plain integers, and they pair with
`proposed_start_date` to give an application the same start-plus-term shape a lease
already has.

Written by the applicant on their own form; read on both surfaces. The Letter of Offer
copies them into its own `lease_term_years`/`lease_term_months` at generation, and
staff may then move the offered term off the request — the LOO's value wins wherever
it is set. Together with the escalation schedule below, this is what closes the LOO's
rent breakdown: without an end date there is no last period to escalate to.

## Proposed end date

`proposed_end_date` is read-only and **computed**, not stored and not accepted. It is the
proposed start plus the proposed term, less a day:

```json
"proposed_end_date": {
  "raw": "2028-02-29",
  "formatted": "29 Feb, 2028",
  "diff": "1 year from now"
}
```

A one-year tenancy from 1 Mar 2027 ends on 29 Feb 2028, not 1 Mar — so the next term
begins the following day rather than overlapping this one. This is the same rule a lease
uses for its own `end_at`, shared in code, so the date shown while an application is being
approved is the date the raised lease carries.

`null` until both halves are answered: no `proposed_start_date`, or no term at all
(`proposed_period_in_years` and `proposed_period_in_months` both unset or zero).

Three things worth knowing if you offer an end-date field that works out the term:

- The term is **years and months only**. An end date that is not a whole number of months
  from the start cannot be stored, so it will move to one that is.
- Month ends **overflow rather than clamp**. One month from 31 Jan is 3 Mar, not 28 Feb,
  because 31 Feb does not exist and the surplus days run on — so a one-month term from
  31 Jan 2026 ends 2 Mar 2026.
- Years are added before months, which matters at those same month ends.

Posting an end date does nothing: on a lease, `end_at` is overwritten by the computed
value; on an application there is no such field.

## Escalations

`lease-applications/{application}/escalations` is the escalation schedule the offer is
priced against — one row per lease component, saying by what percentage that component
escalates and how often.

The application-side twin of a lease's own `lease_escalations`, with the same columns,
so promoting an offer to a lease copies these rows rather than translating them.
Nothing processes them on a timer: they exist to price an offer, and only become live
escalations once there is a lease to escalate.

### Who may read and write them

| | Staff (`/api/v1/app/...`) | Applicant (`/api/v1/tenant/...`) |
|---|---|---|
| Read | Yes, on the escalations endpoints | Yes, as `escalations` on their own application |
| Write | Yes, with `update-lease-application` | **Never** — no tenant route exists |

Staff-written because which rates escalate, and by how much, is a letting decision
rather than something an applicant states about themselves. Applicant-readable because
the same rates are worked through period by period on the offer they are asked to
sign — withholding the inputs would only make that document harder to check.

Unlike the spaces surface, there is **no open-application window** on writing them. A
LOO is drafted *after* an application is approved, so a schedule that locked on
approval could never be set in time to price one.

### Fields

| Field | Required | Type | Notes |
|---|---|---|---|
| `lease_component_id` | Yes | integer | One schedule per component — a second is refused with a 422 |
| `start_at` | Yes | date (`YYYY-MM-DD`) | The **first escalation date**, not the tenancy start |
| `cycle` | Yes | `days` \| `weeks` \| `months` \| `years` | |
| `period` | Yes | integer ≥ 1 | How many cycles between escalations |
| `rate` | Yes | decimal | Percent, compounding — 10% twice is ×1.21, not ×1.2 |
| `next_due` | No | date | Defaults to `start_at`; only exists to keep the row copyable onto `lease_escalations` |

`start_at` being the first escalation date is the thing to get right: a schedule
anchored on the tenancy's own start date does not escalate the first period — it walks
forward and first bites one cycle in, which is what "escalates annually" is normally
taken to mean.

### Endpoints

```
GET    /lease-applications/{application}/escalations
POST   /lease-applications/{application}/escalations
GET    /lease-applications/{application}/escalations/{escalation}
PUT    /lease-applications/{application}/escalations/{escalation}
DELETE /lease-applications/{application}/escalations/{escalation}
```

Filter on `lease_component_id` and `cycle`; sort by `id`, `lease_component_id`,
`start_at`, `rate` or `created_at`, defaulting to `start_at`.

```json
{
  "lease_component_id": 3,
  "start_at": "2027-10-01",
  "cycle": "years",
  "period": 1,
  "rate": 7.5
}
```

Every write reprices the rent breakdown on any LOO from this application that is still
in `draft` or `pending_approval`. Approved, sent and accepted offers are left alone —
that document is in a tenant's hands, and a schedule shifting underneath it is exactly
what the editable-status rule exists to prevent. See
[the LOO rent schedule](loo/README.md#the-rent-schedule).

## Applicant bank details

`bank_account_name`, `bank_account_number` and `bank_branch_id` are the
**applicant's own** account — captured at application time and printed on the
offer's banking tags. They are not the landlord's payment-instruction account,
and nothing bills against them.

All three are optional and independent; `bank_branch_id` resolves through
`bank_branches` to its bank, which is how the `{{bank}}` tag renders
`{branch.name}, {branch.bank.name}`.

## Application Documents

`documents` is where every applicant document lives. Each entry pairs one upload
with one `document_type` and carries its own verification state. An application
holds **at most one document per type**.

The allowed types come from `App\Enums\LeaseApplicationDocumentType`, not from
the database schema, so the set can grow without a migration:

| `document_type` | Meaning |
|---|---|
| `id` | Applicant's identity / registration document (national ID, passport or business licence) |
| `tax_pin` | Tax PIN certificate |
| `tcc` | Tax compliance certificate |
| `vat_exemption` | VAT exemption certificate |
| `financial_statement` | Financial statement — feeds the income-to-rent score |
| `other` | Anything else the applicant attaches |

### Reading documents

| Field | Type | Notes |
|---|---|---|
| `id` | integer | |
| `document_type` | object | `{ "value", "color" }` — see the table above |
| `status` | object | `{ "value", "color" }`. One of `pending` (`warning`), `verified` (`success`), `failed` (`danger`) |
| `reason` | string \| null | Why verification failed, or the note recorded against the outcome |
| `upload` | object | `UploadResource`, present when `documents.upload` is loaded |
| `verified_at` | object \| null | `raw` / `formatted` / `diff` |
| `created` | object | `raw` / `formatted` / `diff` |

Where they appear:

- `GET .../lease-applications?include=documents,documents.upload` — opt in on the list endpoint
- `GET .../lease-applications/{application}` — always loaded, with uploads
- Create and update responses — always loaded, with uploads

```json
{
  "documents": [
    {
      "id": 41,
      "document_type": { "value": "tax_pin", "color": "primary" },
      "status": { "value": "verified", "color": "success" },
      "reason": null,
      "upload": { "id": 902, "file_name": "kra-pin.pdf" },
      "verified_at": { "raw": "2026-08-27T09:14:00.000000Z", "formatted": "27 Aug, 2026", "diff": "2 hours ago" }
    },
    {
      "id": 42,
      "document_type": { "value": "financial_statement", "color": "primary" },
      "status": { "value": "pending", "color": "warning" },
      "reason": null,
      "upload": { "id": 903, "file_name": "accounts-2025.pdf" },
      "verified_at": null
    }
  ]
}
```

The raw extraction and cross-check output behind a verification is stored
server-side and is never returned by the API — clients get the outcome
(`status`) and the explanation (`reason`) only.

`financial_statement` is not pass/fail verified. It feeds the income-to-rent
score rather than a verification, so its `status` stays `pending`.

> **Current behaviour:** automatic verification is not wired up yet, so every
> document stays `pending` and `reason`/`verified_at` stay `null`. Build against
> all three statuses — `verified` and `failed` start appearing once the
> verification pipeline lands, with no API change.

### Submit a Document

`PUT|PATCH /api/v1/tenant/lease-applications/{application}/documents/{documentType}`

`{documentType}` is one of the `document_type` values above. An unknown value
returns `404`.

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `upload_id` | Yes | integer | Must exist in `uploads.id` |

```json
{ "upload_id": 902 }
```

Response `200`:

```json
{
  "message": "Lease Application document submitted successfully.",
  "document": {
    "id": 41,
    "document_type": { "value": "tax_pin", "color": "primary" },
    "status": { "value": "pending", "color": "warning" },
    "reason": null,
    "upload": { "id": 902, "file_name": "kra-pin.pdf" },
    "verified_at": null
  }
}
```

This one endpoint both submits and replaces — there is no separate resubmit
call, and no `DELETE`:

- **First submission** creates the document with `status` `pending`.
- **Replacing** it with a *different* `upload_id` restarts verification: the
  status resets to `pending` and the previous `reason`, result and `verified_at`
  are cleared.
- **Re-sending the same `upload_id`** is a no-op, so an already verified document
  is never reset by a repeated call.
- **A new `financial_statement`** re-runs the income-to-rent score instead of a
  verification — see [Vetting fields](#vetting-fields-staff-only). The score is
  staff-only, so nothing about it comes back in this response.

Requires the `update` ability on the application — the same authorisation as
editing the application itself, so an applicant cannot submit onto someone
else's application.

## Guarantors

Guarantors are a nested resource on the application. Unlike applicant documents,
a guarantor's documents are **fixed fields on the guarantor itself** — there is
no document-type collection here.

`/api/v1/tenant/lease-applications/{application}/guarantors`

| Method | Path | Ability required on the application |
|---|---|---|
| `GET` | `/guarantors` | `view` |
| `POST` | `/guarantors` | `update` |
| `GET` | `/guarantors/{guarantor}` | `view` |
| `PUT/PATCH` | `/guarantors/{guarantor}` | `update` |
| `DELETE` | `/guarantors/{guarantor}` | `update` |

`GET /guarantors` is paginated. Guarantors also come back on the application
itself via `include=guarantors`.

### Create / Update Guarantor

`POST /guarantors` and `PUT/PATCH /guarantors/{guarantor}` take the same body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `name` | Yes | string | |
| `email_address` | Yes | email | |
| `phone_number` | Yes | string | |
| `registration_type` | Yes | string | `national_id`, `business_license` or `passport` |
| `registration_number` | Yes | string | |
| `registration_upload_id` | Yes | integer | Scan of the registration document. Must exist in `uploads.id` |
| `id_upload_id` | No | integer | Guarantor's ID document. Must exist in `uploads.id` |
| `tax_pin_upload_id` | No | integer | Guarantor's tax PIN certificate. Must exist in `uploads.id` |
| `financial_statement_upload_id` | No | integer | Feeds the income-to-rent score. Must exist in `uploads.id` |
| `postal_address` | No | string \| null | The guarantor's postal address (P.O. Box) |
| `city` | No | string \| null | |

```json
{
  "name": "Jane Guarantor",
  "email_address": "jane@acme.test",
  "phone_number": "+254700000000",
  "registration_type": "national_id",
  "registration_number": "12345678",
  "registration_upload_id": 801,
  "id_upload_id": 802,
  "tax_pin_upload_id": 803
}
```

### Guarantor Response

| Field | Type | Notes |
|---|---|---|
| `id` | integer | |
| `name` | string | |
| `email_address` | string \| null | |
| `phone_number` | string | |
| `registration_type` | string \| null | `national_id`, `business_license` or `passport` |
| `registration_number` | string \| null | |
| `postal_address` | string \| null | |
| `city` | string \| null | |
| `registration_upload` | object \| null | `UploadResource` |
| `id_upload` | object \| null | `UploadResource` |
| `id_status` | object \| null | `{ "value", "color" }` — `pending` / `verified` / `failed` |
| `id_reason` | string \| null | Why ID verification failed |
| `tax_pin_upload` | object \| null | `UploadResource` |
| `tax_pin_status` | object \| null | `{ "value", "color" }` — `pending` / `verified` / `failed` |
| `tax_pin_reason` | string \| null | Why tax PIN verification failed |
| `financial_statement_upload` | object \| null | `UploadResource` |

```json
{
  "id": 7,
  "name": "Jane Guarantor",
  "email_address": "jane@acme.test",
  "phone_number": "+254700000000",
  "registration_type": "national_id",
  "registration_number": "12345678",
  "registration_upload": { "id": 801, "file_name": "national-id.pdf" },
  "id_upload": { "id": 802, "file_name": "id-front.png" },
  "id_status": { "value": "verified", "color": "success" },
  "id_reason": null,
  "tax_pin_upload": { "id": 803, "file_name": "guarantor-pin.pdf" },
  "tax_pin_status": { "value": "failed", "color": "danger" },
  "tax_pin_reason": "PIN does not match the applicant name",
  "financial_statement_upload": null
}
```

`email_address`, `registration_type`, `registration_number` and `registration_upload`
can be `null` on a guarantor recorded before documents were collected for guarantors
(for example, one carried over from the previous system). Writes still require them, so
any edit through this endpoint has to supply them.

`id_status`/`id_reason` and `tax_pin_status`/`tax_pin_reason` are **read-only** —
they are written by the verification pipeline, not by the client, and are `null`
until it has run. Only `id` and `tax_pin` are verified; the guarantor's
`financial_statement` feeds the income-to-rent score instead and has no status.

### Submit a Guarantor Document

`PUT|PATCH /api/v1/tenant/lease-applications/{application}/guarantors/{guarantor}/documents/{documentType}`

The guarantor equivalent of [Submit a Document](#submit-a-document), and it
behaves the same way — one endpoint that both attaches and replaces.

`{documentType}` is one of `id`, `tax_pin` or `financial_statement`. `tcc` and
`vat_exemption` are asked of the applicant only, so passing either — or any
unknown value — returns `404`.

Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `upload_id` | Yes | integer | Must exist in `uploads.id` |

Response `200` returns the full guarantor:

```json
{
  "message": "Guarantor document submitted successfully.",
  "guarantor": {
    "id": 7,
    "id_upload": { "id": 802, "file_name": "id-front.png" },
    "id_status": { "value": "pending", "color": "warning" },
    "id_reason": null
  }
}
```

Replacing `id` or `tax_pin` with a different `upload_id` resets that field's
`_status` to `pending` and clears its `_reason`, then re-runs verification.
Replacing `financial_statement` re-runs the income-to-rent score instead — the
guarantor has no `financial_statement_status`, and the resulting score is
staff-only, so nothing about it comes back here.

The same two checks as the guarantor CRUD endpoints apply: the applicant must
own the application and it must still be open to them, and the guarantor named
in the path must belong to that application — pairing your own application with
someone else's guarantor is rejected.

## The offers on an application

Every row of the staff list, and the single application, carries the offers prepared from
it under `loos` — enough to link to each one and show where it stands:

```json
"loos": [
  { "id": 31, "our_ref": "LOO/2026/002", "status": { "value": "sent", "color": "info" } },
  { "id": 28, "our_ref": "LOO/2026/001", "status": { "value": "declined", "color": "danger" } }
]
```

**Newest first**, and `[]` when none have been prepared. Always loaded — no `?include`
needed, so a list page costs one request rather than one per row.

It is a list because an application can hold several: a declined or withdrawn offer stays
on the record and another is prepared beside it. Which one counts as current is yours to
decide — the API does not pick, so changing your mind is not an API change.

Three keys and no more, on purpose. A page holds 25 rows, each able to hold several
offers, and the full offer resource carries clause text, tag values and spaces. Fetch
`loos/{loo}` when the user opens one.

Whether to show the action at all is the plain `view-loo` permission, read once from
`/me/permissions`. There is no per-offer flag and none is needed: reading an offer needs
`view-loo` **and** allocation to its property, an offer takes its property from the
application it was prepared from, and this list only ever returns applications on
properties you are allocated to. So for any row you can see, the allocation half is
already satisfied.

**Staff only.** The field is absent from the applicant surface entirely, whatever an
offer's status — an applicant may read an offer only once it has cleared approval, and
anything short of that is internal drafting.

## Documents and guarantors, from the staff side

Both were reachable only through the applicant's own portal, behind tenant-account
middleware, so staff were refused before any permission was read. The staff routes mirror
them one for one:

```
PUT    …/lease-applications/{application}/documents/{document_type}        { upload_id }
GET    …/lease-applications/{application}/guarantors
POST   …/lease-applications/{application}/guarantors
PUT    …/lease-applications/{application}/guarantors/{guarantor}
DELETE …/lease-applications/{application}/guarantors/{guarantor}
PUT    …/lease-applications/{application}/guarantors/{guarantor}/documents/{document_type}
```

Responses are the tenant shapes — `data.document` and `data.guarantor` — so a page can
refresh from the answer.

**When:** while the application is **pending or under review**. Approved and rejected close
both, the same window as spaces. Read `permissions.submitDocument` and
`permissions.manageGuarantors`; both mean *update permission, application still open*.

**A staff upload is the same act as the applicant's**, because it runs the same code: the
document resets to `pending`, the previous result and reason are cleared, verification is
re-queued, a financial statement rescores instead of re-verifying, and the reviewers are
notified. Re-sending an upload the document already has is a no-op, so a verified document
is not reset for nothing.

**It does not send the application back to pending.** Only a staff *edit* of the
application itself does that. The reviewers are notified of the change, which is the part
that matters.

**The applicant is notified in-app** when somebody other than them attaches a document or
adds, changes or removes a guarantor. Keyed on who acted, not which route was used, so the
applicant doing it themselves triggers nothing.

## Application Spaces

Staff only. `/api/v1/app/{company}/property-management/lease-management/lease-applications/{application}/spaces`

An application states its request as `residential_unit_types` — how many of each
type the applicant wants. That is what they chose from, but it is not something
anyone can price, hold or sign. **Spaces** are the other half: which actual units
they have been allocated, and what each will cost per month.

The two coexist and are never reconciled against each other. `residential_unit_types`
stays exactly as the applicant submitted it, so a shortfall between what was asked
for and what was free stays visible rather than being quietly rewritten.

Allocating a space marks it `under consideration`, which takes it out of
`is_available_for_application` for everyone else. Approving or rejecting the
application releases it again.

### Automatic allocation

On submission the server allocates free units of each requested type and prices them
from the property's indicative rates, so a reviewer opens the application onto a
concrete, costed allocation rather than a list of unit types.

It is provisional in three ways, all deliberate:

- **Best effort.** Five requested with three free allocates three. The submission
  does not fail, and the request still reads as five.
- **Not exclusive.** Two applications submitted for the last free unit both get it.
  `under consideration` means someone is looking at it, not that it is taken —
  choosing between competing applicants is a letting decision, not a queue.
- **Overridable.** These endpoints exist to correct it. Nothing re-runs the
  allocation afterwards, so editing an application's requested unit types will not
  undo staff's work.

Only spaces mapped to a requested unit type
(`facility_spaces.facility_residential_unit_id`) are allocated automatically.
Commercial space-let applications, parking and signage are allocated by hand.

### An application cannot be approved without one

Approval is refused with `422` and a `spaces` key while nothing is allocated:

```json
{ "errors": { "spaces": ["Allocate at least one space before approving this application."] } }
```

Approval closes the allocation for good, and the offer drafted from the application grants
exactly the units allocated to it — so an application approved with none produces an offer
covering no premises, with no rent to schedule and nothing for a lease to bill.

Rejecting and sending back for review are unaffected. An application nobody allocated
anything to is often precisely the one being rejected.

### When the allocation may be changed

Only while the application is **open** — `pending` or `review`. Once it is `approved`
or `rejected` every write here returns `403`: the decision was made against a
particular allocation at a particular price, and moving spaces underneath it
afterwards would silently invalidate it. This is the same rule that applies to
[documents](#submit-a-document).

Reading stays available on a closed application.

Writes need `update-lease-application`; reads need `view-lease-application`. Unlike
`view`, there is no ownership fallback — which units an applicant gets and what they
pay is a letting decision, not something the applicant fills in about themselves.
The `spaces` key is never serialised on the tenant surface.

`permissions.manageSpaces` on the application says whether the caller may write here
right now. It is `false` for either reason — the application is decided, **or** the
caller's role does not hold `update-lease-application` — and the flag does not say
which. So don't word a disabled allocation as "this application has been decided":
read `status` for that, and keep the permission copy about what the user can do.

> **Note:** `update-lease-application` was seeded as a tenant-only permission on
> installs created before 6 Oct 2026, so it never appeared on the staff permission
> list and no staff role held it. `manageSpaces` was therefore `false` for every
> staff user on every application, whatever its status. The same permission gates
> `manageEscalations` and `staffUpdate`, so those were refused too. The catalogue is
> fixed; existing installs need it granted in access management.

### Component options (prefill)

`GET .../lease-applications/{application}/spaces/{facilitySpace}/component-options`

Call this when staff pick a space, to populate the component picker. `{facilitySpace}`
is a facility space id — the space being priced, before it is attached.

```json
{
  "data": {
    "facility_space_id": 88,
    "size": 10,
    "is_lettable": true,
    "components": [
      { "lease_component_id": 1, "name": "Rent",           "cost_per_space_unit": 100, "amount": 1000, "tax_id": 2 },
      { "lease_component_id": 2, "name": "Service Charge", "cost_per_space_unit": 20,  "amount": 200,  "tax_id": 2 },
      { "lease_component_id": 10, "name": "Promotion Fee", "cost_per_space_unit": null, "amount": null, "tax_id": 2 }
    ]
  }
}
```

`size` is the multiplier behind every `amount` (`amount = cost_per_space_unit × size`),
returned so the frontend can recompute a total live as staff edit a rate instead of
round-tripping for it.

**Which components are offered** — only components that are `active` **and** flagged
`is_autobilled`, narrowed by what kind of space it is:

| Space | Components offered |
|---|---|
| `is_parking` | the parking fee |
| `is_signage` | the signage fee |
| `space_type` = `leasable` | **every** autobilled component except the parking and signage fees — rent, service charge, and any other the company bills each cycle (a promotion fee, say) |
| `space_type` = `landlord` or `common` | none — `is_lettable` is `false` and the space cannot be attached |

An application allocates what the tenant will be billed automatically, and nothing else.
The rows are stored by `lease_component_id` and carried through the LOO onto the lease, so
whatever is offered here is what the lease is generated with. A component that is not
autobilled — water and electricity, which are billed from meter readings — is never offered,
and submitting one is a `422`.

**Where the suggested cost comes from.** All of these are net of tax.

| Component | Source |
|---|---|
| Rent | `indicative_rent_per_unit` on the space's residential unit type, else the facility's `indicative_rent_rate_per_space_unit` |
| Service charge | `indicative_service_charge_per_unit`, else the facility's `indicative_service_charge_rate_per_space_unit` |
| Parking fee | the facility's open or closed lot fee, chosen by the space's `parking_type` |
| Signage fee | **none exists** — `cost_per_space_unit` and `amount` come back `null` and staff enter the figure |
| Any other autobilled component | **none exists** — `cost_per_space_unit` and `amount` come back `null` and staff enter the figure. Auto-allocation on submission stores it at `0`, a visible prompt to price it |

A residential unit type quotes a rate for the whole unit, not per space unit, so it is
divided by `size` on the way out — multiplying it back by `size` returns the figure the
property actually quotes. A `null` cost anywhere else means the property has no rate
recorded for that component; the field simply starts empty.

Everything here is a suggestion. What gets stored is whatever staff submit.

### List allocated spaces

`GET .../lease-applications/{application}/spaces`

- Filters: `filter[facility_space_id]`
- Sort: `sort=id,facility_space_id,created_at` (default `id`)
- Pagination: `per_page`, `page`

```json
{
  "data": [
    {
      "id": 14,
      "facility_space_id": 88,
      "facility_space": { "id": 88, "name": "Unit A1", "size": 10, "...": "..." },
      "amount": 1200,
      "components": [
        {
          "id": 31,
          "lease_component_id": 1,
          "lease_component": { "id": 1, "name": "Rent" },
          "cost_per_space_unit": "100.00000",
          "amount": "1000.00000",
          "tax_id": 2,
          "tax": { "id": 2, "name": "VAT Standard", "value": "0.16" }
        }
      ]
    }
  ]
}
```

`amount` on the space is the sum of its components' amounts — what the space costs per
month, net of tax. It is only present once `components` is loaded, so it is never a
misleading zero.

The same shape appears as `spaces` on the staff `GET .../lease-applications/{application}`
response, and via `?include=applicationSpaces` on the staff list.

### Allocate a space

`POST .../lease-applications/{application}/spaces`

| Field | Required | Type | Notes |
|---|---|---|---|
| `facility_space_id` | Yes | integer | Must exist, belong to the application's `facility_id`, be lettable, and be available |
| `components` | Yes | array | At least one entry |
| `components[].lease_component_id` | Yes | integer | Must be one of the components [on offer for that space](#component-options-prefill); `distinct` within the payload |
| `components[].cost_per_space_unit` | Yes | numeric | `≥ 0`, **excluding tax** |
| `components[].tax_id` | No | integer\|null | Defaults to the component's own tax. Send `null` for no tax; omit the key to inherit |

```json
{
  "facility_space_id": 88,
  "components": [
    { "lease_component_id": 1, "cost_per_space_unit": 100, "tax_id": 2 },
    { "lease_component_id": 2, "cost_per_space_unit": 20 }
  ]
}
```

`amount` is **not** accepted — it is computed as `cost_per_space_unit × size` so a
client cannot submit a total that does not follow from the rate beside it.

Prices are held net of tax throughout. `tax_id` is carried for reference and is applied
downstream at invoice time; it never changes the stored figures. Overriding it on one
row does not touch the component's global tax.

`422` with a `facility_space_id` error when the space is on another property, is
landlord or common space, is not available, or is already on this application.
`422` with a `components` error when a component is not billable on that space —
rent on a parking lot, for instance.

### Reprice a space

`PUT/PATCH .../lease-applications/{application}/spaces/{applicationSpace}`

Accepts `components` only, and **replaces the set outright** — the same full-sync
convention as `residential_unit_types`. Dropping a component means leaving it out of
the payload, not a second call.

The space itself cannot be moved; release it and allocate the other one instead.

### Release a space

`DELETE .../lease-applications/{application}/spaces/{applicationSpace}`

Removes the allocation and its component pricing, and returns the space to whatever
its leases say it is.

## Review Application

`PATCH .../lease-applications/{application}/review`

Staff-only. Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `status` | Yes | string | Target `LeaseApplicationStatus` |
| `comments` | No | string | Reviewer notes |
| `signed_agreement_upload_id` | No | integer | Must exist in `uploads.id` |

Approving or rejecting records the review outcome for the application.

### Who is asked to act

Each step raises a pending task for the people the company settings name:

| Application status | Task | Goes to |
|---|---|---|
| `pending` | Review lease application | Holders of the **Lease Application Reviewer Role** (`lease_application_reviewer_role_id`) **on the application's property** |
| `pending`, [returned](#return-an-approved-application) | Re-review returned lease application | The person who approved it (`reviewed_by`), else the reviewer-role holders as above |
| `approved` | Generate Letter of Offer | Holders of the **Letter of Offer Generation Role** (`loo_generation_role_id`) |

The two settings differ in scope:

- **The reviewer role is a property role** (`enforce_on_facility` true). The task goes
  only to whoever holds it on the property applied for.
- **The LOO generation role can be any staff role.** A property role works as above. A
  company-wide role (`enforce_on_facility` false) sends the task to every active member
  of the company who holds it, whatever the property. **Agency**, the default, is
  company-wide: agency staff work across every property.

If nobody qualifies, the task falls back to everyone holding any role on the property.
The applicant never gets the task.

The setting's options list follows the scope: the reviewer setting offers
`access-management/roles?filter[enforce_on_facility]=1`, and the LOO generation setting
offers every app staff role (`access-management/roles?filter[user_group_id]={app group id}`).
Saving a role outside those lists is refused with `422`.

## Return an approved application

`PATCH .../lease-applications/{application}/return`

Sends an approved application back to its reviewer when it is not fit for a Letter of
Offer yet, for example because a document is wrong or the pricing needs another look.
Staff-only. Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `reason` | Yes | string | Max 1000 characters. Shown to the reviewer in the task, the email and the SMS |

```json
{ "reason": "The KRA PIN certificate is for a different company. Please confirm before we draft the offer." }
```

What happens:

- The application goes back to **`pending`**. It is open again, so staff can edit it,
  re-allocate spaces and replace documents, and its spaces show as under consideration
  again.
- `returned_at`, `returned_by` and `return_reason` are recorded on the application.
- The **Generate Letter of Offer** task is cleared. The person who approved the
  application (`reviewed_by`) gets a **Re-review returned lease application** task.
  If nobody is recorded as the approver (for example on an imported application), the
  reviewer-role holders get it instead, as for a new application.
- The same people get a notification in-app, by **email**, and by **SMS** when they
  have a phone number. Each one states the reason and links to the application.

The reviewer then reviews it again with [`PATCH .../review`](#review-application):
approve it, send it to the applicant (`review`), or reject it.

Refused with `403`, and a message saying why, when:

- the caller lacks `return-lease-application`, or the application's property is not
  allocated to them;
- the application is not `approved`;
- it has an offer in progress (any offer that is not declined, expired, rejected or
  withdrawn). [Rebase the offer](loo/loo.md#rebase) with `return_to: reviewer`
  instead: that deletes the offer and returns the application in one step.

A missing `reason` is a `422`.

Read `permissions.returnToReviewer` on the application to decide whether to offer the
button. It is the same check the endpoint runs, so it is `false` in every case above.

### Reading a returned application

The application resource carries:

| Field | Type | Notes |
|---|---|---|
| `is_returned` | boolean | `true` while the application is `pending` because it was returned and has not been reviewed since |
| `return_reason` | string \| null | The reason given on the last return |
| `returned_at` | `{raw, formatted, diff}` \| null | When it was last returned |
| `returned_by` | User \| absent | Who returned it. Loaded on `show` |

The three `return*` fields keep the last return after the application is reviewed again,
so the history stays readable. Use `is_returned` to decide whether to show a "returned"
banner.

## Delete Application

`DELETE .../lease-applications/{application}`

Deletes the application, its residential-unit-type requests, and its space
allocation.

## Who sees what

Two different things are easy to confuse, so to state the split once:

| | Staff (`/api/v1/app/...`) | Applicant (`/api/v1/tenant/...`) |
|---|---|---|
| `income_to_rent_ratio_score`, `_reason`, `ratio_indicator` | Yes | **Never** |
| `frc_check_status` | Yes | **Never** |
| `spaces` (allocated units and their prices) | Yes | **Never** |
| Document `status` / `reason`, guarantor `id_status` / `tax_pin_status` | Yes | Yes |
| `proposed_start_date` | Yes | Yes |
| `proposed_period_in_years`, `proposed_period_in_months` | Yes | Yes |
| `escalations` (the schedule the offer is priced against) | Yes, read and write | Read only |
| `bank_account_name`, `bank_account_number`, `bank_branch_id` | Yes | Yes |
| `is_renewal`, `lease_id` | Yes | Yes |
| [Applicant profile](#applicant-profile) fields | Yes | Yes |
| Raw AI extraction and iTax cross-check output | **Never** | **Never** |

**Vetting** is the server's assessment *of* the applicant — an affordability
score and a watchlist outcome. It is staff-only, on every endpoint, without
exception. **Document verification** is the outcome of checking a file the
applicant uploaded, and the uploader sees it so they know what to fix.

The two are enforced separately. Vetting fields are gated on the `viewVetting`
ability, which — unlike `view` — has no ownership fallback, so an applicant is
refused them on their own application. Verification internals are never
serialised anywhere, on either surface.

## Vetting fields (staff only)

The application also carries automatic vetting results. They are returned **only
on the staff surface** (`/api/v1/app/{company}/...`) — the applicant endpoints
under `/api/v1/tenant/...` omit the keys entirely, so an applicant never sees
their own score or FRC outcome. Document `status` and `reason` remain the only
verification data an applicant sees.

All four are read-only: no request body accepts them, and they appear on every
staff read — `GET /lease-applications`, `GET /lease-applications/{application}`
and the `PATCH .../review` response.

| Field | Type | Notes |
|---|---|---|
| `income_to_rent_ratio_score` | string \| null | **0-100, where 100 is best.** Decimal with 2 places, serialised as a string (e.g. `"50.00"`). `null` means *not scored*, which is a normal outcome, not an error |
| `ratio_indicator` | object \| null | `{ "value", "color" }` for the score's band — `weak` (`danger`), `fair` (`warning`), `strong` (`success`). `null` whenever the score is `null` |
| `income_to_rent_ratio_score_reason` | string \| null | Prose explanation, always present once vetting has run. For a score it names both sides of the comparison and where each came from; for a `null` score it says why it could not be scored |
| `frc_check_status` | object \| null | `{ "value", "color" }`. One of `safe` (`success`), `flagged` (`danger`) |

```json
{
  "id": 18,
  "status": { "value": "pending", "color": "warning" },
  "income_to_rent_ratio_score": "100.00",
  "ratio_indicator": { "value": "strong", "color": "success" },
  "income_to_rent_ratio_score_reason": "Scored 100.00 out of 100. Declared monthly income of KES 650,000.00 covers the expected monthly cost of KES 200,000.00 3.25 times, at or above the 3.00 times needed for a full score. Cost basis: 5 x 2 Bedroom at KES 40,000.00 per unit per month = KES 200,000.00. Income read from the submitted bank statement covering 2026-01-01 to 2026-06-30.",
  "frc_check_status": { "value": "safe", "color": "success" }
}
```

### When they are populated

Vetting runs on the queue, not in the request. All three fields are `null` in
the create response and fill in shortly afterwards, so a staff screen should
render "pending vetting" for `null` rather than treating it as a final result.

- **On submission.** Creating an application *is* submitting it, so both checks
  run once the application is created.
- **On a new financial statement.** Submitting or replacing the
  `financial_statement` document re-runs the income-to-rent score only —
  nothing feeding the FRC check has changed. Replacing any other document type
  does not re-run vetting.

Later edits to an application do not re-run vetting. Two things fill that gap:
the [manual trigger](#re-run-vetting-manually) for a reviewer who knows the
inputs have changed, and an [hourly sweep](#the-hourly-sweep) for runs that
never completed at all.

### Re-run Vetting Manually

`POST .../lease-applications/{application}/vet`

**Staff only.** Requires the `view-lease-application` permission — the same
`viewVetting` gate as reading the results, so an applicant cannot trigger a run
on their own application. No request body.

Automatic vetting covers submission and a replaced financial statement, which is
not everything that changes the answer. Use this endpoint after the inputs have
moved underneath a stored result:

- A **guarantor was added** after submission, or an existing one's documents
  changed. The FRC check and the income picture both read wider than the
  applicant's own row.
- **Documents arrived late** — an identity document uploaded after submission
  changes what the FRC name check has to match against.
- A **failed run** needs retrying: the queue was down at submission, or the AI
  provider was unavailable and the job exhausted its retries.
- The **FRC watchlist was re-imported** and applications vetted against the
  previous list need re-checking.

It is `POST` rather than `GET` because it queues work and overwrites the stored
results — a prefetch, a link crawler or a cache warmer must not be able to
trigger it.

Response `202`-style, returned as `200`:

```json
{ "message": "Lease Application vetting started successfully." }
```

The response says the run was **queued**, not that it finished. Re-read
[the vetting results](#vetting-results-endpoint) afterwards to see the outcome;
the previous values stay in place until the new run completes and replaces them.
A run that throws leaves the stored results untouched rather than half-replaced.

Both checks run — this is a full re-vet, not the income-only refresh that a
replaced financial statement triggers.

| Status | When |
|---|---|
| `403` | The caller lacks `view-lease-application`, including an applicant on their own application |
| `404` | No such application under this company |
| `405` | The request used `GET` |

### The Hourly Sweep

An application whose vetting never completed is picked up automatically. The
scheduled command `lease-applications:vet-pending` runs every hour and queues a
full run for anything still unvetted, which covers a worker killed mid-run, a
queue drained or reset, an application created while the queue was down, or a
run that exhausted its retries against a provider outage.

This is why the API has no "vetting failed" state to render: a stalled
application is `null` across the vetting fields and is retried within the hour,
so a staff screen should keep showing "pending vetting" rather than offering the
reviewer an error to act on. The [manual trigger](#re-run-vetting-manually) is
there for the impatient case, not as the recovery path.

The sweep is driven by a `vetted` flag on the application, set only once every
check has returned. It is **not** part of any API response — `null` results and
"never vetted" are the same thing to a client, and the distinction only matters
to the sweep. A run is skipped for applications created in the last 15 minutes
(their own submission already queued one) and for applications staff have
already approved or rejected.

### Reading `income_to_rent_ratio_score`

The score runs **0 to 100, where 100 is the best**. An applicant reaches 100
once their declared monthly income is at least **three times** the expected
monthly cost of the space they applied for — the usual affordability rule that
rent should sit under a third of income. Below that the score is proportional,
so covering the cost exactly scores about 33 rather than full marks, and a very
high earner is simply capped at 100.

| Income vs. expected monthly cost | Score | `ratio_indicator` |
|---|---|---|
| 0.5x (half the cost) | 16.67 | `weak` / `danger` |
| 1.0x (exactly the cost) | 33.33 | `weak` / `danger` |
| 1.5x | 50.00 | `fair` / `warning` |
| 2.0x | 66.67 | `fair` / `warning` |
| 2.5x | 83.33 | `strong` / `success` |
| 3.0x or more | 100.00 | `strong` / `success` |

The multiple is server configuration
(`config('vetting.income_to_rent.target_multiple')`, default `3`), so it can be
tuned without an API change — read the score, never re-derive it. The
`ratio_indicator` bands are fixed: **below 50** is `weak`, **50 to 70
inclusive** is `fair`, and **above 70** is `strong`. Use `ratio_indicator.color`
for the badge rather than re-implementing the thresholds in the frontend.

The expected monthly cost is derived server-side. Where the application has
[allocated spaces](#application-spaces) that are priced, the cost is the sum of
their components — so parking and signage fees count, which the indicative
estimate never covered. Where it has none, or where the allocation totals
nothing because the property records no rates for it, the cost falls back to the
property's own indicative rates: the requested residential unit types, plus the
requested `space_size` where the application has one.

The income is read from the submitted financial statement. A `null` score with a
populated reason means one of those inputs was missing or unusable: no financial
statement submitted, the property has no indicative rates for the space
requested, the statement could not be read, or no income figure could be found
in it. `ratio_indicator` is
`null` in exactly those cases too — it is never a misleading `weak`.

Amounts in the reason are stated in the property's reporting currency. If the
statement reports a different currency the reason says so explicitly — **no
conversion is applied**, so a cross-currency score is not comparable and the
note is there for the reviewer to catch it.

### Reading `frc_check_status`

`flagged` means the applicant's name matched the current FRC watchlist import
exactly, once casing and spacing are normalised — there is no fuzzy matching, so
a near-miss reads `safe`. Both the name on the application and the name read off
the submitted identity document are checked, and a hit on either flags.

A `flagged` result is advisory. It does not block review, approval or any other
step — it is recorded for staff to weigh, so the UI should surface it to the
reviewer rather than gate on it.

### Vetting Results Endpoint

`GET .../lease-applications/{application}/vetting-results`

**Staff only.** Requires the `view-lease-application` permission. Unlike
`GET /lease-applications/{application}`, holding the application's ownership is
*not* enough — this endpoint is authorised with `viewVetting`, which is the bare
permission with no ownership fallback, so an applicant reading their own
application is refused `403`. There is no tenant equivalent of this route.

The same fields are also returned inline on every staff read of the application
itself, so this endpoint is a convenience for a dedicated vetting screen, not
the only way to reach them. It exists so that screen can fetch the assessment
and the per-document outcomes together without pulling the full application.

Response `200`:

| Field | Type | Notes |
|---|---|---|
| `id` | integer | The application's id |
| `income_to_rent_ratio_score` | string \| null | 0-100, 100 is best. See [Reading the score](#reading-income_to_rent_ratio_score) |
| `income_to_rent_ratio_score_reason` | string \| null | Prose explanation of the score, or of why it could not be scored |
| `ratio_indicator` | object \| null | `{ "value", "color" }` — `weak` / `fair` / `strong` |
| `frc_check_status` | object \| null | `{ "value", "color" }` — `safe` (`success`) or `flagged` (`danger`) |
| `documents` | array | Applicant documents with their verification state — the same shape as [Reading documents](#reading-documents), uploads included |
| `guarantors` | array | Guarantors with `id_status` / `tax_pin_status` and reasons — the same shape as [Guarantor Response](#guarantor-response) |

```json
{
  "data": {
    "id": 18,
    "income_to_rent_ratio_score": "82.50",
    "income_to_rent_ratio_score_reason": "Scored 82.50 out of 100. Declared monthly income of KES 495,000.00 covers the expected monthly cost of KES 200,000.00 2.48 times, against the 3.00 times needed for a full score.",
    "ratio_indicator": { "value": "strong", "color": "success" },
    "frc_check_status": { "value": "flagged", "color": "danger" },
    "documents": [
      {
        "id": 41,
        "document_type": { "value": "tax_pin", "color": "primary" },
        "status": { "value": "verified", "color": "success" },
        "reason": null,
        "upload": { "id": 902, "file_name": "kra-pin.pdf" },
        "verified_at": { "raw": "2026-08-27T09:14:00.000000Z", "formatted": "27 Aug, 2026", "diff": "2 hours ago" }
      }
    ],
    "guarantors": [
      {
        "id": 7,
        "name": "Jane Guarantor",
        "id_status": { "value": "verified", "color": "success" },
        "id_reason": null,
        "tax_pin_status": { "value": "failed", "color": "danger" },
        "tax_pin_reason": "PIN does not match the applicant name"
      }
    ]
  }
}
```

The application's own attributes are deliberately not repeated in this payload
beyond `id` — fetch the application itself for those.

Neither the score nor the FRC result is returned as raw integration output. The
AI extraction and iTax cross-check responses behind a document verification stay
server-side on every surface, including this one, so `verification_result` is
never present.

Both results are **advisory**. Nothing here blocks review or approval; a
`flagged` FRC result and a `weak` score are recorded for the reviewer to weigh.

Errors:

| Status | When |
|---|---|
| `403` | The caller lacks `view-lease-application`, including an applicant on their own application |
| `404` | No such application under this company |
