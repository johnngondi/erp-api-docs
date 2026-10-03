# Payables document emails

The three documents the company sends out when it orders or pays: the LPO to a supplier, the
payment advice to a payee, and the remittance advice to a landlord. Each is a `DocumentEmail`
definition in `app/Support/Mail/Documents` (see [Transactional emails](../emails.md#documents)):
the body comes from the same payload as the public page and the PDF, the button opens the
document's public page (`publicUrl()`), and the PDF is attached when the company has a
template for it. With no template, or a PDF that fails to render, the email goes with the link
alone.

| Document | Sent when | To | Title | Template |
|---|---|---|---|---|
| LPO | The order is issued to the supplier (`IssueLpoToVendorService`) | The supplier (`vendor`) | LOCAL PURCHASE ORDER | `PublicDocumentTemplates`, type `facility_lpo` |
| Payment voucher | The voucher is released and paid (`ReleasePaymentVoucherAction`) | The payee (`payableUser`) | PAYMENT ADVICE | `AdviceTemplateSettings`, `payment_voucher_advice_template_id` |
| Remittance | The remittance becomes due for payment (`unpaid`) | The landlord | REMITTANCE ADVICE | `AdviceTemplateSettings`, `remittance_advice_template_id` |

Each email has a greeting, a one-line introduction, an amount card (order value, amount paid,
amount remitted), the document's details (number, dates, property, supplier or landlord, and
the payment method and reference where there is one), the button to the public page, and a
closing note with the company's contact address.

## LPO

An LPO is issued in one place, `IssueLpoToVendorService::run()`, which already sends
`LpoIssuedNotification` to the supplier: in-app, SMS, and email when the supplier has an
address. The LPO email is that notification's `toMail()`, built from `LpoEmail`. Sending it
again through `SendDocumentEmailAction` would give the supplier two emails about one order, so
`LpoEmail::shouldSend()` is always false and the LPO is never stamped `emailed_at`.

The notification's other channels and its in-app payload (`toArray()`) are unchanged. The
issue service runs only for a placed order (`lpo`), so a pending or rejected order, which the
public page hides, is never emailed.

## Payment voucher

`ReleasePaymentVoucherAction` sends the payment advice inside its transaction, once the voucher
is paid; the email waits for the commit. Every path that pays a voucher goes through this
action, including settlements and voucher reconciliation. Creating a voucher sends nothing:
a pending voucher has not been paid, and its public page is hidden.

A voucher has no property of its own, so the email names the properties of the bills and
remittances it pays.

## Remittance

A remittance is emailed when it becomes `unpaid`, which happens in one of two ways:

- **No approval holds it.** `CreateRemittanceAction` sets `unpaid`, then starts the approval
  chain. When no chain forms (no template, a bypass role, or no step applies), the remittance
  is still `unpaid` afterwards and the action emails it.
- **Its approval completes.** The approval engine sets `unpaid` and fires
  `FacilityRemittanceApprovedEvent`. `SendRemittanceAdviceEmailListener` re-reads the
  remittance and emails it if it is `unpaid`.

A remittance waiting on an approver (`pending`) or `rejected` is not emailed: its public page
hides both. When a payment voucher pays the remittance (`paid`), nothing more is sent about the
remittance; the landlord gets the voucher's payment advice when the voucher is released.

## Sending once

`SendDocumentEmailAction` stamps `emailed_at` on the voucher or remittance and sends nothing for
a document already stamped, so calling it again on re-processing is safe. Inside
`OutboundEmail::suppress()` (the EPMAS importer) nothing is sent and nothing is stamped.

## Previews

Local only, with the newest suitable record in the database: `/_mail/lpo`,
`/_mail/payment-voucher`, `/_mail/remittance`. Each shows nothing when there is no such record.
