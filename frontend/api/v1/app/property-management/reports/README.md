# Reports — the shared contract

Domain: `Property Management > Reports`

Base route:

`/api/v1/app/{company}/property-management/reports`

**Read this once.** Every report in this module shares one self-describing response envelope, one set
of global filters, and one presentation vocabulary (formats, colors, weights, background tints). A
single generic table renderer — the reusable **report template** — can display this report and every
future one without hardcoding columns or colours. Each per-report page (e.g.
[billings-and-collections.md](./billings-and-collections.md),
[income-and-expenditure.md](./income-and-expenditure.md),
[facility-budget.md](./facility-budget.md)) then only documents what is specific to
that report: its extra filters, its buckets, and its numbers.

> **One exception to "global filters".** [trial-balance.md](./trial-balance.md) is a company-wide
> accounting snapshot rather than a party report: it takes **none** of the six global filters and has
> no `period` in its `header` — only `as_at` and `currency_id`. Everything else on this page still
> applies to it unchanged.

## What a report is

- **Read-only** and **computed on the fly** — no persistence, no side effects.
- **Self-describing** — the response tells you its own columns (`fields`) and rows (`items`), so the
  UI renders generically.
- **Discoverable** — every report has a stable string key, and
  [`GET /reports`](#discovering-reports--get-reports) lists every report the caller may view with
  its URL, filters, buckets and export details. Build the nav from that, call the report's URL,
  and render the returned buckets. Never assume a fixed column set — read `fields`.
- **Returned whole** — there is no pagination, sort, or include; a report is computed for the
  resolved period and returned in full.

---

## How reports are organised — parties, parent groups, presets

Every report lives in one backend registry, keyed by a stable string key, and the report nav is
that registry's tree: **party → parent group → report → preset-filter child**. Three kinds of node
appear in it, and only one of them is a report:

| Node | What it is | URL | Permissions |
|---|---|---|---|
| **Report** | A computed report with its own endpoint, filters and export. | Its own: `…/reports/{party-segment}[/{parent}]/{key}` | Its own: `view-{key}-report` / `export-{key}-report` |
| **Parent group** | A navigation heading holding reports and presets. Nav-only — no endpoint, no service, no filters. | **None** | **None** — show it when at least one child is visible |
| **Preset-filter child** | A link that opens an existing report with a fixed filter payload pre-applied. | Its **report's** URL | Its **report's** — none of its own |

### Parties

The top level of the tree. A party is the audience a report is about, and it fixes the URL
segment its reports sit under:

| Party | URL segment | Holds today |
|---|---|---|
| `landlord` | `landlords` | Income & Expenditure, Facility Budget |
| `supplier` | `suppliers` | parent group `expenditure-reports` (Property Expenses, Expenses Summary, Bill Submission, Bill Payment); Supplier Withholding (one report across every withholding tax, narrowed by `withholding_tax_id`); Supplier Contracts; Input VAT Analysis |
| `tenant` | `tenants` | Billings & Collections, parent groups `collections-reports`, `tenancy-reports` (Tenancy Schedule), `tenant-withholding` |
| `system` | `system` | system reports (audit trail and the like) |
| `accounting` | `accounting` | Trial Balance — a company-wide snapshot, not a party report |

> Party names are singular (`landlord`, not `landlords`) but the URL segments stay plural, so no
> existing report URL moved when the parties were renamed.

### Parent groups

A parent group has a key and a label and **nothing else**: no endpoint, no filters, no
permission. It lists its children in display order, and its child reports are served **under its
own segment** — a report `foo` inside `expenditure-reports` is
`GET …/reports/suppliers/expenditure-reports/foo` and exports at `…/foo/export`. A report with no
parent stays top-level under its party, exactly as before.

Four parent groups exist: `expenditure-reports` (supplier); `collections-reports`,
`tenancy-reports` and `tenant-withholding` (tenant). Reports are added beneath them ticket by
ticket; a group with no visible child is simply not shown. `supplier-withholding` began as a group
and is now the [Supplier Withholding](./supplier-withholding.md) report itself. It holds no presets:
withholding taxes are data, so the report covers them all and `withholding_tax_id` narrows it.

### Deprecated paths

Two reports moved into parent groups. Their old URLs **still answer** — same controllers, same
permissions, same payload — but are deprecated aliases kept only so existing integrations do not
break. Migrate to the canonical path; it is the only one `GET /reports` lists.

| Report | Deprecated (still works) | Canonical |
|---|---|---|
| Property Expenses | `GET …/reports/landlords/property-expenses` and `…/export` | `GET …/reports/suppliers/expenditure-reports/property-expenses` and `…/export` |
| Tenancy Schedule | `GET …/reports/tenants/tenancy-schedule` and `…/export` | `GET …/reports/tenants/tenancy-reports/tenancy-schedule` and `…/export` |

The permission names (`view-property-expenses-report`, `export-property-expenses-report`,
`view-tenancy-schedule-report`, `export-tenancy-schedule-report`) are unchanged and gate both
spellings identically. No new permission was introduced.

### Preset-filter children

Some nav entries are **not separate reports** but one report seen through a fixed filter — a
withholding report split into an entry per withholding tax, say. The registry expresses that as a
**preset-filter child**: a node with a key, a label, the `report` it opens and the `filters` to
apply.

To render one, call the **report's** URL with the preset's filter payload as query params,
exactly as if the user had picked those filters — so a preset `{ "withholding_tax_id": 3 }` on the
`supplier-withholding` report is `GET …/reports/suppliers/supplier-withholding?withholding_tax_id=3`,
and its export is the same call on `…/export` with `format` appended. The user may change the
filters afterwards; the preset is only a starting point.

A preset has **no route and no permission of its own**. It is visible exactly when the user holds
`view-{report}-report` for the report it opens, and exportable under that report's
`export-{report}-report`. Never look for a `view-{preset-key}-report` permission — none exists.

Presets suit fixed, code-defined views. They are **not** used for values that live in the database:
[Supplier Withholding](./supplier-withholding.md) has no per-tax presets, because a preset would bake a
tax id into the registry and a tax added later would never appear. It reports every tax and takes an
optional `withholding_tax_id` instead. The preset example further down is illustrative.

---

## Discovering reports — `GET /reports`

```
GET /api/v1/app/{company}/property-management/reports
```

**Do not hardcode the report list.** This endpoint returns the registry tree above, already
filtered to what the caller may see in the route company, with everything a client needs to call
each report: URL, permissions, filter descriptors, buckets and columns, export details and the doc
page. Build the nav from it, and read a report's filters and columns from it. It is the same
catalogue an MCP server will consume, so it describes reports completely rather than minimally.

- **Any authenticated app user may call it.** There is no permission for the endpoint itself;
  authorization is the filtering. No filters, no pagination.
- **A report is present only when the user holds its `view-{key}-report`** in the route company.
  A preset-filter child follows the report it opens. A parent group — or a whole party — left
  with no visible child is **dropped**, never returned empty. A user with no report permission
  gets `{ "data": [] }`.
- **Order is display order:** parties in the order below, top-level nodes in registry order
  within a party, children in their parent's declared order.
- **URLs are relative to the API host** (`/api/v1/app/{company}/…`) and already carry the route
  company, so a different company answers with different URLs — and a different tree, since
  permissions are company-scoped.

### Response

```jsonc
{
  "data": [                                   // parties, in display order; absent when empty
    {
      "type": "party",
      "key": "landlord",
      "label": "Landlord Reports",
      "children": [ /* groups, reports and presets, in display order */ ]
    }
  ]
}
```

Every node carries `type`, `key` and `label`. What else it carries depends on `type`:

| `type` | Extra keys | Notes |
|---|---|---|
| `party` | `children` | Top level. Never has a URL or permissions. |
| `group` | `children` | A parent group. Never has a URL or permissions; present only with at least one visible child. |
| `report` | `description`, `party`, `url`, `permissions`, `can_export`, `export`, `filters`, `buckets`, `summary`, `docs`, `children` | The only node with an endpoint. `children` holds its preset-filter children, `[]` when none. |
| `preset` | `report`, `url`, `permissions`, `can_export`, `export`, `preset_filters` | A preset-filter child. `url`, `permissions` and `export` are its **report's**; `report` is that report's key. Never has `children`. |

#### A report node

```jsonc
{
  "type": "report",
  "key": "property-expenses",
  "label": "Property Expenses",
  "description": "A per-property register of expenses (supplier bills) for the period …",
  "party": "supplier",
  "url": "/api/v1/app/1/property-management/reports/suppliers/expenditure-reports/property-expenses",
  "permissions": { "view": "view-property-expenses-report", "export": "export-property-expenses-report" },
  "can_export": true,                         // the caller holds permissions.export
  "export": {
    "url": "/api/v1/app/1/property-management/reports/suppliers/expenditure-reports/property-expenses/export",
    "param": "format",                        // append ?format=… to the report's own query string
    "formats": ["excel", "pdf"]
  },
  "filters": [ /* one descriptor per accepted query param, in order — see below */ ],
  "buckets": [ /* one entry per bucket the report can return — see below */ ],
  "summary": { "total_amount": "Sum of expense amounts before tax.", "total_tax": "…", "total": "…", "expense_count": "…" },
  "docs": "docs/frontend/api/v1/app/property-management/reports/property-expenses.md",
  "children": []
}
```

- `can_export` is the only per-user flag: the report is present because the user can view it,
  and `can_export` says whether to show the Export button. Both permission names are still
  listed so a client can explain what is missing.
- `export.url` takes **every filter the report takes** plus `format`; the frontend exports what
  is on screen by replaying the report's query string with `format` appended (see
  [The export endpoint](#the-export-endpoint)).
- `summary` maps each key of the report's `data.summary` to what it means. Per-bucket
  `summary` objects use the same keys unless the report's page says otherwise.
- `docs` is the report's contract page in this repository — the narrative that this node
  summarises.

#### A filter descriptor

One per query param the report accepts, **in the order the report declares them**. Derived from
the report's validation class, so it cannot drift from what the endpoint actually accepts.

```jsonc
{
  "name": "facility_id",
  "type": "integer",                          // integer | string | date | enum | boolean | number
  "required": false,                          // true only for a param the endpoint rejects when absent
  "default": "All properties",                // what omitting it means — prose, not a literal value
  "description": "One property. When set the summaries use that property's reporting currency; …",
  "references": { "table": "facilities", "column": "id" },   // present when the value is an entity id
  "required_when": "facility_id is omitted",  // present when a param is conditionally required (422 otherwise)
  "values": ["weekly", "monthly", "quarterly", "annually"],  // present for type: enum — the allowed literals
  "format": "Y-m-d"                           // present for type: date
}
```

`references`, `required_when`, `values` and `format` are **omitted when not applicable**; the
other five keys are always present (`default` and `description` may be `null` for a filter
nobody has annotated yet).

#### A bucket entry

One per bucket the report can return, in the order they appear in the response. Data-driven
buckets are listed once with a placeholder id (`property_{id}`) and a `when` that says how many
to expect.

```jsonc
{
  "bucket": "property_{id}",                  // the response `bucket` id, or its pattern
  "label": "The property's name",             // what header.label will say
  "when": "one per property in scope, sorted by name",   // "always", or the condition it appears under
  "description": "That property's expenses one per row plus a totals row, in its reporting currency …",
  "dynamic_columns": null,                    // prose describing runtime columns, or null when the columns are fixed
  "fields": [ /* the static column definitions, exactly as the response will carry them */ ]
}
```

`fields` are the [column definitions](#fields--column-definitions) the report declares up front
(`label`, `key`, `format`, `type`, `weight`, `background_color`, `alignment`, `visible`,
`togglable`). Where `dynamic_columns` is set, the live response splices extra columns in at the
position it describes — **always render from the response's own `fields`**; the catalogue tells
you what to expect, the response tells you what you got.

#### A preset-filter child

```jsonc
{
  "type": "preset",
  "key": "supplier-withholding-vat",
  "label": "VAT Withholding",
  "report": "supplier-withholding",
  "url": "/api/v1/app/1/property-management/reports/suppliers/supplier-withholding",
  "permissions": { "view": "view-supplier-withholding-report", "export": "export-supplier-withholding-report" },
  "can_export": true,
  "export": { "url": "…/supplier-withholding/export", "param": "format", "formats": ["excel", "pdf"] },
  "preset_filters": { "withholding_tax_id": 3 }
}
```

To open it: `GET {url}?{preset_filters as query params}` — here
`…/supplier-withholding?withholding_tax_id=3`. The filter descriptors for those params live on
the `report` node of the same key (find it by `report`); a preset carries only the values to
pre-fill.

---

## Permissions & export

Every report has **two** permissions, both named after the report's own slug:

| Permission | Guards |
|---|---|
| `view-{report}-report` | `GET {report-url}` — the report itself. Missing ⇒ `403`. |
| `export-{report}-report` | `GET {report-url}/export` — the export. Missing ⇒ `403`. |

The slugs are `income-and-expenditure`, `property-expenses`, `facility-budget`,
`billings-and-collections`, `tenancy-schedule`, `trial-balance` — so e.g. the Tenancy Schedule needs
`view-tenancy-schedule-report` and `export-tenancy-schedule-report`. Viewing does **not** imply
exporting: they are granted independently on a role, so the UI should hide the report and the Export
button separately. Both are listed by
`GET /api/v1/app/{company}/access-management/permissions` under the `Property Management` tag.

Only **reports** have permissions. A parent group has none — hide it when none of its children is
visible — and a preset-filter child is governed by the permissions of the report it opens (see
[How reports are organised](#how-reports-are-organised--parties-parent-groups-presets)).

### Seeding the permissions

A fresh install gets every report permission from the permissions catalogue. An install seeded before
a report landed is missing that report's two permissions, and the report answers `403` for everyone
until they exist. To create the missing ones:

```
php artisan db:seed --class=ReportPermissionsSeeder
```

It takes its list from the report registry, so it covers every report without being edited, and reads
each definition from the same catalogue a fresh install seeds. It is idempotent, and it **grants
nothing**: who may view or export which report is assigned to roles in access management afterwards.

### The export endpoint

Every report has an `…/export` sibling at its own URL:

```
GET …/reports/landlords/facility-budget/export?format=excel
GET …/reports/tenants/tenancy-reports/tenancy-schedule/export?format=pdf&facility_id=1&period_from=2026-07-01
```

- **It accepts every filter its report accepts** — same flat param names, same defaults, including
  that report's extra filters. The export is a rendering of the same generated report, so the
  frontend exports exactly what the user is looking at by **replaying the current query string with
  `format` appended**.
- `format` is **required** and must be `excel` or `pdf`. Anything else ⇒ `422` on `format`.

> Exports return an attachment immediately. Use `format=excel` for an `.xlsx` workbook or
> `format=pdf` for a PDF. Filenames follow
> `{report-slug}_{company}[_{landlord}][_{property}]_{period-from}_{period-to}`; Trial Balance uses
> `{report-slug}_{company}_as-at_{as-at}`. The `landlord` and `property` segments are slugged names
> and only appear when the export was filtered by `landlord_id` / `facility_id` (either or both), so
> a filtered download is identifiable without opening it. The response uses `attachment`
> `Content-Disposition` and the corresponding Excel or PDF `Content-Type`.
> Each report bucket is a separate worksheet in Excel and a separate page in PDF.

#### The PDF first page

> A PDF opens with the company's letterhead — logo (inlined from the stored file), name, tagline and
> contact details, all taken from the company the export was made under — and, under it, a panel
> naming what the report is: **Landlord** / **Property** on the left, **Report** / **Period** on the
> right. The landlord and property lines come from `header.landlord.name` / `header.property.name`,
> so they are printed only when the export was filtered down to one, and any company detail the
> company has not filled in is left out too. **Period** is the resolved window formatted
> `01 Aug, 2026 – 31 Aug, 2026`, suffixed with the cycle when the report declares one
> (`header.filters.report_type` / `report_cycle`) or when the window is exactly one calendar week,
> month, quarter or year; Trial Balance shows `As at 19 Aug, 2026` instead. The Excel workbook has no
> letterhead — each sheet still starts at its bucket label — so the filename is what identifies it.

### Large reports

> Exports with at least `REPORT_EXPORT_STREAM_ROW_THRESHOLD` rows (default: `5000`) use a streamed
> attachment response. The URL, filename, content type, and browser download behaviour do not change.
> There is no queued or polling workflow.

---

## Global filters

Every report supports these six filters (defined on the shared `App\Data\Reports\ReportFilterData`).
Individual reports may add their own on top (e.g. Billings & Collections adds `leases_status`,
`expense_category_id`, `lease_component_id`; Income & Expenditure adds `report_type` for period
grouping, `account_id`, and a `currency_id` required for multi-property roll-ups).

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_id` | `users.id` | All landlords |
| `facility_id` | `facilities.id` | All properties |
| `facility_type_id` | `facility_types.id` | All property types |
| `properties_status` | enum `App\Enums\Status` (`active`, `inactive`, …) | Any property status |
| `period_from` | date `Y-m-d` | **Start of the current month** |
| `period_to` | date `Y-m-d` | **End of the current month** |

Rules that hold for **all** reports:

- All filters are **optional** and passed as **flat** query params (not nested under `filter[...]`).
- `null` means "all" (for ids/types) or "any" (for statuses).
- The period defaults to the **current month** when `period_from` / `period_to` are omitted.
- **Currency is never a filter.** It is resolved from the property's reporting currency
  (`facilities.reporting_currency_id`) and surfaced in the `header`.

Example:

```
GET …/reports/tenants/billings-and-collections?facility_id=1&period_from=2026-07-01&period_to=2026-07-31
```

---

## Shared conventions

Rules that several reports depend on, held in one place so every report resolves them the same
way. A report page that relies on one of these links here rather than restating it.

### Expense categories by role — rent, service charge, service

Some reports must know which expenses are **service charge** expenditure (Excess Service Charge,
Service Charge vs Expenses Summary), and which are rent or service. There is no flag on an expense
for this: it is decided by the expense's **category**. Categories are rows a company owns and may
rename, so the roles are **company settings**, editable in Settings under Property Management →
Finance, alongside the default vendor and default tax:

| Setting key | Role | Defaults to |
|---|---|---|
| `rent_expense_category_id` | Rent | the company's category named "Rent" |
| `service_charge_expense_category_id` | Service Charge | the company's category named "Service Charge" |
| `service_expense_category_id` | Service | the company's category named "Service" |

Each holds one `expense_categories.id` and is rendered as a `select` over the company's expense
categories. New companies get all three seeded; existing ones were backfilled. A setting left
blank, or pointing at a deleted category, means the role is **unconfigured**: reports that depend
on it return nothing for that role rather than falling back to every expense.

Nothing else in the codebase holds these ids. Backend consumers read them through
`ExpenseCategorySettings` (`App\Support\PropertyManagement`) and classify through
`ExpenseCategoryClassifier` (`App\Services\PropertyManagement\Finance`), keyed by
`App\Enums\ExpenseCategoryKind`; `FacilityExpense::ofCategoryKind()` applies the same constraint
to a query.

**The classification rule — read this before touching any service-charge figure:**

- An expense is classified by **its own** `facility_expenses.expense_category_id`. That column is
  set on every expense and is the only input to the decision.
- **Never** resolve the category through the expense type
  (`facility_expense_types.expense_category_id` / `expense_type.expense_category_id`). One expense
  type carries expenses of several categories — Plumbing Works under Repairs & Maintenance is a
  service charge expense, while Painting under that same type is a rent expense — so the type is
  not a proxy for the category. That mapping is a stale default slated for removal, and it is the
  more discoverable relation, so it is the one a new consumer finds first. An expense whose *type*
  sits under the service-charge category but whose *own* category does not is **not** service
  charge.
- A role counts **across every expense type** — the expense type never narrows it.

For the frontend this means a service-charge figure on any report is already classified
server-side; there is nothing to recompute from expense types, and no filter to send.

### Colouring occupancy, collection and budget health

Two colour rules recur across reports. They are different rules and are never swapped:

| Measure | Rule | Colours |
|---|---|---|
| Occupancy rate, collection rate | Fixed bands on the rate | above 80% `success` · 50–80% `warning` (both ends) · below 50% `danger`; nothing to measure → no `color` |
| Income realisation, expense utilisation | Budget judgement: time-prorated, per-property at-risk threshold, coloured by favourability | income `over` `success` / `under` `danger`; expense `over` `danger` / `under` `success`; `ok` → no `color` |

Both are computed server-side. **Render the returned `color`; never derive one from a percentage or a
status word.** [property-status.md](./property-status.md#the-two-colour-rules) walks through both.

---

## The response envelope

```jsonc
{
  "data": {
    "header":  { /* who/what/when the report is for */ },
    "report":  [ /* buckets, in display order */
      {
        "bucket":  "property_1",  /* stable machine id, unique within the report */
        "header":  { "label": "ACK Gardens" },  /* + any report-specific bucket info */
        "fields":  [ /* column definitions, in display order */ ],
        "items":   [ /* rows; keys match each field's `key` */ ],
        "summary": { /* bucket-level totals for headline KPI tiles; may be {} */ }
      }
    ],
    "summary": { /* report-level totals for headline KPI tiles */ }
  }
}
```

`report` is an **array**, never an object keyed by bucket id. Buckets vary per report and some are
data-driven (Income & Expenditure emits one per property in scope), so a bucket carries its own
identity and label rather than the renderer deriving them from a key. **Array order is display
order.** A report with no buckets returns `[]`.

### `header`

- `property` — `{ id, name, currency: { code, name } }`, or `null` when no single property is
  selected. **Currency lives on the property** (its reporting currency). When `property` is `null`,
  a top-level `header.currency` fallback (the company default currency, or a report-selected currency)
  is included instead.
- `landlord` — `{ id, name }` when a `landlord_id` filter is supplied, otherwise `null`. Present on
  every report that accepts `landlord_id` (i.e. all but Trial Balance).
- `period` — `{ from, to }` (the resolved period, after defaults).
- `filters` — every filter echoed back (nulls included) so the UI can show the active scope.
- `generated_at` — ISO-8601 timestamp.
- A report **may add** header fields — e.g. Trial Balance adds `as_at` and `notes` in place of
  `period`. Per-report pages document any such additions.

> Not to be confused with a **bucket's** `header` (below), which describes one bucket. This one
> describes the whole report.

### A bucket

Every bucket carries these five keys, and **all five are always present** — buckets are exempt from
the lean-payload convention that governs cells and rows (below). There are only a handful of buckets
per report, so predictability beats leanness here.

| Key | Type | Notes |
|---|---|---|
| `bucket` | string | Stable machine id, unique within the report (e.g. `overview`, `property_1`, `overall`). Use it as a React key / deep-link target — **don't parse it for meaning**. |
| `header` | object | Info about this bucket. Always carries `label` (see below); a report may add more. |
| `fields` | array | This bucket's column definitions, in display order. |
| `items` | array | This bucket's rows. |
| `summary` | object | Bucket-level KPI scalars. May be `{}` — never `[]`. |

`header.label` is the **only guaranteed key** and is always a non-empty human-readable string — a
generic renderer can title a tab or section from it without knowing the report. Everything else in
`header` is report-specific and documented on that report's page (e.g. Income & Expenditure adds
`property` on its per-property buckets).

Different buckets in one report **may have different columns** — always read each bucket's own
`fields`.

### `fields` — column definitions

Render your table columns from this array, **in order**. Each field:

| Key | Type | Notes |
|---|---|---|
| `label` | string | Column header text. |
| `key` | string | Matches the property holding the cell in each `items` row. Stable machine id (derived from the entity id, not its name). |
| `format` | `string` \| `integer` \| `money` | Drives value formatting. See [Formats](#formats). |
| `type` | `normal` \| `subtotal` \| `grosstotal` | Visual role of the column. See [Field & row type](#field--row-type). Default `normal`. |
| `weight` | Tailwind font weight | e.g. `font-normal`, `font-medium`, `font-bold`. Default `font-normal`. |
| `background_color` | color enum | Tints the **whole column**. See [Colors](#colors). Default `none`. |
| `alignment` | `left` \| `center` \| `right` | Convention: text → left, numbers → right. |
| `visible` | boolean | Initial visibility. Hidden columns should not render by default. |
| `togglable` | boolean | If `true`, offer it in a column picker; if `false`, it is fixed. |

**Dynamic columns:** some buckets append runtime columns (one per account or per component) with keys
like `account_2_billings`, `component_2`. These always come back `togglable: true` and carry an
explicit entity ref (`account_id` = `expense_categories.id`, `component_id` = `lease_components.id`)
so you can group or deep-link without parsing the key. **Treat `fields` as the source of truth.**

### `items` — rows

Each row is keyed by the field `key`s, **plus row-level metadata**:

- `type` — `normal` | `subtotal` | `grosstotal` (see below). Data rows are `normal`; the totals row
  is `subtotal` (or `grosstotal` where a report has a distinct grand total). Always present.
- `background_color` — color enum tinting the **whole row** (e.g. `secondary` on the totals row).
  **Optional — omitted when `none`.** A missing key means no row tint.

A row **need not contain every field key.** A key may be omitted because a preceding cell spans it
(`col_span`, below) or because the row simply doesn't populate it — render such columns empty.

Every **cell** (each field-keyed value) is an object. Only `value` is guaranteed:

```jsonc
{ "value": 4800, "color": "danger" }   // fully specified
{ "value": 4800 }                      // no special colour, single column
{ "value": "Income Received", "col_span": 5 }  // spans 5 columns
```

- `value` — the raw value; format it by the field's `format`. A `money` value may be a number or a
  decimal string — coerce to number before formatting.
- `color` — a color enum applied to that individual cell (e.g. an overdue balance in `danger`).
  **Optional — omitted when `none`.** A missing key means no cell colour.
- `col_span` — a positive integer > 1; the cell occupies that many columns and the row **omits** the
  next `col_span − 1` field keys. **Optional — omitted when `1`** (the default). Used for full-width
  banner rows (e.g. a section header spanning the whole table).

> **Payload convention:** absent `color` / `background_color` ⇒ `none`; absent `col_span` ⇒ `1`. The
> backend drops these keys when they hold their default to keep the response small — never assume a
> key is present, and never treat its absence as anything but the default.

### `summary`

Totals for headline tiles (plain scalars, **not** cells) — e.g. `lease_count`, `total_billings`,
`total_collections`. It appears in two places, with the same shape:

- **`data.summary`** — report-level, across everything in scope.
- **`report[i].summary`** — that bucket's own totals. Always present; may be `{}` when the bucket has
  no meaningful headline number.

---

## Presentation reference

Three placements share **one color palette**, so a dev learns the meanings once:

- **cell `color`** — a single value,
- **field `background_color`** — a whole column,
- **row `background_color`** — a whole row.

### Formats

| `format` | Meaning |
|---|---|
| `string` | Plain text. Left-align. |
| `integer` | Whole number. |
| `money` | Decimal amount in **currency units** (not cents), formatted with the `header` currency. Right-align. |

### Field & row type

| `type` | Meaning | Suggested emphasis |
|---|---|---|
| `normal` | Ordinary data column / data row. | none |
| `subtotal` | A section/period total (e.g. Total Billings) or the per-bucket totals row. | medium weight, tinted row |
| `grosstotal` | A grand / net total (e.g. Gross Total, Balance C/F) or a final grand-total row. | bold, stronger tint |

`type` appears on **both** fields (the column's role) and rows (the row's role). The backend also sets
`weight` on emphasised columns; the template may add its own emphasis keyed off `type`.

**How the Excel / PDF exports render it** — worth matching on screen so an export looks like the report
it came from:

| `type` | Weight | Size |
|---|---|---|
| `grosstotal` | bold (700) | one step above `subtotal` |
| `subtotal` | semi-bold (600) | one step above normal |
| `normal` | the field's declared `weight`, or normal | normal |

A cell takes the **strongest** tier of its own `type`, its column's and its row's — so an ordinary
column inside a grand-total row is still bold. A `normal` cell is only ever emphasised by an explicit
`weight` on its field, and a declared `weight` never changes the size. (In xlsx the semi-bold step is
unavailable — its font model is bold or not — so there the size step alone separates the two tiers.)

### Colors

The enum is `none | primary | secondary | success | danger | warning | info`. Meanings and a
**suggested** Tailwind mapping (adopt one mapping in the template so every report looks consistent —
the backend sends the semantic token, the frontend owns the exact classes):

| Color | Meaning (semantic) | Suggested text class | Suggested bg class |
|---|---|---|---|
| `none` | Default — no emphasis. | inherit | none |
| `primary` | Neutral emphasis / brand highlight. | `text-primary-600` | `bg-primary-50` |
| `secondary` | Muted band, e.g. a totals/footer row. | `text-gray-700` | `bg-gray-100` |
| `success` | Positive — paid, in credit, on target. | `text-green-600` | `bg-green-50` |
| `danger` | Negative — overdue, in arrears, negative balance. | `text-red-600` | `bg-red-50` |
| `warning` | Caution — due soon, partial. | `text-amber-600` | `bg-amber-50` |
| `info` | Informational highlight. | `text-blue-600` | `bg-blue-50` |

### Weight

`weight` is a Tailwind font-weight token (`font-thin` … `font-black`, commonly `font-normal`,
`font-medium`, `font-bold`). The backend applies heavier weights to `subtotal` / `grosstotal` columns.

> The backend chooses `format`, `color`, `weight`, and `background_color` per report — they are
> report-specific presentation choices, not a fixed global scheme. The template must render whatever
> the payload declares, not assume a set.

---

## How to render (report template)

The generic algorithm the reusable frontend template implements, for any report:

1. Read `header.property.currency` (or top-level `header.currency` fallback) for money formatting;
   show `header.period` and active `header.filters` as the report scope.
2. Iterate `report` **in array order** — that is the display order. Title each bucket from its
   `header.label`; key it by `bucket`. Then build that bucket's columns from its own `fields`
   **in order**; skip `visible: false` columns by default and expose `togglable: true` columns in a
   column picker.
3. For each row in `items`:
   - apply the row's `background_color` if present (whole-row tint); absent ⇒ no tint.
   - walk the `fields` in order keeping a **skip counter**. For each field:
     - if the skip counter is positive, decrement it and render nothing (a previous cell spans here);
     - else read the row's cell for that field `key`. If the key is **absent**, render an empty cell.
       Otherwise apply (in order) the column's `background_color`, then the cell's `color` if present,
       then the field's `weight` and `alignment`; format `value` by the field's `format`. If the cell
       has `col_span: N`, render it across N columns and set the skip counter to `N − 1`.
   - render `type: "subtotal"` / `type: "grosstotal"` rows as footer / emphasised rows.
4. Use `data.summary` for the report's headline KPI tiles, and each bucket's `summary` for its own.

Remember: a missing `color` / `background_color` means `none`, and a missing `col_span` means `1`.

**Never hardcode columns or a fixed palette — `fields` and the payload are the source of truth.**

For a concrete, worked example see [billings-and-collections.md](./billings-and-collections.md).
