# Lease Applications API

Domain: `Property Management > Lease Management`

A lease application is a prospective tenant's request to lease space. Applicants
(tenants) submit and maintain their own applications; staff list, review, and
approve/reject them.

An application must identify its facility and the specific facility spaces it is
for. The server derives company ownership from that facility and verifies every
selected space belongs to the same facility company; staff see the application
only under that company. While the application is
open (`pending` / `review`), each selected space shows as
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
| `facility_id` | Yes | integer | Must exist in `facilities.id`. The server derives the application's company from this facility. |
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
| `preferred_facilities` | No | array | Free-form preference list only. It does not establish the application's company or replace `facility_space_ids`. |
| `facility_space_ids` | Yes | int[] | One or more specific FacilitySpace IDs. Every ID must exist in `facility_spaces.id` and belong to the selected facility's company. These spaces are marked `under consideration` while the application is open. |
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
  "registration_upload_id": 101,
  "tax_pin_cert_upload_id": 102,
  "financial_upload_id": 103,
  "preferred_facilities": [12],
  "facility_space_ids": [345, 346]
}
```

`345` and `346` must belong to facility `12` and its company. `preferred_facilities`
may be sent for preference information, but it does not replace the required
facility and selected spaces.

## Update Application

`PUT/PATCH /api/v1/tenant/lease-applications/{application}`

Accepts the same body as create. Notes on `facility_space_ids`:

- **Omit** the field to leave the current selection unchanged, provided the
  application already has selected spaces.
- Send an **array** to replace the selection (a full `sync`); spaces added and
  spaces removed are both re-evaluated for occupancy.
- When changing `facility_space_ids`, also send `facility_id`; the server
  derives and updates the application's company from the selected facility.
- The array must contain at least one valid FacilitySpace ID, and all selected
  spaces must belong to the selected facility and its company. An empty array
  is rejected.

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
