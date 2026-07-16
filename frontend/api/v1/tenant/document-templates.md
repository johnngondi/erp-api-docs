# Tenant Document Templates API

Base route:

`/api/v1/tenant/document-templates`

Let a tenant **print the documents they own** (invoices, receipts, credit notes)
through a saved template — the same template designer / renderer used by the
property-management portal, but scoped to the tenant's own records.

Auth:

- Requires `auth:sanctum` and tenant portal access.
- Tenant-scoped: a document is only reachable if it belongs to the tenant's own
  leases (receipts also via the paying user). A foreign/unknown `resource_id`
  returns `404`.

## Printable document types

| `document_type`        | Source document |
| ---------------------- | --------------- |
| `facility_invoice`     | an invoice on one of the tenant's leases |
| `facility_receipt`     | a receipt the tenant paid, or allocated to their invoices |
| `facility_credit_note` | a credit note on one of the tenant's leases |

Any other type returns `403`.

## Endpoints

- `GET  /document-templates`
- `POST /document-templates/{documentTemplate}/render`

---

## List usable templates

`GET /api/v1/tenant/document-templates`

Returns the active templates that can print a **specific** document the tenant
owns. The company + facility are derived from that document, so you must pass it.

Query params (both required):

- `document_type` — one of the printable types above.
- `resource_id` — the id of the document to print (e.g. the invoice id).

```
GET /api/v1/tenant/document-templates?document_type=facility_invoice&resource_id=481
```

Response: a `data` array of templates (`DocumentTemplateResource` — `id`, `name`,
`document_type`, `document_type_label`, `is_default`, `page_setup`, `layout`, …).
Use it to let the tenant pick which template to print with (or auto-pick
`is_default`).

Errors:

- `422` — missing/invalid `document_type` or `resource_id`.
- `404` — the `resource_id` is not a document this tenant owns.

---

## Render / print a document

`POST /api/v1/tenant/document-templates/{documentTemplate}/render`

Body:

```jsonc
{ "resource_id": 481 } // the document to render with this template
```

Query params:

- `format=html` — return `text/html` (handy for an in-page preview iframe).
- *(default)* — return an `application/pdf` stream (inline).

```
POST /api/v1/tenant/document-templates/12/render
Accept: application/pdf
{ "resource_id": 481 }
```

The server resolves `resource_id` through the tenant's ownership scope, checks
the template applies to the document's company + facility, then renders real
data. There is no `?preview=1` here — the tenant always prints a real, owned
document.

### Access rules

- `403` — the template's `document_type` is not printable by the tenant portal,
  or the template does not belong to the document's company / facility.
- `404` — the `resource_id` is not a document this tenant owns.

### Notes

- PDF rendering needs headless Chrome on the host; if it is unavailable the
  request fails. Use `?format=html` when you only need an on-screen preview.
- `model_id` is accepted as a legacy alias for `resource_id`.
