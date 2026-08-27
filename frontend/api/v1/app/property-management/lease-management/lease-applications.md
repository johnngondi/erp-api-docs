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

## Vetting fields (staff only)

The application also carries automatic vetting results. They are returned **only
on the staff surface** (`/api/v1/app/{company}/...`) — the applicant endpoints
under `/api/v1/tenant/...` omit the keys entirely, so an applicant never sees
their own score or FRC outcome. Document `status` and `reason` remain the only
verification data an applicant sees.

All three are read-only: no request body accepts them, and they appear on every
staff read — `GET /lease-applications`, `GET /lease-applications/{application}`
and the `PATCH .../review` response.

| Field | Type | Notes |
|---|---|---|
| `income_to_rent_ratio_score` | string \| null | Decimal with 2 places, serialised as a string (e.g. `"3.25"`) — how many times the applicant's declared monthly income covers the expected monthly cost of the space requested. `null` means *not scored*, which is a normal outcome, not an error |
| `income_to_rent_ratio_score_reason` | string \| null | Prose explanation, always present once vetting has run. For a score it names both sides of the ratio and where each came from; for a `null` score it says why it could not be scored |
| `frc_check_status` | object \| null | `{ "value", "color" }`. One of `safe` (`success`), `flagged` (`danger`) |

```json
{
  "id": 18,
  "status": { "value": "pending", "color": "warning" },
  "income_to_rent_ratio_score": "3.25",
  "income_to_rent_ratio_score_reason": "Declared monthly income of KES 650,000.00 covers the expected monthly cost of KES 200,000.00 3.25 times. Cost basis: 5 x 2 Bedroom at KES 40,000.00 per unit per month = KES 200,000.00. Income read from the submitted bank statement covering 2026-01-01 to 2026-06-30.",
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

Later edits to an application do not re-run vetting.

### Reading `income_to_rent_ratio_score`

The expected monthly cost is derived server-side from the property's own
indicative rates — the requested residential unit types, plus the requested
`space_size` where the application has one. The income is read from the
submitted financial statement. A `null` score with a populated reason means one
of those inputs was missing or unusable: no financial statement submitted, the
property has no indicative rates for the space requested, the statement could
not be read, or no income figure could be found in it.

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
