# Settlements API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/settlements`

## Endpoints

- `GET /settlements`
- `POST /settlements`
- `GET /settlements/{settlement}`
- `PUT/PATCH /settlements/{settlement}`
- `DELETE /settlements/{settlement}`
- `PATCH /settlements/{settlement}/cancel`
- `PUT /settlements/{settlement}/settle`
- `GET /settlements/{settlement}/export?format=excel|pdf`

## List Settlements

`GET /api/v1/app/{company}/property-management/finance/settlements`

Supported query params:

- Filters:
  - `filter[search]` (Scout-backed search; supports CSV IDs and payment advice numbers)
  - `filter[debit_bank_account_id]`, `filter[payment_method_id]`, `filter[status]`, `filter[created_at]`

**Ids, `status` and `type` match exactly.** They were partial until 2026-09-29, so
`filter[status]=paid` also returned `unpaid` and `partially-paid`, and `filter[facility_id]=6`
returned rows for facility 60. Send the whole value. Date filters are still partial, so
`filter[created_at]=2026-09` means that month.

- Sort:
  - `sort=id,created_at,updated_at`
- Include:
  - `include=paymentVouchers`
- Pagination:
  - `per_page`, `page`

`GET /settlements/{settlement}` and the create/update responses load each voucher's payee (`payment_vouchers[].payable_user`). The list only returns `payment_vouchers` with `include=paymentVouchers`, and without payees.

Enum filter options:

- `filter[status]`: `pending`, `settled`, `cancelled` (from `SettlementStatus` enum)

Sample list response (`FacilitySettlementResource`):

```json
{
  "data": [
    {
      "id": 120,
      "debit_bank_account": { "id": 13, "account_name": "System Disbursement Account" },
      "payment_method": { "id": 4, "name": "ETF", "code": "ETF" },
      "payment_advice_number": "SET-ADV-120",
      "payment_advice_upload": { "id": 45, "name": "set-adv-120.pdf" },
      "status": { "value": "pending", "color": "info" },
      "payment_vouchers": [
        { "id": 801, "amount": "2500.00", "payable_user": { "id": 57, "name": "Mwangi Security Services Ltd" } }
      ],
      "created_at": {
        "raw": "2026-05-09T10:20:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      },
      "updated_at": {
        "raw": "2026-05-09T10:20:00.000000Z",
        "formatted": "09 May, 2026",
        "diff": "moments ago"
      }
    }
  ]
}
```

## Create payload (`SettlementData`)

| Field | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `debit_bank_account_id` | Yes | integer | Must be an active `system` bank account |
| `payment_method_id` | Yes | integer | An active `payment_methods.id`. It picks the bank payment file layout (EFT or RTGS) |
| `payment_vouchers` | Yes | array[int] | Vouchers must be `released` and unused in pending/settled settlement |
| `payment_advice_number` | No | string | Optional |
| `payment_advice_upload_id` | No | integer | Must exist in `uploads.id` |

## Settle payload

| Field | Required | Type |
|---|---|---|
| `payment_advice_number` | Yes | string |
| `payment_advice_upload_id` | Yes | integer (`uploads.id`) |

## Export

`GET /api/v1/app/{company}/property-management/finance/settlements/{settlement}/export?format=excel`

Query params:

| Param | Required | Type | Allowed Values / Notes |
|---|---|---|---|
| `format` | Yes | string | `excel`, `pdf` |
| `document_template_id` | No | integer | PDF only. An active `facility_settlement` document template. When omitted, the company's default settlement template is used, else its newest active one |

Permission: the same as viewing the settlement (`view-facility-settlement`).

### Excel: bank payment file

`format=excel` downloads the file you upload to the bank to make the payments. It holds only the bank's layout: one row per payment voucher in the settlement, plus any title and total rows that the bank's layout has. There is no other styling.

The layout is chosen in two steps:

1. The template comes from the bank of the settlement's debit bank account (`settlement_template`, see Common Settings, Banks). A bank without one uses the `default` template.
2. Within that template, the layout comes from the settlement's payment method code: `ETF`/`EFT` uses `eft`, `RTGS` uses `rtgs`.

Templates are defined in `config/settlement_exports.php`, with each column's heading and the path it reads from the settlement or voucher.

| Template | Layouts |
|---|---|
| `default` | `eft`, `rtgs` |
| `scbk` (Standard Chartered) | `eft`, `rtgs` |
| `kcb` (KCB) | `eft`, `rtgs` |

Response: `200` with `Content-Disposition: attachment; filename="settlement-{id}-{template}-{layout}.xlsx"`.

Errors (`422`):

- `payment_method_id`: the settlement has no payment method, or the template has no layout for its payment method (for example Cheque).
- `settlement`: the settlement is cancelled, so it has no vouchers left to pay.
- `settlement`: the debit bank's `settlement_template` names a template that is not in the config.

### PDF

`format=pdf` prints the settlement through a `facility_settlement` document template (Settings > Documents). The seeded "Settlement — Standard" template shows the settlement details, one row per voucher (voucher no., payee, bank, account no., reference, amount) with a total, and approval lines.

Response: `200` with `Content-Type: application/pdf` and `Content-Disposition: attachment; filename="settlement-{id}.pdf"`.

Errors (`422`):

- `document_template_id`: the given template is not an active settlement template, or the company has none.

Frontend: request either format with `responseType: 'blob'` and save the file using the name in `Content-Disposition`.
