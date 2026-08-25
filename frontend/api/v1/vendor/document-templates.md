# Vendor Document Templates API

Base route:

`/api/v1/vendor/document-templates`

Let a vendor **print the LPOs addressed to them** through a saved template — the
same template designer / renderer used by the property-management portal, but
scoped to the vendor's own records.

Auth:

- Requires `auth:sanctum` and vendor portal access.
- Vendor-scoped: an LPO is only reachable if its `vendor_id` is the current
  vendor. A foreign/unknown `resource_id` returns `404`.

## Printable document types

| `document_type` | Source document |
| --------------- | --------------- |
| `facility_lpo`  | a purchase order (LPO) addressed to the vendor |

Any other type returns `403`.

## Endpoints

- `GET  /document-templates`
- `POST /document-templates/{documentTemplate}/render`

---

## List usable templates

`GET /api/v1/vendor/document-templates`

Returns the active templates that can print a **specific** LPO the vendor owns.
The company + facility are derived from that LPO, so you must pass it.

Query params (both required):

- `document_type` — `facility_lpo`.
- `resource_id` — the LPO id.

```
GET /api/v1/vendor/document-templates?document_type=facility_lpo&resource_id=93
```

Response: a `data` array of templates (`DocumentTemplateResource` — `id`, `name`,
`document_type`, `document_type_label`, `is_default`, `page_setup`, `layout`, …).
Use it to let the vendor pick which template to print with (or auto-pick
`is_default`).

Errors:

- `422` — missing/invalid `document_type` or `resource_id`.
- `404` — the `resource_id` is not an LPO addressed to this vendor.

---

## Render / print an LPO

`POST /api/v1/vendor/document-templates/{documentTemplate}/render`

Body:

```jsonc
{ "resource_id": 93 } // the LPO to render with this template
```

Query params:

- `format=html` — return `text/html` (handy for an in-page preview iframe).
- *(default)* — return an `application/pdf` stream (inline).

```
POST /api/v1/vendor/document-templates/12/render
Accept: application/pdf
{ "resource_id": 93 }
```

The server resolves `resource_id` through the vendor's ownership scope, checks
the template applies to the LPO's company + facility, then renders real data.
There is no `?preview=1` here — the vendor always prints a real, owned document.

### Access rules

- `403` — the template's `document_type` is not printable by the vendor portal,
  or the template does not belong to the LPO's company / facility.
- `404` — the `resource_id` is not an LPO addressed to this vendor.

### Notes

- PDF rendering needs headless Chrome on the host; if it is unavailable the
  request fails. Use `?format=html` when you only need an on-screen preview.
- `model_id` is accepted as a legacy alias for `resource_id`.
