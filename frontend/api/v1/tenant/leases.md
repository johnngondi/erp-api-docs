# Tenant Leases API

Base route:

`/api/v1/tenant/leases`

These endpoints are read-only for tenant users.

## Endpoints

- `GET /leases`
- `GET /leases/{lease}`

## List Leases

`GET /api/v1/tenant/leases`

Supported query params:

- Filters:
  - `filter[facility_id]`
  - `filter[status]`
  - `filter[created_at]`
  - `filter[start_at]`
  - `filter[end_at]`
  - `filter[next_due_at]`
  - `filter[billing_cycle]`
  - `filter[user_id]`
- Sort:
  - `sort=user_id,status,created_at,start_at,end_at,next_due_at,billing_cycle`
- Pagination:
  - `per_page`
  - `page`

## Show Lease

`GET /api/v1/tenant/leases/{lease}`

If `{lease}` does not belong to the authenticated tenant, the API returns `404`.

## Lease Documents

`GET /api/v1/tenant/leases/{lease}/documents`

Read only. Tenants download what is already on the lease; attaching is staff-side.

`GET /leases/{lease}` also returns an `uploads` key holding only the files attached to the lease
itself, matching the staff resource. The endpoint below is the one the Documents tab reads.

If `{lease}` does not belong to the authenticated tenant, the API returns `404`.

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

Two things are narrower than the staff list:

- an offer letter whose Loo has not cleared approval does not appear - it is internal drafting;
- `application-document` rows are never served here. Those were collected for vetting.

`removable` is always `false` on this surface.
