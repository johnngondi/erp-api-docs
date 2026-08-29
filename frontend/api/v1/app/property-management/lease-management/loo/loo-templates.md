# LOO Templates — Data Model & Contracts

Domain: `Property Management > Lease Management`

The reusable clause text a Letter of Offer is drafted from. For the document
itself see [loo.md](./loo.md); for the tag registry both sides share, and the
status lifecycle, see [README.md](./README.md).

> **Status: complete.** `LooTemplate`, its scoping rules and the Settings CRUD
> surface are all built.

## Templates

A template carries two independent content blobs, `offer_content` and
`agreement_content`, both `{{tag}}`-bearing HTML.

Scoping is two-dimensional, and `NULL` means "no restriction" on either axis:

| Field | `NULL` means | Set means |
|---|---|---|
| `facility_ids` (json array) | usable at every facility of the company | only those facility ids |
| `facility_type_id` | usable for any space category | only that facility type |

The eligible set for a generation is `active AND facility-usable AND type-usable`.
A company may mark one template `is_default` per `facility_type_id`; setting a new
default clears the previous one.


## What a generated LOO takes

Generation **copies** `offer_content` and `agreement_content` onto the LOO. The
two are independent from that moment: editing a LOO never touches its template,
and editing a template never changes a LOO already generated from it.

The `{{tags}}` are copied across as literal tokens — they are never substituted
into the stored text. Resolved values live in a separate per-LOO store, which is
what lets a reviewer correct one value without reflowing the clause it sits in.
See [README.md](./README.md#the-tag-registry) for the registry and
[README.md](./README.md#tag-resolution) for how values are resolved.

## Endpoints

Base: `api/v1/app/{company}/property-management/settings/loo-templates`

| Method | Path | Permission |
|---|---|---|
| `GET` | `` | `view-loo-template` |
| `POST` | `` | `manage-loo-template` |
| `GET` | `/{looTemplate}` | `view-loo-template` |
| `PUT`/`PATCH` | `/{looTemplate}` | `manage-loo-template` |
| `DELETE` | `/{looTemplate}` | `manage-loo-template` |
| `POST` | `/{looTemplate}/set-default` | `manage-loo-template` |

### Payload

```json
{
  "name": "Commercial — Standard Offer",
  "offer_content": "<p>We offer {{tenant_name}} the premises at {{property_name}}.</p>",
  "agreement_content": "<p>…</p>",
  "facility_ids": [12, 18],
  "facility_type_id": 3,
  "is_default": true,
  "is_active": true
}
```

Both content blobs are optional — a template can be created with the offer
drafted and the agreement not. `facility_ids` must all belong to the current
company; `null` on either scoping field means "no restriction".

### Filtering the index

```http
GET settings/loo-templates
    ?filter[facility_id]=12&filter[facility_type_id]=3&filter[is_active]=1
```

`facility_id` and `facility_type_id` are **usability** filters, not equality
ones: each returns the company-wide templates *plus* the ones scoped to that
value. Together they are the eligible set the generation screen offers. Defaults
sort first.

### Deleting

A soft delete. Offers already generated from the template are untouched and keep
working — they hold their own copy of the clause text.

### Choosing a template at generation

`POST .../loos` takes an optional `loo_template_id`. Omit it and the server
resolves one: the default for the property and space type, or the only eligible
one. It **refuses** rather than guessing when several are eligible and none is
marked default — which clauses a tenant is asked to sign is not a coin toss. An
explicit id that is not eligible for the property and space type is refused too,
so a template scoped elsewhere cannot be reached by id.

## Related

- [Tag reference](./loo-tags.md) — every tag that can be dragged into this text, and how each resolves
- [LOO document](./loo.md) — what a generated LOO holds, and its approval chain
- [README](./README.md) — the tag registry, resolution, and the status lifecycle
