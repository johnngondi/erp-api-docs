# QuickBooks Online API

Domain: `Property Management > Finance`

Base route:

`/api/v1/app/{company}/property-management/finance/quickbooks`

Each company connects its own QuickBooks Online company (one realm per company). The connection is
what customer invoices are sent through, and what landlords are created in as QuickBooks customers
(see [Landlords: customer and QuickBooks](../../users/landlords.md#customer-and-quickbooks)).

## Endpoints

- `GET /quickbooks/connection`
- `POST /quickbooks/connection`
- `DELETE /quickbooks/connection`
- `GET /quickbooks/items`
- `GET /quickbooks/tax-codes`

Every endpoint needs the `manage-quickbooks` permission (`403` otherwise).

## Connection status

`GET /api/v1/app/{company}/property-management/finance/quickbooks/connection`

Not connected:

```json
{
  "data": {
    "connected": false,
    "environment": "sandbox"
  }
}
```

Connected:

```json
{
  "data": {
    "connected": true,
    "environment": "production",
    "realm_id": "9341452811542090",
    "connected_at": { "raw": "2026-10-19T09:12:00.000000Z", "formatted": "19 Oct, 2026", "diff": "2 hours ago" },
    "connected_by": { "id": 12, "name": "Jane Wanjiru" },
    "refresh_token_expires_at": { "raw": "2027-01-27T09:12:00.000000Z", "formatted": "27 Jan, 2027", "diff": "3 months from now" },
    "last_error": null
  }
}
```

| Field | Notes |
|---|---|
| `environment` | `sandbox` or `production`, from the server's `QUICKBOOKS_ENVIRONMENT`. Not chosen per company |
| `refresh_token_expires_at` | QuickBooks refresh tokens last about 100 days and are renewed every time they are used. A company that sends nothing for that long must connect again |
| `last_error` | The last token refresh failure, e.g. the user revoked access in QuickBooks. Show it with a **Reconnect** button. `null` when healthy |

## Connect

`POST /api/v1/app/{company}/property-management/finance/quickbooks/connection`

Starts the Intuit sign-in. The frontend sends the user's browser to `authorize_url`; Intuit asks them
to sign in and pick a company, then returns to the backend callback, which stores the tokens and
redirects the browser back to the frontend.

Payload:

| Field | Required | Type | Notes |
|---|---|---|---|
| `return_path` | No | string | A path in the frontend app to come back to, starting with `/` (e.g. `/app/property-management/settings/finance`). Default `/`. Full URLs are refused (`422`) |

Response:

```json
{
  "data": {
    "authorize_url": "https://appcenter.intuit.com/connect/oauth2?client_id=...&scope=com.intuit.quickbooks.accounting&redirect_uri=...&response_type=code&state=..."
  }
}
```

The browser comes back to `{FRONTEND_URL}{return_path}` with one of:

- `?quickbooks=connected`
- `?quickbooks=error&message=<url-encoded reason>`, e.g. the user cancelled on Intuit, or the link
  expired (the `state` is valid for 30 minutes).

Connecting again while connected replaces the stored connection (for example to switch QuickBooks
company). Landlords already linked keep their customer ids, so switching companies needs a fresh
[sync](../../users/landlords.md#customer-and-quickbooks).

Errors:

- `422` `QuickBooks is not configured on this server.` when the Intuit app keys are missing.

## Disconnect

`DELETE /api/v1/app/{company}/property-management/finance/quickbooks/connection`

Revokes the tokens at Intuit (best effort) and forgets the connection. Landlord customer links are kept.

```json
{ "data": { "message": "QuickBooks disconnected." } }
```

`404` when the company is not connected.

## Items

`GET /api/v1/app/{company}/property-management/finance/quickbooks/items`

The active QuickBooks products and services, for the item settings below. Read live from QuickBooks.

```json
{
  "data": [
    { "id": "21", "name": "Management Fee", "type": "Service" },
    { "id": "22", "name": "Letting Fee", "type": "Service" }
  ]
}
```

## Tax codes

`GET /api/v1/app/{company}/property-management/finance/quickbooks/tax-codes`

The active QuickBooks tax codes, for the tax code settings below. Read live from QuickBooks.

```json
{
  "data": [
    { "id": "5", "name": "VAT 16%", "description": "Standard rated 16%" },
    { "id": "7", "name": "Zero-rated", "description": null }
  ]
}
```

Errors for `items` and `tax-codes`:

- `422` `QuickBooks is not connected for this company.`
- `422` with QuickBooks' own message when it refuses the request (e.g. an expired connection).

## Settings

Seeded per company in the general settings, group `finance` + `quickbooks`
(`GET /api/v1/app/{company}/property-management/settings/general?filter[group]=quickbooks`, see
[Settings](../settings.md#general-settings)).
The selects load their options from the endpoints above.

| Key | Type | Options | Notes |
|---|---|---|---|
| `quickbooks_item_management_fee` | `select` | `items` endpoint | QuickBooks item a management fee line is sent as |
| `quickbooks_item_letting_fee` | `select` | `items` endpoint | QuickBooks item a letting fee line is sent as |
| `quickbooks_item_re_letting_fee` | `select` | `items` endpoint | QuickBooks item a re-letting fee line is sent as. Empty = the letting fee item |
| `quickbooks_tax_code_vat` | `select` | `tax-codes` endpoint | Tax code for lines that carry VAT |
| `quickbooks_tax_code_zero` | `select` | `tax-codes` endpoint | Tax code for lines with no tax |
| `quickbooks_cu_custom_field` | `text` | | Name of the QuickBooks invoice custom field the ETR CU number is written to after signing. Default `Cu Invoice Number` |
| `quickbooks_invoice_terms_days` | `number` | | Days from the invoice date to its due date in QuickBooks. Default `30` |

## Server configuration

| Env | Notes |
|---|---|
| `QUICKBOOKS_CLIENT_ID`, `QUICKBOOKS_CLIENT_SECRET` | The Intuit developer app's keys |
| `QUICKBOOKS_REDIRECT_URI` | Must be registered on the Intuit app. `{APP_URL}/quickbooks/callback` |
| `QUICKBOOKS_ENVIRONMENT` | `sandbox` (default) or `production` |
| `QUICKBOOKS_DEBUG_LOG` | `true` writes each QuickBooks request and response to `storage/logs/quickbooks` |

## Commands

| Command | What it does |
|---|---|
| `php artisan quickbooks:sync-customers {company} [--customer=*] [--unlinked] [--dry-run]` | Makes every landlord of the company a customer (if missing) and syncs each to QuickBooks: found by PIN or email and updated, else created |
| `php artisan quickbooks:link-customers {company} --from-epmas [--deactivate-replaced] [--no-update] [--dry-run]` | Points customers at the QuickBooks customers EPMAS already had (`merchants.qbo_reference_id`, read on the `epmas` connection), matching merchants to landlords the way the EPMAS import did (`users.old_system_id` = `Merchant:{id}`, or `epmas_import_aliases`). Each id is checked in QuickBooks first; the landlord's details are then written onto it. `--deactivate-replaced` makes inactive the customer a sync had **created** for that landlord (never one it found). A landlord with several EPMAS QuickBooks ids is skipped for a person to decide |
| `php artisan quickbooks:link-customers {company} --map=links.csv [...]` | The same from a CSV of `user_id,qbo_customer_id`, when the EPMAS database is not reachable |

A customer linked this way shows `quickbooks.matched_by = linked`.
