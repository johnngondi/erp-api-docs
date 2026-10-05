# Transactional emails

The platform sends a fixed list of emails. Each one is the same branded shell with a list of
blocks in its body, built in PHP and sent through the notification system, so the per-company
mailer and the delivery log (see [messaging.md](messaging.md)) apply to every one of them.

Everything else a notification says is in-app (and SMS) only.

## The shell

Every email has, top to bottom:

- a hidden preheader (the line inboxes show after the subject);
- a header in the company's brand colour: its logo on the left, the title (`INVOICE`,
  `WELCOME`) on the right;
- a 4px rule in the accent colour;
- the body: the blocks, in the order they were added;
- the footer: a services band in the accent colour (left out when the company lists no
  services), a contact line (name, postal address, phone, email, website), a note that the
  message is automated, and "Powered by EPMAS".

All of it comes from the company record: `name`, `brand_color`, `text_color_on_brand_bg`,
`accent_color`, `text_color_on_accent_bg` (through `CompanyPalette`, so the env defaults
apply), `services`, `company_phone`, `company_email`, `company_website`, `postal_address`,
`profile_photo_path` and `light_logo_path`. With no company (a personal notification) the
email carries `config('app.name')` and the palette defaults.

The header logo:

1. `light_logo_path`, when set: a logo made to sit on the brand colour.
2. Otherwise `profile_photo_path` on a white rounded chip.
3. Otherwise, or when the only logo is not a PNG, JPG or GIF (an SVG, which mail clients do
   not render), the company name as text.

The markup reproduces `resources/views/mail/_design/reference.html`: tables, inline CSS, a
600px wrapper, the `@media (max-width:620px)` rules and the Outlook conditional. Mail clients
are not browsers, so keep new markup in that style.

## Building an email

`App\Support\Mail\EmailMessage` is a fluent builder that returns Laravel's `MailMessage`:

```php
public function toMail(object $notifiable): MailMessage
{
    return EmailMessage::for($company)
        ->title(__('INVOICE'))
        ->preheader(__('Invoice :number for :property', [...]))
        ->text([__('Dear :name,', [...]), __('Your invoice is ready.')])
        ->amountCard(__('Amount due'), 'KES 46,915.80', __('Due by :date', [...]))
        ->keyValues([__('Invoice number') => 'INV0001', __('Total') => 'KES 46,915.80'], emphasiseLast: true)
        ->button(__('View and pay invoice'), $invoice->publicUrl(), __('Pay by bank transfer or M-Pesa.'))
        ->paymentMethods($bank, $mobileMoney)
        ->note(__('Questions? Write to :email.', [...]))
        ->toMailMessage(__('Invoice :number', [...]));
}
```

`toMailMessage($subject)` sets the subject and the views `mail.layout` (HTML) and `mail.text`
(plain text), and attaches anything added with `attach($data, $name, $mime)`. The title
defaults to the subject, the preheader to the subject.

Values are plain strings and Blade escapes them. Where a line needs inline markup, pass an
`Illuminate\Support\HtmlString` and escape what goes into it yourself:
`new HtmlString(e(__('Quote')).' <strong>'.e($number).'</strong>')`.

### Blocks

One Blade partial each, in `resources/views/mail/blocks/`. Each block is stored as
`['type' => ..., ...]` in `$mail->viewData['email']['blocks']`, which is what tests assert on.

| Method | Block | Shows |
|---|---|---|
| `text(string\|array $paragraphs)` | `text` | Greeting and paragraphs |
| `headline($headline, $eyebrow = null)` | `headline` | Accent-coloured eyebrow over a large headline |
| `amountCard($label, $amount, $sub = null, $accent = 'brand')` | `amount-card` | The amount, on a tinted card with a brand edge (`'accent'` for overdue) |
| `keyValues([label => value], $emphasiseLast = false)` | `key-values` | Label/value rows; the last one bold when it is a total. Null values drop their row |
| `card(?$label, [label => value])` | `card` | The same rows on a tinted card, optionally labelled (credentials, notice details) |
| `table($headers, $rows, $caption = null)` | `table` | Column headers and rows; a header may be `['label' => ..., 'align' => 'right']` |
| `button($label, $url, $note = null)` | `button` | The call to action, with a side note |
| `paymentMethods($bank, $mobileMoney)` | `payment-methods` | Bank and M-Pesa columns; each argument one method or a list of them (keys below) |
| `ageing($buckets, $total, $asAt = null)` | `ageing` | A bar split by bucket, each bucket's balance, the total outstanding |
| `code($code, $validUntil = null, $label = null)` | `code` | A one-time code, large and spaced, and when it expires |
| `callout($lead, $text)` | `callout` | Tinted box with a bold lead ("Didn't request this?") |
| `steps(?$title, $items)` | `steps` | A numbered list; an item is a string or `['lead' => ..., 'text' => ...]` |
| `signature($name, $role = null, $contact = null)` | `signature` | Who it is from |
| `note($text)` | `note` | A muted closing line |

`paymentMethods()` keys: bank `title` (default "Bank transfer"), `bank`, `branch`,
`account_name`, `account_number`; mobile money `title` (default "M-Pesa paybill"),
`business_number`, `account_number`, `note` (default "Use the account number exactly as
shown."; `null` hides it). An empty array leaves that kind out.

`ageing()` buckets: `['label' => '31–60 days', 'amount' => 'KES 46,915.80', 'value' => 46915.8]`.
`value` sizes the bucket's share of the bar and marks its row as owing.

### View data

All view data travels under one key, `email`. The mailer reserves `$message`, and
`MailMessage::data()` merges `subject`, `greeting`, `actionText`, `actionUrl`, `introLines`,
`outroLines` and `level` into the view, so a view key with one of those names would be
overwritten. The plain-text view renders with `{!! !!}` through
`App\Support\Mail\EmailPlainText`.

### Simple notifications

`RendersNotificationLayout::layoutMail($subject, $greeting, $lines, $actionUrl, $actionText, $footerLines)`
builds the same email for a notification that is a greeting, a few lines and a button: the
subject is the title, the lines a text block, the action a button, each footer line a note.
It brands the email with the notification's company when it implements
`HasNotificationCompany`.

## Which notifications email

Email is on by exception.

- `DeliversOnDefaultChannels::defaultChannels()` returns `database` and, with a phone number,
  SMS. Never `mail`.
- A notification emails only when it implements `App\Contracts\Notifications\SendsEmail`.
  It adds mail with `withEmail($this->defaultChannels($notifiable), $notifiable)`, which
  appends `mail` when the recipient has an address (an `AnonymousNotifiable` with a mail route
  gets `['mail']`), or lists `mail` in its own `via()`.
- `tests/Feature/Emails/EmailAllowListTest.php` walks every class under `app/Notifications` and
  fails when one without the marker can return `mail`.

Marked today: `TemporaryPinNotification`, `LooSentNotification`, `TestMessagingNotification`,
`LpoIssuedNotification`, `QuoteSubmissionReminderNotification`,
`BillInvoiceRejectedNotification`, `DocumentIssuedNotification`, and the new email types
below.

## Sending

Send every email through `App\Support\Mail\OutboundEmail::send($notifiable, $notification)`:

- **After commit.** The send waits in `DB::afterCommit()`: at once when no transaction is
  open, after the commit when one is, never when it rolls back. A notification's own
  `->afterCommit()` does nothing on the `sync` queue that `.env` and `phpunit.xml` use, so a
  welcome email with a password would otherwise leave for an account that was never saved.
- **Off switch.** Nothing is sent inside `OutboundEmail::suppress(fn () => ...)`. It nests, and
  is read when `send()` is called. `EpmasImportCommand` runs its whole import inside it: the
  importer creates invoices and credit notes through the actions that email them.
  `LandlordController::store` creates the new landlord's properties inside it, so the landlord
  gets one welcome email with the portfolio rather than one email per property.
- **Secrets.** Emails that carry a password or a reset code implement
  `Illuminate\Contracts\Queue\ShouldBeEncrypted` when queued, and are not written to the
  in-app list. The delivery log stores no body.

## Documents

An issued document is emailed by `App\Actions\Documents\SendDocumentEmailAction::execute($document)`,
called wherever the document becomes final. It looks the model up in
`config('emails.documents')` and sends `DocumentIssuedNotification` (queued, email only) once:

- nothing for a model with no definition, a document still `pending` (whatever its
  definition says), a definition whose `shouldSend()` is false, a recipient with no address, or
  a document whose `emailed_at` is set;
- otherwise it stamps `emailed_at` (quietly, in the caller's transaction) and sends through
  `OutboundEmail`. Inside `suppress()` it neither sends nor stamps.

A definition extends `App\Support\Mail\Documents\DocumentEmail`:

| Method | Returns |
|---|---|
| `documentType()` | The key in `config/document_templates.php`, e.g. `facility_invoice` |
| `recipient()` | The notifiable (usually a `User`), or null |
| `company()` | Whose branding and mailer |
| `title()`, `subject()` | Header title and subject |
| `template()` | The PDF template, from `InvoiceReminderSettings`, `PublicDocumentTemplates` or `AdviceTemplateSettings`, never picked by hand |
| `filename()` | The attachment name, e.g. `INV0042.pdf` |
| `compose($email, $payload)` | The body; the button links `$document->publicUrl()` |
| `shouldSend()` | Default true; false for re-issues, reversals, opening balances |

The payload is `DocumentRenderer::documentPayload()` after eager-loading the type's `with`
list: the same data as the PDF and the public page. The PDF from `DocumentRenderer::pdf()` is
attached inside a try/catch; with no template, or a render that fails (logged with
`log_exception`), the email goes with the link alone.

Registered definitions: `InvoiceEmail`, `CreditNoteEmail`, `ReceiptEmail`, `LpoEmail`,
`PaymentVoucherEmail`, `RemittanceEmail`.

**A document goes out only after it is processed, never while it is pending.** The hooks that
call `SendDocumentEmailAction` sit in the processing step (`ProcessInvoiceAction`,
`ProcessCreditNoteAction`, `ProcessReceiptAction`, the remittance and voucher release paths),
never in creation, and the action itself refuses any document whose status is `pending`. The LPO,
which is issued outside this action, follows the same rule: `IssueLpoToVendorService` sends only
for a placed order (`lpo`). A lease's deposit and opening balance documents are not even raised
while the lease is pending (see the lease deposits and opening balances API docs).

## Adding an email type

1. A notification implementing `SendsEmail` (and `HasNotificationCompany` for branding and the
   company mailer), with `toMail()` built from `EmailMessage`. Queue it (`ShouldQueue` and the
   `QueuesOutboundChannels` trait, see [Queues](messaging.md#queues)); add `ShouldBeEncrypted`
   when it carries a secret.
2. Send it with `OutboundEmail::send()` from the action where the event happens, never from a
   model observer, and stamp a column if it must go once.
3. Links to the frontend come from `FrontendUrl` (below); links to a document from its
   `publicUrl()`.
4. A preview class in `App\Support\Mail\Previews` registered in `config('emails.previews')`.
5. A test in `tests/Feature/Emails/` that renders it and asserts content and link.

For a document, write a `DocumentEmail` definition and register it in
`config('emails.documents')` instead of a notification.

## Frontend links

`config/frontend.php` holds the Vue frontend's base URL (`FRONTEND_URL`) and its paths.

```php
FrontendUrl::to('tenant.lease', ['id' => 7]);   // https://app.example.com/tenant/lease-management/leases/7
FrontendUrl::to('vendor.bid', ['job' => 12]);   // .../vendor/procurement/rfq-quotes/create?job=12
FrontendUrl::to('tenant.invoice', ['id' => 3, 'tab' => 'payments']); // unused params become the query
FrontendUrl::login('portals.vendor');           // .../auth/login?redirect=%2Fvendor
```

Keys are dot paths under `frontend.links`. An unknown key, or a placeholder no parameter fills,
throws. `login()`'s `redirect` is the target's path, not an absolute URL. The vendor portal is
`/vendor`, never `/supplier`: that alias fails the post-login redirect. An empty
`FRONTEND_URL` falls back to `APP_URL`.

## Registries

`config/emails.php`:

- `welcome_sections`: the per-portal part of the welcome email, by portal (`app`, `tenant`,
  `vendor`, `landlord`), each an `App\Support\Mail\Welcome\WelcomeSection`.
- `documents`: document model to definition.
- `invoice_reminder_days` (`[1, 7, 14, 30]`) and `invoice_reminder_repeat_days` (`30`): the
  reminder cadence, while the company setting `invoice_reminders_enabled` is `yes`
  (`InvoiceReminderSettings::enabledFor($companyId)`; default `no`).
- `previews`: key to `App\Support\Mail\Previews\EmailPreview`.

## Seeing it

- `php artisan messaging:test --email=you@example.com --company=1` sends a showcase that uses
  every block, in the company's branding. `--message` sets its opening paragraph; SMS and
  WhatsApp are unchanged. Use `MAIL_MAILER=log` or Mailtrap.
- Locally (`APP_ENV=local`), `GET /_mail` lists the email types and `GET /_mail/{type}` renders
  one with sample data, without sending. A type with no sample data yet says so.

## Deployment

- Set `FRONTEND_URL` to the Vue frontend's URL.
- Run the migrations: `companies.light_logo_path`, the stamps (`users.welcome_emailed_at`,
  `leases.tenant_notified_at`, `emailed_at` on the six document tables,
  `facility_invoices.last_reminded_at`), `user_password_reset_codes`, and the
  `invoice_reminders_enabled` setting backfill.
- Run a real queue worker (`QUEUE_CONNECTION=database`) that serves the `otp` and
  `notifications` queues: `php artisan queue:work --queue=otp,notifications,default`. On `sync`,
  a document email renders its PDF inside the request, which can take seconds.

## Email types

- [Welcome and the migrated-users command](emails/welcome.md)
- [Supplier: prequalified categories, bill rejected](emails/supplier.md)
- [Lease](emails/lease.md)
- [Landlord properties](emails/landlord.md)
- [Codes: PIN reset and forgot password](emails/codes.md)
- [Tenant documents and invoice reminders](emails/tenant-documents.md)
- [Payables documents: LPO, payment advice, remittance advice](emails/payables-documents.md)
- [Procurement and offers: RFQ invitation, quote reminder, Letter of Offer](emails/procurement-and-offers.md)
