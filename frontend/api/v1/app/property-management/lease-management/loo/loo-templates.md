# LOO Templates — Data Model & Contracts

Domain: `Property Management > Lease Management`

The reusable clause text a Letter of Offer is drafted from. For the document
itself see [loo.md](./loo.md); for the tag registry both sides share, and the
status lifecycle, see [README.md](./README.md).

> **Status: schema only.** `LooTemplate` and its scoping rules are built. There
> are no template endpoints yet — CRUD under Settings lands in Ticket 4.

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

## Related

- [LOO document](./loo.md) — what a generated LOO holds, and its approval chain
- [README](./README.md) — the tag registry, resolution, and the status lifecycle
