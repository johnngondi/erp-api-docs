# Lease Applications API

Domain: `Property Management > Lease Management`

A lease application is a prospective tenant's request to lease space. Applicants
(tenants) submit and maintain their own applications; staff list, review, and
approve/reject them.

An application identifies one facility and the residential unit types requested
at that facility. The server derives company ownership from the facility, so
staff can retrieve the application under the correct company.

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
- `.../{application}/guarantors` (nested resource)

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
- Includes: `include=guarantors`
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
| `registration_upload_id` | No | integer | Must exist in `uploads.id` |
| `tax_pin_cert_upload_id` | No | integer | Must exist in `uploads.id` |
| `financial_upload_id` | No | integer | Must exist in `uploads.id` |
| `other_docs_upload_id` | No | integer | Must exist in `uploads.id` |
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

## Update Application

`PUT/PATCH /api/v1/tenant/lease-applications/{application}`

Accepts the same body as create. `residential_unit_types` replaces the current
unit-type requests as a full sync, and the application company is refreshed
from the selected `facility_id`.

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
