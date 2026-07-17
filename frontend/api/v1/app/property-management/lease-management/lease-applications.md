# Lease Applications API

Domain: `Property Management > Lease Management`

A lease application is a prospective tenant's request to lease space. Applicants
(tenants) submit and maintain their own applications; staff list, review, and
approve/reject them.

An application may name the specific facility spaces it is for. While the
application is open (`pending` / `review`), each of those spaces shows as
`under consideration` and is hidden from other applicants' space pickers;
approving or rejecting the application releases them. See
[space occupancy](../facilities/space-occupancy.md).

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
| `city_id` | Yes | integer | Must exist in `cities.id` |
| `registration_type` | Yes | string | `national_id`, `business_license` or `passport` |
| `registration_number` | Yes | string | |
| `tax_pin` | Yes | string | |
| `registration_upload_id` | Yes | integer | Must exist in `uploads.id` |
| `tax_pin_cert_upload_id` | Yes | integer | Must exist in `uploads.id` |
| `financial_upload_id` | Yes | integer | Must exist in `uploads.id` |
| `other_docs_upload_id` | No | integer | Must exist in `uploads.id` |
| `region` | No | string | |
| `parking_required` | No | boolean | Defaults to `true` |
| `generator_required` | No | boolean | Defaults to `true` |
| `occupants` | No | integer | `0`–`100` |
| `space_size` | No | integer | `0`–`1,000,000` |
| `preferred_facilities` | No | array | Free-form preference list |
| `facility_space_ids` | No | int[] | Specific spaces this application is for. Each must exist in `facility_spaces.id`. Attaching them marks each free space `under consideration`. |
| `status` | No | string | Defaults to `pending` |
| `comments` | No | string | |
| `application_submitted_at` | No | date | Defaults to now |

## Update Application

`PUT/PATCH /api/v1/tenant/lease-applications/{application}`

Accepts the same body as create. Notes on `facility_space_ids`:

- **Omit** the field to leave the current space selection unchanged.
- Send an **array** to replace the selection (a full `sync`); spaces added and
  spaces removed are both re-evaluated for occupancy.
- Send an **empty array** to clear the selection.

## Review Application

`PATCH .../lease-applications/{application}/review`

Staff-only. Request body:

| Field | Required | Type | Notes |
|---|---|---|---|
| `status` | Yes | string | Target `LeaseApplicationStatus` |
| `comments` | No | string | Reviewer notes |
| `signed_agreement_upload_id` | No | integer | Must exist in `uploads.id` |

Approving or rejecting closes the application and releases any spaces it held as
`under consideration`.

## Delete Application

`DELETE .../lease-applications/{application}`

Deletes the application; any spaces it held as `under consideration` are released.
