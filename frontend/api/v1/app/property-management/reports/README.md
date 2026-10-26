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
> accounting snapshot rather than a party report: it takes **none** of the global filters (no
> landlord, property, bucket or block/wing/floor selection) and has no `period` in its `header` —
> only `as_at` and `currency_id`. Everything else on this page still applies to it unchanged.

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
| `tenant` | `tenants` | Billings & Collections, Tenant Deposits, Ageing Report, Debtors Listing, Unbilled Leases, Output VAT Analysis; Tenant Withholding (one report across both withholding methods, with a preset child per method); parent group `collections-reports` (Property Collections, Collections Summary, Legal Fee Collection); parent group `tenancy-reports` (Tenancy Schedule, Summary Occupancy) |
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

One per query param the report accepts. Derived from the report's validation class, so it cannot
drift from what the endpoint actually accepts.

```jsonc
{
  "name": "currency_id",
  "type": "integer",                          // integer | string | date | enum | boolean | number | array
  "required": false,                          // true only for a param the endpoint rejects when absent
  "default": "Each property's reporting currency",   // what omitting it means — prose, not a literal value
  "description": "The currency multi-property figures are converted to; …",
  "references": { "table": "currencies", "column": "id" },   // present when the value is an entity id
  "required_when": "no single property is selected (facility_ids has zero or several entries)", // present when a param is conditionally required (422 otherwise)
  "values": ["weekly", "monthly", "quarterly", "annually"],  // present for type: enum — the allowed literals
  "format": "Y-m-d"                           // present for type: date
}
```

A **list filter** (see [Global filters](#global-filters)) is described with `type: "array"`:

```jsonc
{
  "name": "facility_ids",
  "type": "array",
  "items": { "type": "integer" },             // the type of each element
  "multiple": true,                           // render a multi-select
  "required": false,
  "default": "All properties",
  "description": "The properties to report on. …",
  "references": { "table": "facilities", "column": "id" }    // what each element refers to
}
```

```jsonc
{
  "name": "buckets",
  "type": "array",
  "items": { "type": "string" },
  "multiple": true,
  "required": false,
  "default": "Every bucket",
  "description": "Which buckets to return. …",
  "values": ["overview", "property_{id}"]     // this report's declared bucket ids and patterns
}
```

A superseded filter that is still accepted carries `deprecated` and the filter that replaced it.
Don't render it as its own control — the replacement covers it:

```jsonc
{ "name": "facility_id", "type": "integer", "required": false, "deprecated": true, "replaced_by": "facility_ids", … }
```

`references`, `required_when`, `values`, `format`, `items`, `multiple`, `deprecated` and
`replaced_by` are **omitted when not applicable**; the other five keys are always present
(`default` and `description` may be `null` for a filter nobody has annotated yet).

**Order.** The params the report's validation class declares come first, in declaration order
(the global scalars, then the report's own). The list filters follow, always in this order:
`landlord_ids`, `facility_ids`, `buckets`, then — on the fourteen
[tenant reports](#splitting-by-block--wing--floor) only — `facility_block_ids`,
`facility_wing_ids`, `facility_floor_ids`.

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
GET …/reports/tenants/tenancy-reports/tenancy-schedule/export?format=pdf&facility_ids[]=1&period_from=2026-07-01
```

- **It accepts every filter its report accepts** — same flat param names, same defaults, including
  that report's extra filters. The export is a rendering of the same generated report, so the
  frontend exports exactly what the user is looking at by **replaying the current query string with
  `format` appended**.
- `format` is **required** and must be `excel` or `pdf`. Anything else ⇒ `422` on `format`.

> Exports return an attachment immediately. Use `format=excel` for an `.xlsx` workbook or
> `format=pdf` for a PDF. Filenames follow
> `{report-slug}_{company}[_{landlord}][_{property}]_{period-from}_{period-to}`; Trial Balance uses
> `{report-slug}_{company}_as-at_{as-at}`. The `landlord` and `property` segments only appear when
> the export was filtered by landlord / property (either or both — `landlord_ids` / `facility_ids`
> or their deprecated aliases), so a filtered download is identifiable without opening it. With
> **one** landlord selected the segment is the landlord's slugged name; with **several** it is
> `{n}-landlords` (`3-landlords`). The property segment works the same way: the slugged name, or
> `{n}-properties`. The response uses `attachment` `Content-Disposition` and the corresponding
> Excel or PDF `Content-Type`.
> Each report bucket is a separate worksheet in Excel and a separate page in PDF. When the request
> sends [`buckets`](#selecting-buckets--buckets), only those buckets become sheets / pages.
>
> Money cells are exported as bare amounts, never prefixed with a currency code — in Excel as real
> numbers (`money` columns carry the `#,##0.00` number format, `integer` columns `#,##0`), so the sheet
> can be summed, sorted and charted; in the PDF as `1,250.50`. The
> currency is named once per bucket instead: a bucket with money columns is titled with its currency
> appended — `Summary (KES)` — using the bucket's own `header.currency` when it has one, else the
> report's (`header.property.currency`, then `header.currency`). The Excel sheet's first row carries
> the same title. The PDF shrinks a wide bucket's table text (down to 5px from the normal 10px) so
> every column fits the page width; padding and the total-row size steps shrink with it.

#### The PDF first page and footer

> A PDF opens with a header: the logo of the company the export was made under on the left (inlined
> from the stored file; the company name is printed in its place when there is no logo), and the
> document's name — the report's registry label, e.g. `Tenant Statement` — on the right. Under it, an
> identity band says what the report covers: **Landlord** / **Property** on the left, **Period** /
> **Currency** on the right, printed in the company's `text_color_on_brand_bg` on its `brand_color`
> (falling back to `config('app.company_palette')` — env `COMPANY_TEXT_COLOR_ON_BRAND_BG` `#ffffff` /
> `COMPANY_BRAND_COLOR` `#005044`). The footer's services strip uses the company's `accent_color`.
>
> A statement can lay the band out itself: its envelope header carries `cover.left` / `cover.right`,
> each a list of `{label, value}` lines or `{label, fact}` lines, where `fact` is `period` (the
> window alone, no cycle) or `currency`. It can also set `bucket_titles: false` to drop the title above
> its table, and `date_format` to print its dates its own way (the statements use `d-M-y`).
>
> Without a declared layout, the band is laid out as follows. With **one** landlord / property
> selected, a **Landlord** / **Property** line prints its name (`header.landlord.name` /
> `header.property.name`). With **several**, the line is titled **Landlords** / **Properties** and
> lists the selected names from `header.landlords` / `header.properties`, shortened past a few to
> `A, B and 3 more`. Nothing selected prints no line. On the
> [tenant reports](#splitting-by-block--wing--floor), **Blocks**, **Wings** and **Floors** lines list
> the selected nodes the same way, each only when that filter was used. **Period** is the resolved window formatted
> `01 Aug, 2026 – 31 Aug, 2026`, suffixed with the cycle when the report declares one
> (`header.filters.report_type` / `report_cycle`) or when the window is exactly one calendar week,
> month, quarter or year; Trial Balance shows `As at 19 Aug, 2026` instead. **Currency** is the
> report's currency as `Kenyan Shilling (KES)` (the bare code when it has no name), omitted when the
> header carries none.
>
> Every bucket ends with the same printout footer documents carry — the company's services strip,
> its contact line (name, postal address, phone, email, website; blanks left out) and the
> `Printed at` / `Powered by` stamp. It is printed once per bucket, at the bottom of that bucket's
> last page; when the bucket leaves too little room for it, it moves to the bottom of the next page.
>
> The Excel workbook has neither — each sheet still starts at its bucket label — so the filename is
> what identifies it.

### Large reports

> Exports with at least `REPORT_EXPORT_STREAM_ROW_THRESHOLD` rows (default: `5000`) use a streamed
> attachment response. The URL, filename, content type, and browser download behaviour do not change.
> There is no queued or polling workflow.

---

## Global filters

Every report except Trial Balance supports these filters (defined on the shared
`App\Data\Reports\ReportFilterData`). Individual reports may add their own on top (e.g. Billings &
Collections adds `leases_status`, `expense_category_id`, `lease_component_id`; Income & Expenditure
adds `report_type` for period grouping, `account_id`, and a `currency_id` required for
multi-property roll-ups). The fourteen tenant reports also take the block / wing / floor lists — see
[Splitting by block / wing / floor](#splitting-by-block--wing--floor).

| Param | Type | Meaning when omitted |
|---|---|---|
| `landlord_ids` | list of `users.id` | All landlords |
| `facility_ids` | list of `facilities.id` | All properties |
| `facility_type_id` | `facility_types.id` | All property types |
| `properties_status` | enum `App\Enums\Status` (`active`, `inactive`, …) | Any property status |
| `period_from` | date `Y-m-d` | **Start of the current month** |
| `period_to` | date `Y-m-d` | **End of the current month** |
| `buckets` | list of bucket ids | Every bucket — see [Selecting buckets](#selecting-buckets--buckets) |
| `landlord_id` | `users.id` — **deprecated**, use `landlord_ids` | — |
| `facility_id` | `facilities.id` — **deprecated**, use `facility_ids` | — |

Rules that hold for **all** reports:

- All filters are **optional** and passed as **flat** query params (not nested under `filter[...]`).
- `null` (or an empty list) means "all" (for ids/types) or "any" (for statuses).
- The period defaults to the **current month** when `period_from` / `period_to` are omitted.
- **Currency is never a filter.** It is resolved from the property's reporting currency
  (`facilities.reporting_currency_id`) and surfaced in the `header`. (A few landlord and supplier
  reports take a `currency_id` to convert a multi-property roll-up into — that is a conversion
  target, not a filter.)

Example:

```
GET …/reports/tenants/billings-and-collections?facility_ids[]=1&period_from=2026-07-01&period_to=2026-07-31
```

### List filters

`landlord_ids`, `facility_ids`, `buckets` and the tenant reports' `facility_block_ids`,
`facility_wing_ids`, `facility_floor_ids` each take **several** values. A list may be sent three
ways, all equivalent:

```
?facility_ids[]=1&facility_ids[]=2      // repeated key[] — what axios and most HTTP clients send for an array
?facility_ids=1,2                       // one comma-separated string
?facility_ids=1                         // a single scalar, read as a one-element list
```

- **At most 500 ids** per list. More ⇒ `422` on the list's name.
- **Every id must exist.** One unknown id fails the whole list with a `422` keyed on the list's
  name, never on an element:

  ```json
  {
    "message": "The properties contains unknown ids.",
    "errors": { "facility_ids": ["The properties contains unknown ids."] }
  }
  ```

- Values are combined with **OR** within a list and **AND** across filters: `landlord_ids[]=4&
  landlord_ids[]=7&facility_type_id=2` is "properties of landlord 4 or 7 that are of type 2".
- Duplicates are ignored, and order does not matter.

### The deprecated `landlord_id` / `facility_id`

The single-value `landlord_id` and `facility_id` are **still accepted** as aliases, so existing
links, presets and integrations keep working. Each is **merged into its list** (a union):
`facility_id=3&facility_ids[]=5` selects properties 3 and 5. Both are still echoed in
`header.filters`, and discovery lists them with `"deprecated": true` and
`"replaced_by": "landlord_ids"` / `"facility_ids"`. New code should send the lists only.

### Exactly one property

Several reports behave differently when the user is looking at **one** property. That condition is
now **exactly one property selected** — `facility_ids` with one entry, or the `facility_id` alias
on its own (or both naming the same property). It is about what was **selected**, not what the
other filters happen to leave: a landlord who owns a single property still gives a
multi-property report unless that property is selected. Where it applies:

- `header.property` is set (with the property's reporting currency) — see [`header`](#header).
- Billings & Collections omits `summary_by_property`; the Ageing Report omits
  `properties-summary`.
- Income & Expenditure, Facility Budget and Property Expenses **require `currency_id` unless
  exactly one property is selected** (`422` on `currency_id` otherwise), and use that property's
  reporting currency when it is.
- The block / wing / floor filters are accepted only then.

---

## Selecting buckets — `buckets`

By default a report returns every bucket it produces. `buckets` asks for **only some of them** —
e.g. a UI that shows one tab at a time, or an export of just the summary:

```
GET …/reports/tenants/billings-and-collections?facility_ids[]=1&buckets[]=overview&buckets[]=billings
```

- **Only `data.report` shrinks** — and with it the Excel sheets and PDF pages of an export.
  `header` and `data.summary` are **always computed for the whole filtered scope**, whichever
  buckets were asked for, so the headline tiles do not change when a tab is switched.
- Buckets keep the report's own order, not the order they were requested in.
- **Each id must be one the report declares** — a fixed bucket id (`overview`), or a match for a
  declared pattern: `property_{id}` accepts any `property_<integer>` (`property_12`). Patterns keep
  each report's own spelling — `property_{id}`, `property-{id}`, `landlord-{id}` — so match the
  `bucket` ids the report actually returns. Requesting the literal pattern
  (`buckets[]=property_{id}`) selects **every** bucket of that pattern.
  Discovery lists the declared ids and patterns as the `buckets` descriptor's `values`.
- An id the report declares but this run did not produce (a property out of scope, or
  `summary_by_property` on a single-property run) is **simply absent** — not an error.
- An id the report does not declare ⇒ `422` on that element:

  ```json
  {
    "message": "The bucket weekly_totals is not one this report returns.",
    "errors": { "buckets.1": ["The bucket weekly_totals is not one this report returns."] }
  }
  ```

---

## Splitting by block / wing / floor

The fourteen **tenant reports** can be narrowed to part of a property:
[Billings & Collections](./billings-and-collections.md),
[Property Collections](./property-collections.md),
[Collections Summary](./collections-summary.md),
[Legal Fee Collection](./legal-fee-collection.md),
[Tenancy Schedule](./tenancy-schedule.md),
[Summary Occupancy](./summary-occupancy.md),
[Tenant Deposits](./tenant-deposits.md),
[Ageing Report](./ageing-report.md),
[Debtors Listing](./debtors-listing.md),
[Unbilled Leases](./unbilled-leases.md),
[Output VAT Analysis](./vat-analysis-output.md),
[Tenant Withholding](./tenant-withholding.md),
[Billing & Collection Trend](./billing-collection-trend.md) and
[Occupancy Trend](./occupancy-trend.md). They take three more [list filters](#list-filters):

| Param | Type | Meaning when omitted |
|---|---|---|
| `facility_block_ids` | list of `facility_blocks.id` | The whole property |
| `facility_wing_ids` | list of `facility_wings.id` | The whole property |
| `facility_floor_ids` | list of `facility_floors.id` | The whole property |

These are **filters, not groupings.** They narrow which rows are counted; bucket shapes, row shapes
and Total rows stay exactly as documented on each report's page, and no block subtotal rows are
added. To compare blocks, run the report once per block — or select several in one run. A run with
none of these filters is the report as it always was.

Rules (each failure is a `422`):

- **Exactly one property must be selected** ([see above](#exactly-one-property)). A node filter
  with no property, or with several, fails on `facility_ids`:
  `"Select exactly one property to filter by block, wing or floor."`
- **Every node must belong to that property.** Otherwise the error is keyed on the list —
  `facility_block_ids`: `"The selected block does not belong to the selected property."` (and
  `wing` / `floor` likewise on `facility_wing_ids` / `facility_floor_ids`).
- **The property must be one the caller is allocated to.**
- Several nodes are a **union**: blocks A and B is everything in either. A selection **inside** a
  selected node **refines** it, so the pickers read as a drill-down: Block A plus Wing 1 of Block A
  is Wing 1 only, and a wing plus one of its floors is that floor. A block with nothing selected
  inside it is the whole block.

### Which node a row belongs to — the primary space

Billings carry no space (a billing is a lease and a component), and opening balances are per lease,
so a lease cannot be split across spaces without inventing figures. Instead **every row's lease is
placed whole under one node**, through its **primary space**:

1. the space on the lease item that holds the **Rent** component; else
2. the space on the lease item that holds the **Service Charge** component; else
3. the space on the lowest-id lease item that has a space.

The lease belongs to that space's floor, and through it to the floor's wing and block. Everything
else follows its lease:

- **Invoices and credit notes** — through their lease.
- **Receipts** — through the lease of the invoice on the receipt's **first** allocation.
- **Deposits, billings, collections, balances** — through their lease.

A lease with **no spaced item** (or whose primary space sits outside the lease's own property) is
**unassigned**: it is counted in a property-level run, and left out of **every** block / wing /
floor-filtered run. A soft-deleted floor still places its leases, so attribution does not move
when the structure is edited.

### What adds up

For a property split into Blocks A and B, for every money and count column of every bucket's Total
row, and for every money and count key of `data.summary`:

- **Block A run + Block B run + unassigned leases = the whole-property run.**
- **A run with Blocks A and B selected together = Block A run + Block B run.**

The same holds for wings and floors. **Rates and percentages are recomputed for each run, never
summed** — a collection rate for Block A is Block A's collections over Block A's billings. Stored
amounts carry five decimal places and a Total row rounds the raw sum, so two block runs may differ
from the property run **by a cent** in a rounded column; that is rounding, not a lost row.

Two reports split differently:

- **Summary Occupancy** counts spaces, not leases, so it filters spaces by their floor directly —
  exact by construction, no primary-space rule.
- **Tenancy Schedule** lists spaces too, so it splits **per space**, while the other reports split
  **per lease**. A lease holding a shop in Block A (with the Rent) and a parking bay in Block B
  shows the parking bay under Block B in the schedule, but all of its money under Block A in every
  other report.

### Picking blocks, wings and floors

```
GET /api/v1/app/{company}/property-management/reports/filter-options/structure?facility_id=1
```

Returns one property's blocks, wings and floors for the pickers. Any authenticated app user may
call it — like discovery, there is no permission of its own — but the property must be one the
caller is allocated to (`404` otherwise). `facility_id` is required.

```jsonc
{
  "data": {
    "facility": { "id": 1, "name": "ACK Gardens", "has_blocks": true, "has_wings": false },
    "blocks": [
      { "id": 11, "name": "Block A", "has_wings": true },
      { "id": 12, "name": "Block B", "has_wings": false }
    ],
    "wings": [
      { "id": 21, "name": "Wing 1", "facility_block_id": 11 }
    ],
    "floors": [
      { "id": 31, "name": "Basement", "facility_block_id": 12, "facility_wing_id": null },
      { "id": 32, "name": "Ground Floor", "facility_block_id": 11, "facility_wing_id": 21 },
      { "id": 33, "name": "First Floor", "facility_block_id": 11, "facility_wing_id": 21 }
    ]
  }
}
```

- Each list is ordered by name. A node hanging directly off the property has `null` parent ids.
- **Offer blocks** only when `facility.has_blocks`; **wings** when `facility.has_wings` or a
  selected block `has_wings`; **floors** always.
- Narrow the wing and floor pickers to the selected blocks / wings with the parent ids. Disable
  all three pickers unless exactly one property is selected — the report would reject them.

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

- `property` — `{ id, name, currency: { code, name } }` only when
  [exactly one property is selected](#exactly-one-property), otherwise `null` (several selected, or
  none). **Currency lives on the property** (its reporting currency). When `property` is `null`,
  a top-level `header.currency` fallback (the company default currency, or a report-selected currency)
  is included instead.
- `properties` — `[{ id, name }]`, the properties **selected** (`facility_ids` plus the
  `facility_id` alias). `[]` means none was selected, i.e. all properties — it is not the list of
  properties the report covered.
- `landlord` — `{ id, name }` only when exactly one landlord is selected, otherwise `null`.
- `landlords` — `[{ id, name }]`, the landlords selected (`landlord_ids` plus the `landlord_id`
  alias); `[]` means all.
- `blocks`, `wings`, `floors` — **tenant reports only**: `[{ id, name, facility_id }]`, the nodes
  selected by `facility_block_ids` / `facility_wing_ids` / `facility_floor_ids`; `[]` when that
  filter is unused.
- These are present on every report that accepts the global filters (i.e. all but Trial Balance).
  Use them to label the scope ("Properties: ACK Gardens, Beta House") without a lookup.
- `period` — `{ from, to }` (the resolved period, after defaults).
- `filters` — every filter echoed back (nulls included) so the UI can show the active scope. That
  includes the lists (`landlord_ids`, `facility_ids`, `buckets`, and on tenant reports
  `facility_block_ids`, `facility_wing_ids`, `facility_floor_ids`) **and** the deprecated
  `landlord_id` / `facility_id`. For the resolved selection read `landlords` / `properties` above,
  which already merge the aliases.
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
| `format` | `string` \| `id` \| `integer` \| `money` | Drives value formatting. See [Formats](#formats). |
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
| `id` | An identifier — a database id or a document number. **Print it exactly as given.** Left-align. |
| `integer` | A whole number you can count or sum. Group the thousands. |
| `money` | Decimal amount in **currency units** (not cents), formatted with the `header` currency. Right-align. |

**`id` and `integer` are both whole numbers and must not be rendered alike.** `integer` counts
things — 1,234 units — and reads better grouped. `id` names one thing, and grouping it produces
`1,234` for lease 1234, which is not a number the user can look up or type back. Anything that ends
in `_id`, plus document numbers such as `invoice_no`, is an `id`.

A renderer that does not know `id` should fall through to printing the raw value, which is already
the correct output — so this is safe to adopt late.

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
