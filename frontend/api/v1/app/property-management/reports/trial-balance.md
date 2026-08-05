# Trial Balance Report API

Domain: `Property Management > Reports > Accounting`

Base route:

`/api/v1/app/{company}/property-management/reports`

The Trial Balance is a **company-wide accounting snapshot as at a cut-off date**: what the company is
owed and holds in cash, against what it owes. It is **read-only** and computed on the fly.

> **Read [the shared reports contract](./README.md) first.** This page only documents what is specific
> to this report. The response envelope (`header` + a `report` array of buckets + `summary`), the
> bucket keys, the field keys, the per-cell `{ value, color }` shape, and the colour/format vocabulary
> all live in the README and apply here unchanged.

## Endpoints

- `GET /reports/accounting/trial-balance` — requires `view-trial-balance-report`
- `GET /reports/accounting/trial-balance/export` — requires `export-trial-balance-report`. Takes a
  required `format` (`excel` | `pdf`) plus the same filters as the report. Currently answers `501` —
  see [the shared contract](./README.md#permissions--export).

## What makes this report different

- **It is not a party report.** It sits under `accounting`, not `landlords` / `tenants` / `suppliers`,
  because it belongs to no single landlord, tenant or supplier. It therefore takes **none of the six
  [global filters](./README.md#global-filters)** — no `landlord_id`, no `facility_id`, no
  `facility_type_id`, no `properties_status`, no `period_from` / `period_to`.
- **A cut-off, not a period.** There is one date param, `as_at`.
- **One bucket, always.** The response always carries exactly one bucket, `trial_balance`, with three
  columns: `label`, `debit`, `credit`.
- **`header` has no `property` or `period`.** It carries `as_at`, `currency`, `filters`, `notes` and
  `generated_at` instead. A generic renderer that reads `header.period` must tolerate its absence.

## Get the report

`GET /api/v1/app/{company}/property-management/reports/accounting/trial-balance`

### Filters (query params)

Both are **optional** and flat (not nested under `filter[...]`).

| Param | Type | Meaning when omitted |
|---|---|---|
| `as_at` | date `Y-m-d` | **End of today.** The cut-off every figure is stated as at. |
| `currency_id` | `currencies.id` | The company's `default_currency` setting. Every figure is converted into this currency. |

If neither `currency_id` nor a configured `default_currency` resolves, the endpoint answers `422` on
`currency_id`.

```
GET …/reports/accounting/trial-balance?as_at=2026-06-30&currency_id=1
```

## What each line means

### Assets (the `debit` column)

| Row | Source |
|---|---|
| `Accounts Receivable` | The outstanding balance of tenant invoices (`unpaid` + `partially paid`) due on or before the cut-off, converted from each lease's currency. |
| `Cash at Bank — {account} ({number})` | One row per in-scope bank account with a **positive** balance. |

### Liabilities (the `credit` column)

| Row | Source |
|---|---|
| `Accounts Payable` | Outstanding bills (`unpaid` + `partially paid`) of **every type except `liability`**, dated on or before the cut-off, **less the tax still withheld** on them. Credit-note bills carry a negative balance and so reduce this figure. |
| `Liability Bills` | Outstanding bills of type `liability` — withheld tax and VAT owed onward to the revenue authority. |
| `Bank Overdraft — {account} ({number})` | One row per in-scope bank account with a **negative** balance, shown as a positive credit. |

Zero-balance bank accounts are omitted entirely.

### Which bank accounts are in scope

Active bank accounts that are either:

- of type `system`, **or**
- of type `user` **and** belonging to the user group named by the company's
  `landlord_user_group_id` setting (the landlord group by default).

The setting is editable under `Settings > General` (module `pm`, group `finance`). When it is blank,
no `user` account is included.

### The `Difference (Suspense)` row

This report has no equity / retained-earnings side, so the two columns **will not agree**. The gap is
stated explicitly as a final `grosstotal` row rather than left implicit:

`difference = total_assets − total_liabilities`

It is placed in whichever column is short — a positive difference sits in `debit`, a negative one in
`credit` (as its absolute value). Render it as a caution row (`background_color: "warning"`).

## As-at accuracy — read this

**Bank balances are a true as-at figure.** They are summed from the cashbook
(`SUM(debit − credit)` over transactions on or before the cut-off), so back-dating `as_at` gives the
real historical cash position.

**Receivables and payables are not.** Invoice and bill balances are stored as live snapshots with no
history, so those two sides use the **current** outstanding balance of documents *dated* on or before
the cut-off. Back-dating `as_at` therefore narrows which documents count, but each one still
contributes today's balance.

`header.notes` says this in one sentence — surface it near the report title so the figures are not
misread.

## Response shape

```jsonc
{
  "data": {
    "header": {
      "as_at": "2026-06-30",
      "currency": { "code": "KES", "name": "Kenyan Shilling" },
      "filters": { "as_at": "2026-06-30", "currency_id": 1 },
      "notes": "Bank balances are as at the cut-off date. Receivables and payables are the current outstanding balances of documents dated on or before it.",
      "generated_at": "2026-08-03T09:12:44+00:00"
    },
    "report": [
      {
        "bucket": "trial_balance",
        "header": { "label": "Trial Balance", "currency": { "code": "KES", "name": "Kenyan Shilling" } },
        "fields": [
          { "label": "Account", "key": "label",  "format": "string", "alignment": "left",  "type": "normal", "weight": "font-normal", "background_color": "none", "visible": true, "togglable": false },
          { "label": "Debit",   "key": "debit",  "format": "money",  "alignment": "right", "type": "normal", "weight": "font-normal", "background_color": "none", "visible": true, "togglable": false },
          { "label": "Credit",  "key": "credit", "format": "money",  "alignment": "right", "type": "normal", "weight": "font-normal", "background_color": "none", "visible": true, "togglable": false }
        ],
        "items": [
          { "label": { "value": "Assets", "col_span": 3 }, "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Accounts Receivable" }, "debit": { "value": 412000 }, "credit": { "value": 0 }, "type": "normal" },
          { "label": { "value": "Cash at Bank — Main Collection (0102030405)" }, "debit": { "value": 1850000 }, "credit": { "value": 0 }, "type": "normal" },
          { "label": { "value": "Total Assets" }, "debit": { "value": 2262000 }, "credit": { "value": 0 }, "type": "subtotal", "background_color": "secondary" },

          { "label": { "value": "Liabilities", "col_span": 3 }, "type": "subtotal", "background_color": "primary" },
          { "label": { "value": "Accounts Payable" }, "debit": { "value": 0 }, "credit": { "value": 640000 }, "type": "normal" },
          { "label": { "value": "Liability Bills" }, "debit": { "value": 0 }, "credit": { "value": 96000 }, "type": "normal" },
          { "label": { "value": "Total Liabilities" }, "debit": { "value": 0 }, "credit": { "value": 736000 }, "type": "subtotal", "background_color": "secondary" },

          { "label": { "value": "Total" }, "debit": { "value": 2262000 }, "credit": { "value": 736000 }, "type": "grosstotal", "background_color": "secondary" },
          { "label": { "value": "Difference (Suspense)" }, "debit": { "value": 1526000 }, "credit": { "value": 0 }, "type": "grosstotal", "background_color": "warning" }
        ],
        "summary": { /* same object as data.summary below */ }
      }
    ],
    "summary": {
      "accounts_receivable": 412000,
      "accounts_payable": 640000,
      "liability_bills": 96000,
      "bank_balance": 1850000,
      "total_assets": 2262000,
      "total_liabilities": 736000,
      "difference": 1526000,
      "bank_account_count": 1
    }
  }
}
```

### `summary` keys

| Key | Meaning |
|---|---|
| `accounts_receivable` | Outstanding tenant invoice balances. |
| `accounts_payable` | Outstanding non-liability bills, net of withheld tax. |
| `liability_bills` | Outstanding `liability` bills. |
| `bank_balance` | Net cashbook position across all in-scope accounts (may be negative). |
| `total_assets` | `accounts_receivable` + positive bank balances. |
| `total_liabilities` | `accounts_payable` + `liability_bills` + overdrafts. |
| `difference` | `total_assets − total_liabilities`. |
| `bank_account_count` | In-scope accounts with a non-zero balance. |

Use these for the headline KPI tiles; the bucket's own `summary` is the same object.
