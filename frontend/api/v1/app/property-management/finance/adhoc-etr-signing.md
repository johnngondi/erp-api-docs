# Adhoc ETR Signing

Signs an invoice or credit note on an ETR (KRA fiscal) device from typed-in figures, for a document
that does not exist in the ERP. The request goes straight to the device and nothing is saved, so
the CU number in the response is the only copy the ERP hands back. Show it to the user.

Opened from the **Create** (quick actions) menu. Show the entry only when the user has the
`sign-facility-invoice` permission, which `me/permissions` returns.

## Sign

`POST /api/v1/app/{company}/property-management/etr/adhoc-sign`

### Request

```json
{
  "device_type": "incotex", // incotex | tevin | novitas | etims
  "seller_pin": "P051123456A",
  "customer_pin": "A004512367K",
  "invoice_number": "CN-000318",
  "invoice_date": "2026-11-11",
  "amount": 85000,
  "tax_id": 2,
  "tax_amount": 13600,
  "total": 98600,
  "is_credit_note": true,
  "reference_number": "0040712340000218",
  "notes": "Credit for overbilled November rent."
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `device_type` | string | Required. `incotex`, `tevin`, `novitas` or `etims`. `etims` is not live yet, and signing on it fails with a clear error. |
| `seller_pin` | string | Required, max 30. Picks the device, see below. |
| `customer_pin` | string | Required, max 30. |
| `invoice_number` | string | Required, max 50. The number the device records against the signature. |
| `invoice_date` | date | Required, `YYYY-MM-DD`, today or earlier in the company's timezone. |
| `amount` | number | Required, above 0. Net of tax. |
| `tax_id` | integer | Required, a row from `settings/.../taxes`. Supplies the rate sent to the device. |
| `tax_amount` | number | Required, 0 or more. The frontend fills in `amount x tax rate` and the user may change it. |
| `total` | number | Required, above 0. The frontend fills in `amount + tax_amount` and the user may change it. |
| `is_credit_note` | boolean | Optional, default `false`. |
| `reference_number` | string | Required when `is_credit_note` is `true`, max 50. The CU number of the invoice being credited, usually about 16 characters. Ignored on an invoice. |
| `notes` | string\|null | Optional, max 255. Becomes the description of the document's single line, with line breaks collapsed to spaces. |

Money is sent to the device at two decimals. Credit note figures are positive.

### How the device is chosen

1. `seller_pin` equal to the company's KRA PIN uses the company device: the `device_type` driver's
   system default connection.
2. Otherwise `seller_pin` must be the KRA PIN of a landlord of a property in this company. The
   device is that landlord's property set to `device_type`. A property with no device of its own
   counts when `device_type` is the system default device.

### Response `200`

```json
{
  "data": {
    "message": "Credit note signed successfully",
    "document_type": "credit_note",
    "invoice_number": "CN-000318",
    "cu_invoice_number": "0040712340000219",
    "cu_serial_number": "KRAMW004071234",
    "cu_verify_url": "https://itax.kra.go.ke/KCSE/verify?x=1"
  }
}
```

`cu_serial_number` and `cu_verify_url` are left out when the device does not return them.

### Errors

| Status | When |
| --- | --- |
| `403` | The user lacks `sign-facility-invoice`. |
| `422` `errors.seller_pin` | The PIN is neither the company's nor a landlord's in this company. |
| `422` `errors.device_type` | The landlord has no property on that device type. |
| `422` `errors.etr` | The device refused the document, could not be reached or stayed busy, or it is not configured. The message says which. |
| `422` other fields | Validation, as in the table above. |

The call waits for the device, up to about 45 seconds when the device is busy. Use a long client
timeout, and do not resend automatically: a device that signed but whose answer was lost has
already recorded the document.

The current-month rule on facility invoices does not apply here.
