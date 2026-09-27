# Access Management: Messaging Channels

A company can deliver its email, SMS and WhatsApp through its own provider account.
Each channel is one record: the driver the company picked and the credentials for it.

A channel with no record, an inactive record, or incomplete credentials sends through the
platform's own account, configured on the server. Nothing breaks when a company never
configures anything.

Base prefix:

`/api/v1/app/{company}/access-management`

## Endpoints

- `GET /messaging-channels/drivers`: drivers, their credential fields, and what each channel sends through now
- `GET /messaging-channels`
- `POST /messaging-channels`
- `GET /messaging-channels/{messagingChannel}`
- `PUT/PATCH /messaging-channels/{messagingChannel}`
- `DELETE /messaging-channels/{messagingChannel}`
- `PATCH /messaging-channels/{messagingChannel}/activate`
- `PATCH /messaging-channels/{messagingChannel}/deactivate`
- `POST /messaging-channels/{messagingChannel}/test`: send a test message with this record's credentials

## Permissions

| Endpoint | Permission |
| --- | --- |
| `drivers`, `index`, `show` | `view-messaging-channel` |
| `store` | `create-messaging-channel` |
| `update`, `activate`, `deactivate`, `test` | `update-messaging-channel` |
| `destroy` | `delete-messaging-channel` |

## Company Scope

- `company_id` comes from route `{company}`. Do not send it.
- A company has at most one record per channel. A second `POST` for the same channel returns `422`.
- A record from another company returns `404`.

## Driver Catalog

`GET /api/v1/app/{company}/access-management/messaging-channels/drivers`

Build the settings form from this response. Do not hard-code field lists: a new
provider appears here without a frontend change.

```json
{
  "data": {
    "channels": [
      {
        "channel": { "value": "sms", "color": "info" },
        "label": "SMS",
        "platform": { "driver": "africas_talking", "configured": true },
        "effective": {
          "driver": "africas_talking",
          "source": { "value": "platform", "color": "info" }
        },
        "drivers": [
          {
            "name": "africas_talking",
            "label": "Africa's Talking",
            "fields": [
              { "key": "username", "label": "Username", "type": "string", "required": true, "secret": false, "options": [], "help": null },
              { "key": "api_key", "label": "API key", "type": "secret", "required": true, "secret": true, "options": [], "help": null },
              { "key": "sender_id", "label": "Sender ID", "type": "string", "required": false, "secret": false, "options": [], "help": "Alphanumeric sender ID or short code registered on the account." },
              { "key": "sandbox", "label": "Sandbox", "type": "boolean", "required": false, "secret": false, "options": [], "help": null }
            ]
          }
        ]
      }
    ]
  }
}
```

- `platform.configured`: whether the platform account for this channel has credentials.
  When it is `false` and the company has no working record, messages are only written to
  the server log.
- `effective.source`:
  - `company`: the company's active record with complete credentials
  - `platform`: the platform account
  - `fallback`: nothing is configured, so messages are logged and not delivered
- `fields[].type`: `string`, `secret`, `email`, `integer`, `boolean` or `select` (values in `options`).
  Render `secret` as a password input.

Drivers by channel:

| Channel | Driver | Required fields | Optional fields |
| --- | --- | --- | --- |
| `email` | `resend` | `api_key`, `from_address` | `from_name`, `reply_to`, `webhook_secret` |
| `email` | `smtp` | `host`, `port`, `from_address` | `username`, `password`, `encryption` (`tls`, `ssl`), `from_name`, `reply_to` |
| `sms` | `africas_talking` | `username`, `api_key` | `sender_id`, `sandbox` |
| `whatsapp` | `meta_cloud` | `phone_number_id`, `access_token` | `business_account_id`, `app_secret`, `verify_token` |

`webhook_secret`, `app_secret` and `verify_token` are not needed to send. They are stored
now so delivery-status webhooks can be switched on later without asking companies again.

## List

`GET /api/v1/app/{company}/access-management/messaging-channels`

- Filters: `filter[channel]`, `filter[driver]`, `filter[status]`
- Sort: `sort=channel`, `sort=driver`, `sort=created_at` (default: latest first)
- Pagination: `per_page`, `page`

## Create Payload

`POST /api/v1/app/{company}/access-management/messaging-channels`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `channel` | string | Yes | `email`, `sms` or `whatsapp` |
| `driver` | string | Yes | A driver `name` the catalog lists for that channel |
| `credentials` | object | Yes | Keys from that driver's `fields`. Every `required` field must be filled |

A new record is `active`, so the company sends through it straight away. Use `test` first
if you want to check the credentials.

```json
{
  "channel": "email", // email | sms | whatsapp
  "driver": "resend", // resend | smtp (email), africas_talking (sms), meta_cloud (whatsapp)
  "credentials": {
    "api_key": "re_123456789",
    "from_address": "notices@acme-properties.co.ke",
    "from_name": "Acme Properties",
    "reply_to": "accounts@acme-properties.co.ke"
  }
}
```

## Update Payload

`PUT/PATCH /api/v1/app/{company}/access-management/messaging-channels/{messagingChannel}`

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `driver` | string | No | Omit to keep the current driver. The channel cannot change |
| `credentials` | object | No | See below |

- **Same driver:** keys you send replace stored ones; keys you omit are kept. A `secret`
  field sent as `null`, `""`, or the masked value from the response keeps its stored
  value, so the form can post back what it loaded.
- **Driver changed:** the stored credentials are discarded, and `credentials` must carry
  every required field of the new driver.

## Response Shape

`messaging_channel` in `store`, `update`, `activate`, `deactivate` and `destroy`; `data` in `show`.

```json
{
  "id": 4,
  "channel": { "value": "email", "color": "primary" },
  "driver": "resend",
  "credentials": {
    "api_key": "••••••••6789",
    "from_address": "notices@acme-properties.co.ke",
    "from_name": "Acme Properties",
    "reply_to": "accounts@acme-properties.co.ke"
  },
  "is_configured": true,
  "status": { "value": "active", "color": "success" },
  "created": { "raw": "...", "formatted": "27 Sep, 2026", "diff": "1 minute ago" },
  "updated": { "raw": "...", "formatted": "27 Sep, 2026", "diff": "1 minute ago" }
}
```

- Secret values never leave the server. A stored secret comes back masked (`••••••••` plus
  its last four characters); an empty one comes back `null`.
- `is_configured`: every required field is filled. An active record that is not configured
  is skipped, and the channel sends through the platform account.

## Activate / Deactivate

`PATCH .../messaging-channels/{messagingChannel}/activate`, `.../deactivate`

Deactivating keeps the credentials but sends through the platform account until the record
is activated again. `DELETE` removes the record and its credentials.

## Send a Test

`POST /api/v1/app/{company}/access-management/messaging-channels/{messagingChannel}/test`

Sends one message using this record's driver and credentials, whether or not the record is
active. It never falls back to the platform account, so a failure here means these
credentials failed.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `recipient` | string | Yes | An email address for `email`; a phone number for `sms` and `whatsapp` |
| `message` | string | No | Defaults to a short test text |

```json
{
  "data": {
    "message": "Test message sent.",
    "result": {
      "successful": true,
      "driver": "africas_talking",
      "recipient": "+254712345678",
      "provider_message_id": "ATXid_4f1...",
      "error": null
    }
  }
}
```

A provider rejection is still a `200`, with `successful: false` and the provider's reason in
`error`. Show it to the user. Credentials that are incomplete return `422`.

WhatsApp note: Meta delivers free-form text only inside the 24-hour window after the
recipient last messaged the business number. A test to a number that has never messaged it
comes back with `successful: false`.
