# Welcome email

One email welcomes a person to any portal: staff (`app`), tenant, supplier (`vendor`) and
landlord. It is built on the shared shell described in [../emails.md](../emails.md).

## When it is sent

`App\Actions\Auth\SendWelcomeEmailAction::execute($user, $portal, $password, $company, $newAccount)`
is called where the account is created, inside that request's transaction. The email leaves only
after the commit (`OutboundEmail::send()`), so a request that fails and rolls back sends nothing.

| Where | Portal | New account | Existing person | Company (branding and mailer) |
|---|---|---|---|---|
| `POST app/{company}/users/tenants` (`TenantController::store`) | `tenant` | Generated password shown | Elevated: no credentials. Nothing when already a tenant | The route's company |
| `POST app/{company}/users/vendors` (`VendorController::store`) | `vendor` | Generated password shown | Elevated: no credentials (an existing supplier is refused) | The route's company |
| `POST app/{company}/users/landlords` (`LandlordController::store`) | `landlord` | Generated password shown | Elevated: no credentials | The route's company |
| `POST app/{company}/property-management/.../leases` with an inline `user` (`LeaseController::store`) | `tenant` | Generated password shown | (the inline tenant is always new) | The route's company |
| `POST vendor/users` (`VendorUserController::store`, a supplier's own user) | `vendor` | Generated password shown | Elevated: no credentials. Nothing when already a supplier user | None: platform branding |
| `POST app/{company}/access-management/company-users` (`CreateCompanyUserAction`) | `app` | Generated password shown | Given the staff portal now: no credentials. Nothing when already staff | The route's company |
| `POST auth/register` (`RegisterController`) | The registered portal | Sign-in id only (they chose the password) | (always new) | None: platform branding |

The action sends nothing when:

- the user has no address, or an EPMAS placeholder address (`...@no-email.epmas.import`);
- it runs inside `OutboundEmail::suppress()`.

Otherwise it stamps `users.welcome_emailed_at` (quietly, in the caller's transaction) and sends
`App\Notifications\Auth\WelcomeNotification`.

The notification is queued, encrypted on the queue (`ShouldBeEncrypted`, it may carry a
password) and goes by mail only: it is never written to the in-app notification list.

## What it contains

Title `WELCOME`, then:

1. Greeting and an intro for the portal. For an existing person given a new portal, the intro
   says their existing sign-in now opens that portal too.
2. **Your account** card (new accounts only): sign-in email, phone and, only when the system set
   it, the password, followed by a line asking them to change it after signing in.
3. The portal's section, from `config('emails.welcome_sections')`. A portal with no registered
   section is skipped.
   - Staff: **Your access** card with the company and the user's staff roles in it.
   - Supplier: the prequalified categories (see the supplier email ticket).
   - Landlord: the properties managed for them, with their status (see the landlord email ticket).
   - Tenant: nothing; the lease email carries the property and lease detail.
4. **Sign in** button.
   - Staff, supplier, landlord: the sign-in page with a redirect to their portal
     (`FrontendUrl::login('portals.{portal}')`, e.g. `/auth/login?redirect=%2Fvendor`).
   - Tenant: the plain sign-in page (`FrontendUrl::to('login')`), since a tenant without an
     active lease is refused at `/tenant`.
5. **What to expect**, a numbered list per portal. Tenant: invoices, payments, receipts and
   statements, maintenance. Supplier: requests for quotation, purchase orders, invoices, payments.
   Landlord: properties, remittances, payments, reports. Staff: access, tasks, notifications.
6. A closing note with the company's email and phone.

Subjects: "Welcome to {company}" for a new account; "Your {portal} access at {company}" for an
existing person; "Your {company} account is ready" for a migrated person.

## Migrated people

```
php artisan users:send-migration-welcome [--company=] [--portal=] [--limit=] [--dry-run]
```

Welcomes the people imported from EPMAS. A person is picked when:

- `users.old_system_id` is set;
- `users.welcome_emailed_at` is empty (so each person is welcomed once, by this command or by
  a creation site);
- the account is active and not deleted;
- the address is real (not `...@no-email.epmas.import`);
- they belong to a portal. People with no portal are counted and skipped.

The email is the welcome without a password. It says to sign in with the same email or phone
number and the password they used on EPMAS, and the button note links to Forgot password
(`FrontendUrl::to('forgot_password')`) for anyone whose password does not work (people who had
no EPMAS password were given a random one at import). The button goes to the plain sign-in page.

| Option | Effect |
|---|---|
| `--portal=` | Only people holding that portal (`app`, `tenant`, `vendor`, `landlord`), welcomed in it. Without it, each person's portal is the one `App\Support\RecipientPortal` picks: staff, then landlord, then supplier, then tenant |
| `--company=` | The company whose branding and mailer every email uses. Without it: the person's staff company (`company_users`), else the company of their first lease, else of their first property as landlord, else the first company |
| `--limit=` | Send at most this many |
| `--dry-run` | Print the counts per portal; send and stamp nothing |

People are processed in chunks of 200; each is stamped as their email is queued. The command
prints a table of sent (or, on a dry run, would-send) counts per portal.

Run it with a real queue worker. To rehearse, use `MAIL_MAILER=log` and `--dry-run` first.

## Preview

Locally, `GET /_mail/welcome` renders it for a real user. Query parameters:
`portal=app|tenant|vendor|landlord` (default `tenant`) and `variant=new|existing|migrated`
(default `new`, with a sample password).
