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
- `GET /lease-applications/{application}`
- `DELETE /lease-applications/{application}`
- `PATCH /lease-applications/{application}/review`
- `GET /lease-applications/{application}/vetting-results` — staff-only vetting summary
- `POST /lease-applications/{application}/vet` — re-run vetting for one application

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
  - `filter[reviewed_at]`
  - `filter[parking_required]`
  - `filter[generator_required]`
  - `filter[registration_type]`
  - `filter[created_at]`
- Includes: `include=guarantors`, `documents`, `documents.upload`, `facility`, `residentialUnitTypes`
- Sort: `sort=id,created_at,application_submitted_at,reviewed_at`
- Pagination: `per_page`, `page`

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
| `parking_required` | No | boolean | Defaults to `true` |
| `generator_required` | No | boolean | Defaults to `true` |
| `occupants` | No | integer | `0`–`100` |
| `space_size` | No | integer | `0`–`1,000,000` |
| `residential_unit_types` | Yes | array | One or more requested unit types, each with a facility-level `FacilityResidentialUnit` `id` and a positive integer `quantity`. Each unit type must belong directly to `facility_id`. |
| `status` | No | string | Defaults to `pending` |
| `comments` | No | string | |
| `application_submitted_at` | No | date | Defaults to now |

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

The application no longer carries any document upload fields. Create the
application first, then submit each document to
[its own endpoint](#submit-a-document).

## Update Application

`PUT/PATCH /api/v1/tenant/lease-applications/{application}`

Accepts the same body as create. `residential_unit_types` replaces the current
unit-type requests as a full sync, and the application company is refreshed
from the selected `facility_id`. Documents are not part of this payload.

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
| `email_address` | string | |
| `phone_number` | string | |
| `registration_type` | string | `national_id`, `business_license` or `passport` |
| `registration_number` | string | |
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

## Review Application

`PATCH .../lease-applications/{application}/review`

Staff-only. Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `status` | Yes | string | Target `LeaseApplicationStatus` |
| `comments` | No | string | Reviewer notes |
| `signed_agreement_upload_id` | No | integer | Must exist in `uploads.id` |

Approving or rejecting records the review outcome for the application.

## Delete Application

`DELETE .../lease-applications/{application}`

Deletes the application and its residential-unit-type requests.

## Who sees what

Two different things are easy to confuse, so to state the split once:

| | Staff (`/api/v1/app/...`) | Applicant (`/api/v1/tenant/...`) |
|---|---|---|
| `income_to_rent_ratio_score`, `_reason`, `ratio_indicator` | Yes | **Never** |
| `frc_check_status` | Yes | **Never** |
| Document `status` / `reason`, guarantor `id_status` / `tax_pin_status` | Yes | Yes |
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

The expected monthly cost is derived server-side from the property's own
indicative rates — the requested residential unit types, plus the requested
`space_size` where the application has one. The income is read from the
submitted financial statement. A `null` score with a populated reason means one
of those inputs was missing or unusable: no financial statement submitted, the
property has no indicative rates for the space requested, the statement could
not be read, or no income figure could be found in it. `ratio_indicator` is
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
