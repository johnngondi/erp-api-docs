# Messaging: email, SMS and WhatsApp drivers

Every outbound email, SMS and WhatsApp message goes through a driver. The code that sends
never names a provider. Delivery picks the credentials at send time, in this order:

1. **Company:** the company's active `company_messaging_channels` row for the channel,
   when every required credential is filled.
2. **Platform:** the default driver in `config/messaging.php` (email: `MAIL_MAILER`),
   using the credentials in `.env`.
3. **Fallback:** the `log` driver. It writes the message to the log and reports it as
   sent, so a missing configuration never throws from inside a queued job.

The company and platform credentials are never merged. A company's WhatsApp phone number
id with the platform's access token would send from an account that does not own the
number, and a company's `from_address` is only verified on its own Resend account.
A half-filled company row counts as not configured.

`APP_NOTIFY=false` routes every SMS and WhatsApp message to the log driver, as the old
`SMSSdk` did. Email follows `MAIL_MAILER` as before.

## Sending

Notifications need nothing new. Delivery reads the company from
`HasNotificationCompany::notificationCompanyId()`. A notification without one, or with a
null company, sends through the platform account.

| Channel | `via()` entry | Message method | Recipient |
| --- | --- | --- | --- |
| Email | `'mail'` | `toMail()` | `routeNotificationForMail()` |
| SMS | `App\Channels\SMSChannel::class` | `toSms()` | `routeNotificationForSms()`, else `$notifiable->phone` |
| WhatsApp | `App\Channels\WhatsAppChannel::class` | `toWhatsApp()` | `routeNotificationForWhatsapp()`, else `$notifiable->phone` |

- `'mail'` is overridden by `App\Channels\MailChannel` in `AppServiceProvider`, the same
  way `'database'` is. It sets the company's mailer on the message unless the notification
  picked a mailer itself.
- `toSms()` and `toWhatsApp()` may return a string, or an
  `App\Notifications\Messages\SmsMessage` / `WhatsAppMessage` when the message has to name
  its company (`->from($company)`) or recipient (`->to($phone)`) explicitly.
- Phone numbers are normalised with `format_country_phone_number()` and sent in E.164.

## Which notifications text

SMS and WhatsApp are on by exception, like email (see [emails.md](emails.md#which-notifications-email)).
`App\Notifications\Concerns\DeliversOnDefaultChannels::defaultChannels()` returns `database` only
(`[]` for an `AnonymousNotifiable`). A notification adds a text channel itself:

- `withSms($channels, $notifiable)` appends `SMSChannel` when the recipient has a number
  (`routeNotificationFor('sms')`, else `$notifiable->phone`);
- `withWhatsapp($channels, $notifiable)` does the same for `WhatsAppChannel`. Meta delivers
  free-form text only within 24 hours of the recipient writing to the business, so add it
  alongside SMS, never instead of it.

Text is allowed only for one-time codes, a document sent to a tenant or a supplier, an RFQ sent to a
supplier, and reminders to a supplier. Today that is:

| Purpose | Notification (under `App\Notifications`) | Channels |
| --- | --- | --- |
| One-time codes | `Auth\TemporaryPinNotification`, `Auth\PasswordResetCodeNotification` (one notification per channel, from `SessionPinService` / `PasswordResetCodeService`) | the channel asked for |
| A document sent to a supplier | `PropertyManagement\Procurement\LpoIssuedNotification` | database, mail, SMS |
| A reminder to a supplier | `PropertyManagement\Procurement\QuoteSubmissionReminderNotification` | database, mail, SMS, WhatsApp |
| Provider check | `Messaging\TestMessagingNotification` (`messaging:test`) | the channel asked for |

Everything else (approvals, changes requested, workflow steps, document changes, comments,
responsibility transfers, rejections, staff and sensor alerts) is in-app only, plus email where it
is on the email list. `tests/Feature/Notifications/SmsAllowListTest.php` holds this list and fails
when any other notification defines `toSms()` / `toWhatsApp()` or returns a text channel. Adding
to it is a product decision.

Outside notifications:

```php
app(SmsManager::class)->for($company)->send($phone, $text);       // MessageResult
app(WhatsAppManager::class)->for($company)->send($phone, $text);  // MessageResult
Mail::mailer(app(EmailManager::class)->mailerFor($company))->send(...);
```

`for(null)` means the platform account.

## Queues

Outbound messages run on named queues, so a backlog of emails cannot hold up a one-time code:

| Queue | What runs on it |
| --- | --- |
| `otp` | `TemporaryPinNotification` and `PasswordResetCodeNotification`, on every channel |
| `notifications` | The email, SMS and WhatsApp sends of every other notification |
| `default` | The in-app (`database`) copy of a notification, and every other job |

- A notification that sends email, SMS or WhatsApp implements `ShouldQueue` and uses
  `App\Notifications\Concerns\QueuesOutboundChannels`. Its `viaQueues()` puts `'mail'`,
  `SMSChannel` and `WhatsAppChannel` on `notifications`. The `database` channel stays on `default`.
- The two OTP notifications call `onQueue(NotificationQueue::OTP)` in their constructor, and
  implement `ShouldBeEncrypted` so the code cannot be read from the `jobs` table.
- The names are constants on `App\Support\Messaging\NotificationQueue`.
- `TestMessagingNotification` is the exception. `messaging:test` sends it with `notifyNow()` so
  it can print each provider's answer.
- `tests/Feature/Messaging/NotificationQueuesTest.php` fails when a notification that can send
  email, SMS or WhatsApp is not queued this way.

A worker only serves the queues it is told about. One started without `--queue` serves `default`
alone, and no email, SMS or WhatsApp would leave. List them in priority order:

```bash
php artisan queue:work --queue=otp,notifications,default
```

or run a separate worker for `otp` so a code never waits behind anything else.

## Layout

`app/Support/Messaging`:

- `ChannelManager`: resolves drivers from `config("messaging.{channel}.drivers")` and applies
  the precedence above. `EmailManager`, `SmsManager` and `WhatsAppManager` extend it. It is
  deliberately not `Illuminate\Support\Manager`, which caches one instance per driver name
  for the life of the process: a queue worker serves many companies, and a cached instance
  would carry one company's credentials into the next send.
- `Drivers/Driver`: holds a credential bag and checks it against the driver's `fields()`.
  `withCredentials()` returns a clone, so a shared instance is never mutated.
- `Drivers/Email/{Resend,Smtp}Driver`: turn credentials into a Laravel mailer definition.
  `EmailManager::mailerFor()` registers it as `company-{id}` and purges the cached mailer
  every time, so a rotated key is picked up without restarting workers.
- `Drivers/Sms/AfricasTalkingDriver`, `Drivers/WhatsApp/MetaCloudDriver`: send and return a
  `MessageResult`. A provider rejection is a failed result, never an exception.
- `Mail/ResendTransport`: Laravel 10 has no Resend transport, and the `resend/resend-laravel`
  package binds one global API key, which rules out per-company keys. This transport takes
  its key from the mailer config and is registered as the `resend` mail transport. On
  Laravel 11 the framework's own transport can replace it.

### Adding a provider

1. A class extending `Drivers\Driver` that implements `SmsDriver`, `WhatsAppDriver` or
   `EmailDriver`, declaring its credential `fields()`.
2. An entry under `messaging.{channel}.drivers` with `via`, `label` and `platform`
   credentials.
3. Its env variables in `.env.example`.

The driver catalog endpoint and validation both read `fields()`, so the API and frontend
pick it up without other changes.

## Storage

`company_messaging_channels`: one row per company and channel (unique).

| Column | Notes |
| --- | --- |
| `channel` | `email`, `sms`, `whatsapp` (`App\Enums\MessagingChannel`) |
| `driver` | Driver name from `config/messaging.php` |
| `credentials` | `encrypted:array`, encrypted with `APP_KEY`. Rotating `APP_KEY` makes stored credentials unreadable |
| `status` | `active` / `inactive` (`App\Enums\Status`) |
| `webhook_token` | Random 40-character token, reserved for per-company webhook URLs |

`message_deliveries`: one row per message handed to a provider, written by the channels
and the test endpoint.

| Column | Notes |
| --- | --- |
| `company_id` | The company whose delivery settings were used. Null when there was none |
| `channel`, `driver` | What sent it |
| `credential_source` | `company`, `platform`, `fallback` (`App\Enums\MessagingCredentialSource`) |
| `recipient` | Address or E.164 number |
| `provider_message_id` | The provider's id: Resend `X-Resend-Email-ID`, Africa's Talking `messageId`, Meta `wamid`. Null for SMTP and the log driver |
| `status` | `sent`, `failed` now; `delivered`, `read`, `undelivered` are for webhooks (`App\Enums\MessageDeliveryStatus`) |
| `error` | Provider's reason on failure |
| `notification_type`, `notifiable_type`, `notifiable_id` | What produced it, when a notification did |

## Webhooks (not built)

Only delivery exists. The foundations for delivery-status webhooks are in place:

- `message_deliveries.provider_message_id` is indexed with `driver`. A webhook resolves the
  row by it and moves `status` and `status_updated_at`.
- Signing secrets have a home at both levels: `RESEND_WEBHOOK_SECRET`,
  `WHATSAPP_META_APP_SECRET`, `WHATSAPP_META_VERIFY_TOKEN` and `AFRICASTALKING_WEBHOOK_TOKEN`
  for the platform, and the optional `webhook_secret`, `app_secret` and `verify_token`
  credential fields for a company.
- `company_messaging_channels.webhook_token` lets a company-level callback URL identify its
  row without exposing the numeric id, e.g. `/api/v1/webhooks/{driver}/{webhook_token}`.
  Africa's Talking does not sign callbacks, so for it the URL token is the only proof of
  origin.

## Environment

| Variable | Used for |
| --- | --- |
| `MAIL_MAILER` | Platform email driver: `resend` in production, `smtp` (Mailtrap) for testing, `log` locally |
| `MAIL_HOST`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `MAIL_ENCRYPTION` | SMTP |
| `RESEND_API_KEY`, `RESEND_WEBHOOK_SECRET` | Resend |
| `MAIL_FROM_ADDRESS`, `MAIL_FROM_NAME` | Platform sender. With Resend, the address must be on a domain verified on the account |
| `SMS_DRIVER` | Platform SMS driver (`africas_talking`) |
| `AFRICASTALKING_USERNAME`, `AFRICASTALKING_API_KEY`, `AFRICASTALKING_SENDER_ID`, `AFRICASTALKING_SANDBOX`, `AFRICASTALKING_WEBHOOK_TOKEN` | Africa's Talking |
| `WHATSAPP_DRIVER` | Platform WhatsApp driver (`meta_cloud`) |
| `WHATSAPP_META_PHONE_NUMBER_ID`, `WHATSAPP_META_BUSINESS_ACCOUNT_ID`, `WHATSAPP_META_ACCESS_TOKEN`, `WHATSAPP_META_APP_SECRET`, `WHATSAPP_META_VERIFY_TOKEN`, `WHATSAPP_META_VERSION` | Meta Cloud API |
| `MESSAGING_TIMEOUT` | HTTP timeout in seconds for provider calls (default 15) |

`config/sms.php` and `App\SDKs\SMSSdk` are gone. The old `SMS_API_USERNAME`, `SMS_API_KEY`
and `SMS_SHORTCODE` variables are no longer read. Their replacements are
`AFRICASTALKING_USERNAME`, `AFRICASTALKING_API_KEY` and `AFRICASTALKING_SENDER_ID`.

## Testing

`Messaging::fake()` swaps the SMS and WhatsApp drivers for one `MessagingFake`, which
records sends (`assertSentTo()`, `assertSentCount()`, `failNextWith()`). Email uses
`Mail::fake()` or `Http::fake()` against `api.resend.com`.
