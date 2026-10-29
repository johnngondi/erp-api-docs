# Prequalified Suppliers Report API

Domain: `Property Management > Reports > Suppliers`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Prequalified Suppliers report lists **which suppliers are attached to which expense
sub-category** (`facility_expense_sub_type_vendors`). It opens with an overview of every
sub-category and how many suppliers it has, then gives one bucket per sub-category listing its
suppliers. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page documents what is specific to
> this report. The response envelope, the bucket keys, the field keys, the per-cell `{ value, color }`
> shape and the colour vocabulary live in the README and apply here unchanged.

## Endpoints

- `GET /reports/suppliers/prequalified-suppliers` — requires `view-prequalified-suppliers-report`
- `GET /reports/suppliers/prequalified-suppliers/export` — requires
  `export-prequalified-suppliers-report`. Takes a required `format` (`excel` | `pdf`) plus the same
  filters as the report; any other format, or none, returns `422` with a `format` error.
- Listed by [`GET /reports`](./README.md#discovering-reports--get-reports) as a top-level supplier
  report (it is not in a parent group).

The view and export permissions are **independent**.

## What is listed

- **Every attached row**, prequalified or not. A row whose prequalification was switched off
  (`is_prequalified = false`, for example by terminating the supplier) is still listed, with
  `prequalified: "No"` and a `danger` tint. Use the `prequalified` column to tell them apart.
- **Sub-categories** are the company's own (`facility_expense_sub_types`, not deleted), each under
  its expense type.
- **As at today.** The register shows the current state. The period filters (`period_from` /
  `period_to`) are **echoed in the header but narrow nothing**. `header.as_at` is today's date.
- There is no money in this report, so `header.currency` is always `null`.

## Filters

The [global filters](./README.md#global-filters) plus:

| Param | Type | Meaning when omitted |
|---|---|---|
| [global filters](./README.md#global-filters) — `landlord_ids`, `facility_ids`, `facility_type_id`, `properties_status`, period | see the shared contract and the property rule below | every supplier row (the period is echoed only) |
| `expense_type_ids` | list of `facility_expense_types.id` (`expense_type_ids[]=1&expense_type_ids[]=2`, or `1,2`) | Every expense type |
| `supplier_id` | `users.id` (the row's `vendor_id`) | All suppliers |
| `buckets` | list of this report's bucket ids (`overview`, `sub-type-{id}`) | Every bucket — see [Selecting buckets](./README.md#selecting-buckets--buckets) |

- **`expense_type_ids`** keeps only sub-categories under these expense types. A sub-category under
  them that has no supplier still appears in the overview, as a `0` row.
- **`supplier_id`** keeps only that supplier's rows, and **only the sub-categories that supplier is
  attached to** appear, in the overview and as buckets. The supplier shows in every bucket they are in.
  `header.supplier` names them.
- **Property rule.** A supplier row carries a list of the properties it serves (`facilities`).
  `null` means **every property**, the same rule procurement uses when it picks suppliers. A row is
  listed when:
  - its list is `null` (all properties), or
  - its list shares a property with the properties in scope (the ones you are allocated to, narrowed
    by `landlord_ids`, `facility_ids`, `facility_type_id` and `properties_status`), or
  - no property filter is set and its list is empty.

  So selecting a property keeps the "all properties" suppliers and the suppliers named on it, and
  drops the rest. A row that only names properties outside your allocation is never listed.

`header.filters` echoes every filter.

```
GET …/reports/suppliers/prequalified-suppliers
GET …/reports/suppliers/prequalified-suppliers?supplier_id=42
GET …/reports/suppliers/prequalified-suppliers?expense_type_ids[]=3&expense_type_ids[]=7&facility_ids[]=12
```

## Header

- `property` / `properties` / `landlord` / `landlords` — the selection, as in the
  [shared `header`](./README.md#header).
- `currency` — always `null` (no money in this report).
- `as_at` — today's date (`Y-m-d`).
- `period` — the echoed period (not used to narrow).
- `supplier` — `{ id, name }` when `supplier_id` is given, otherwise `null`.
- `expense_types` — `[{ id, name }]` for the selected `expense_type_ids`, ordered by name; `[]` when none.

## Buckets

In this order:

| `bucket` | `header.label` | When | One row per… |
|---|---|---|---|
| `overview` | `By Sub-category` | always, first | sub-category in scope, then a `Total` subtotal row |
| `sub-type-{id}` | `{sub-category} ({expense type})`, or just the sub-category when it has no expense type | one per sub-category with **at least one listed supplier** | supplier, then a subtotal row |

Sub-categories are ordered by expense type name (untyped last), then by sub-category name, in the
overview and across the `sub-type-{id}` buckets alike. **A sub-category with no listed supplier gets
an overview row but no bucket of its own.** With `supplier_id`, it gets neither.

Each `sub-type-{id}` bucket header also carries `expense_type` (`{ id, name }`, or `null`) and
`expense_sub_type` (`{ id, name }`).

### `overview` columns

| Column | `key` | `format` | Source |
|---|---|---|---|
| Expense Type | `expense_type` | string | the sub-category's expense type, or `Untyped` |
| Sub-category | `sub_type` | string | the sub-category's name |
| Suppliers | `supplier_count` | integer | rows listed in its bucket (the number of rows in `sub-type-{id}`) |
| Prequalified | `prequalified_count` | integer | of those, the rows with `prequalified: "Yes"` |
| Default Supplier | `default_supplier` | string | the category-wide default supplier (`default_vendor_id`), when the sub-category allows a default supplier (`can_have_default_vendor`); otherwise `null` |

A row with `prequalified_count` of `0` is tinted `warning`: nobody can be invited for that
sub-category. The closing row is a label `Total` with `col_span: 2`, then `supplier_count` and
`prequalified_count` summed over the rows above; `default_supplier` is omitted. It has
`type: subtotal` and `background_color: secondary`.

### `sub-type-{id}` columns

| Column | `key` | `format` | Source |
|---|---|---|---|
| Supplier | `supplier` | string | the supplier's name |
| Email | `email` | string | the supplier's email |
| Phone | `phone` | string | the supplier's phone |
| Tax PIN | `tax_pin` | string | the supplier's tax PIN |
| Status | `status` | string | the supplier's status; the cell carries its colour (`active` success, `suspended` warning, `pending` secondary, `inactive` / `terminated` danger, …) |
| Prequalified | `prequalified` | string | `Yes` or `No` (`is_prequalified`) |
| Properties | `properties` | string | `All properties` when the row's list is `null`; otherwise the names of its properties that you are allocated to, comma-separated and sorted; `null` when it names none you can see |
| Default In | `default_in` | string | where the supplier is the default, comma-separated: `Category default` when they are the sub-category's `default_vendor_id`, then the property named by `is_default_in_facility_id` (if you are allocated to it). Only filled when the sub-category allows a default supplier; `null` otherwise |

Rows are ordered by supplier name. The closing row is a label
(`Total (N suppliers, M prequalified)`) with `col_span: 8`, so every other key is omitted. It has
`type: subtotal` and `background_color: secondary`.

## Tinting

`sub-type-{id}` rows carry a `background_color`:

| When | `background_color` |
|---|---|
| The row is not prequalified | `danger` |
| Prequalified, but the supplier is not `active` | `warning` |
| Otherwise | none (key omitted) |

`overview` rows are tinted `warning` when `prequalified_count` is `0`.

## Summary

Each `sub-type-{id}` bucket's `summary`:

| Key | Meaning |
|---|---|
| `supplier_count` | Rows listed |
| `prequalified_count` | Rows with `prequalified: "Yes"` |
| `active_count` | Rows whose supplier is `active` |

`overview.summary` and `data.summary`:

| Key | Meaning |
|---|---|
| `sub_type_count` | Overview rows (sub-categories in scope) |
| `sub_types_without_prequalified` | Overview rows with `prequalified_count` of `0` — the `warning` rows |
| `supplier_count` | Distinct suppliers across every bucket (one supplier in three sub-categories counts once) |
| `assignment_count` | Rows across every bucket (that supplier counts three times) |
| `prequalified_count` | Rows with `prequalified: "Yes"` across every bucket |

## Example response

No filters, as at 29 Oct 2026:

```jsonc
{
  "data": {
    "header": {
      "property": null, "properties": [], "landlord": null, "landlords": [],
      "currency": null,
      "as_at": "2026-10-29",
      "period": { "from": "2026-10-01", "to": "2026-10-31" },
      "supplier": null,
      "expense_types": [],
      "filters": { "landlord_id": null, "facility_id": null, "facility_type_id": null, "properties_status": null,
                   "period_from": null, "period_to": null, "expense_type_ids": [], "supplier_id": null,
                   "landlord_ids": [], "facility_ids": [], "buckets": [] },
      "generated_at": "2026-10-29T12:00:00+03:00"
    },
    "report": [
      {
        "bucket": "overview",
        "header": { "label": "By Sub-category" },
        "fields": [ /* expense_type, sub_type, supplier_count, prequalified_count, default_supplier */ ],
        "items": [
          { "expense_type": { "value": "Electrical" }, "sub_type": { "value": "Wiring" }, "supplier_count": { "value": 2 },
            "prequalified_count": { "value": 1 }, "default_supplier": { "value": "Bright Sparks Ltd" }, "type": "normal" },
          { "expense_type": { "value": "Plumbing" }, "sub_type": { "value": "Drainage" }, "supplier_count": { "value": 0 },
            "prequalified_count": { "value": 0 }, "default_supplier": { "value": null }, "type": "normal", "background_color": "warning" },
          { "expense_type": { "value": "Total", "col_span": 2 }, "supplier_count": { "value": 2 }, "prequalified_count": { "value": 1 },
            "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "sub_type_count": 2, "sub_types_without_prequalified": 1, "supplier_count": 2, "assignment_count": 2, "prequalified_count": 1 }
      },
      {
        "bucket": "sub-type-14",
        "header": { "label": "Wiring (Electrical)", "expense_type": { "id": 3, "name": "Electrical" }, "expense_sub_type": { "id": 14, "name": "Wiring" } },
        "fields": [ /* the eight columns above, in order */ ],
        "items": [
          { "supplier": { "value": "Bright Sparks Ltd" }, "email": { "value": "info@brightsparks.co.ke" }, "phone": { "value": "0700000001" },
            "tax_pin": { "value": "P051234567A" }, "status": { "value": "active", "color": "success" }, "prequalified": { "value": "Yes" },
            "properties": { "value": "All properties" }, "default_in": { "value": "Category default, ACK Gardens" }, "type": "normal" },
          { "supplier": { "value": "Volt Masters" }, "email": { "value": "volt@example.com" }, "phone": { "value": null },
            "tax_pin": { "value": null }, "status": { "value": "terminated", "color": "danger" }, "prequalified": { "value": "No" },
            "properties": { "value": "ACK Gardens, Beta House" }, "default_in": { "value": null }, "type": "normal", "background_color": "danger" },
          { "supplier": { "value": "Total (2 suppliers, 1 prequalified)", "col_span": 8 }, "type": "subtotal", "background_color": "secondary" }
        ],
        "summary": { "supplier_count": 2, "prequalified_count": 1, "active_count": 1 }
      }
    ],
    "summary": { "sub_type_count": 2, "sub_types_without_prequalified": 1, "supplier_count": 2, "assignment_count": 2, "prequalified_count": 1 }
  }
}
```
