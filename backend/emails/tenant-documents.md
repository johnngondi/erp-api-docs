# Tenant document emails

The emails a tenant gets about their account: the invoice, the credit note and the receipt
when each is issued, and payment reminders for overdue invoices. All of them are built on the
shared shell and blocks described in [emails.md](../emails.md).

## Invoice, credit note, receipt

Each is a `DocumentEmail` definition in `App\Support\Mail\Documents`, sent by
`SendDocumentEmailAction` through `DocumentIssuedNotification` (queued, email only).

| Document | Definition | Recipient | Sent when |
|---|---|---|---|
| Invoice | `InvoiceEmail` | the lease tenant | `ProcessInvoiceAction` finalises it (status `unpaid`, or `paid` / `partially paid` when an overpayment or open credit settles it at once). An invoice that carries tax waits for its ETR signature: it is emailed from `SignEtrDocumentAction` the moment it is signed |
| Credit note | `CreditNoteEmail` | the lease tenant | `ProcessCreditNoteAction` processes it for the first time |
| Receipt | `ReceiptEmail` | the payer (`paying_user_id`) | it is first confirmed: `ReceiptService::createReceipt()` when no approval applies, `ProcessReceiptAction` on the approval and bypass paths |

- **Once.** `emailed_at` is stamped when the email is queued, and a stamped document is never
  emailed again: re-processing an invoice, a receipt re-run with new allocations, or an
  approval listener firing twice sends nothing.
- **After commit.** Each hook registers the send with `DB::afterCommit()`, so a rolled-back
  request sends and stamps nothing. It is also why the hook can see links that a caller writes
  after processing the document (the opening balance row, the reversal link).
- **A taxed invoice goes signed.** An invoice with tax is not emailed until it has a CU number.
  Signed is enough: a failed KRA verification of the number (`cu_validation_status = failed`)
  does not hold it back. An invoice signed while still pending is emailed when it is processed.
  An invoice with no tax is emailed at processing, signed or not.
- **Not while suppressed.** Nothing is sent inside `OutboundEmail::suppress()` (the EPMAS
  importer).
- **No address, no email.** A tenant or payer with no email address gets nothing, and the
  document is not stamped.

### Content

Built from `DocumentRenderer::documentPayload()`, the data the PDF and the public page use, so
the figures match the public page to the cent.

- **Invoice:** greeting; "your invoice for {property} is ready"; amount due (the balance) with
  the due date; invoice number, date, property, unit, landlord, lease, description, VAT and
  total (paid and balance due when something was settled at issue); the button "View and pay
  invoice" to the public invoice page; the bank and M-Pesa paybill details from the property's
  collection accounts; a closing line with the company's email and phone. An invoice settled
  in full at issue says "View invoice" and leaves the payment details out.
- **Credit note:** the amount credited, what it is applied against (or that it stays on the
  account), the reason and totals, and "View credit note".
- **Receipt:** the amount received with the date, method and reference, what it was applied
  to, anything held on account, and "View receipt".

The PDF is attached when the company has a usable template, chosen by
`PublicDocumentTemplates::templateFor()`: the company's default template for the document type
and property, else its newest active one. A new invoice uses the default invoice template; the
"Invoice Reminder Template" setting is for payment reminders only. With no template, or a render that fails, the email
goes with the link alone.

### Exclusions

| Not emailed | How it is recognised |
|---|---|
| Pending, rejected or cancelled documents | status; the public page hides pending and rejected, and a cancelled document is not issued. `SendDocumentEmailAction` also refuses any `pending` document outright, whatever its definition says |
| Anything on a lease that is still pending | its deposits and opening balances raise no document until the lease is approved or activated; the deposit invoice is emailed then |
| An invoice or credit note for a lease opening balance | a `lease_opening_balances` row (trashed included) names it in `invoice_id` / `credit_note_id`. This covers the documents raised by creating, updating and deleting an opening balance |
| An invoice re-issued for a cancelled ETR-signed credit note | `facility_invoices.reissued_from_credit_note_id` is set |
| A credit note that reverses an ETR-signed invoice | an invoice names it in `reversal_credit_note_id` |
| The negative mirror receipt that cancels a remitted receipt | its amount is negative (it is also written directly, never through the hooks) |

### Sending invoices that were never sent

`invoices:send-issued` emails ETR-signed invoices that have not been sent yet (`emailed_at`
empty), as the "new invoice" email with the default template, not a reminder.

```bash
php artisan invoices:send-issued --from=2026-10-01 --dry-run   # list, send nothing
php artisan invoices:send-issued --from=2026-10-01 --company=1
```

| Option | Meaning |
|---|---|
| `--from=` | Required. Earliest issue date (the backdated `transaction_at`, else the day it was raised). Imported EPMAS invoices carry CU numbers too, so there is no default |
| `--to=` | Latest issue date |
| `--company=` | Only this company |
| `--limit=` | Send at most this many |
| `--without-etr` | Also send taxed invoices that are not ETR signed yet, for when the ETR device is down. They go without a CU number, and signing them later does not send them again |
| `--dry-run` | Print, per company, what would be sent |

It applies the same checks as the automatic email (status, opening balance, re-issue, a tenant
with an address) and stamps `emailed_at`, so running it twice sends nothing new.

Everything else that reaches the hooks is a real charge or credit to the tenant and is
emailed: the monthly invoice run and "generate next period", manual invoices and credit
notes, deposit invoices, utility billing, the bulk charge import, the receipt cancellation fee
invoice, and the credit notes that settle arrears from the deposit on an exit notice.

## Payment reminders

`php artisan invoices:send-reminders {--company=} {--dry-run}`, scheduled daily at 09:30
(`withoutOverlapping`).

Off until a company turns it on: the setting `invoice_reminders_enabled` (Settings → Finance,
"Send Invoice Payment Reminders", `yes` / `no`, default `no`), read with
`InvoiceReminderSettings::enabledFor($companyId)`.

### Which invoices

In each company with reminders on, an invoice that is

- `unpaid` or `partially paid`,
- with a `balance` above zero,
- past its `due_at`,
- not an opening balance (no `lease_opening_balances` row names it),
- and, when it carries tax, ETR signed (it has a CU number).

`emailed_at` records when the invoice was first sent; `last_reminded_at` when the latest reminder
went. Both are returned on the invoice resource.

### Cadence

`config('emails.invoice_reminder_days')` (`[1, 7, 14, 30]`) are the days past due a reminder
goes on; after the last of them, one goes every `invoice_reminder_repeat_days` (`30`): days
60, 90, 120 and so on.

Each run works out the latest step the invoice has reached and sends only when the last
reminder (`facility_invoices.last_reminded_at`) was before that step. So:

- a second run on the same day sends nothing;
- an invoice that is 40 days overdue and was never reminded gets one reminder (the 30-day
  step), not four, and the next at 60 days.

The command stamps `last_reminded_at` and sends `InvoiceReminderNotification` through
`OutboundEmail` in one transaction per invoice. A tenant with no email address is skipped
and not stamped. `--dry-run` prints, per company, the invoices it would remind (tenant, days
overdue, step, balance, last reminder) without sending or stamping.

### The reminder

`App\Notifications\PropertyManagement\InvoiceReminderNotification` (queued, email only),
titled PAYMENT REMINDER:

- the invoice, its due date and how many days overdue it is, what has been paid against it,
  and "if you have already paid, share the transaction reference";
- the overdue amount on an accent-coloured card;
- **ageing of the account:** the tenant's outstanding balance on that lease (`total - paid` of
  every invoice that is not pending, cancelled or rejected, opening balances included) in
  Current, 1–30, 31–60, 61–90 and Over 90 days past due, as at the day the email is built, and
  the total. An invoice with no due date ages from when it was raised;
- "Pay {amount} now" to the public invoice page, then the bank and M-Pesa details;
- a closing line with the company's contact.

It reuses `InvoiceEmail`, so the invoice PDF printed with `InvoiceReminderSettings::templateFor()`
is attached, or the link goes alone.

## Previews

Locally, `GET /_mail/invoice`, `/_mail/credit-note`, `/_mail/receipt` and
`/_mail/invoice-reminder` render the newest suitable record in the database. Each says there
is no sample data when there is none.
