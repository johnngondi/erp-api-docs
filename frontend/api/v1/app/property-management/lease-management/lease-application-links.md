# Lease applications through a signed link

Staff start a lease application with the applicant's details, the lease terms and the spaces;
the applicant finishes it on a page they open from a signed link, with no sign-in. Submitting
locks the link and hands the application to the property team as an ordinary submitted
application (review, approval, Letter of Offer as before).

Row 157b. Built to the PM's spec *Lease Application via Signed Link — Dev Tasks*.

- [Staff: initiate](#staff-initiate)
- [Staff: resend and revoke](#staff-resend-and-revoke)
- [Staff: the application afterwards](#staff-the-application-afterwards)
- [The applicant's page](#the-applicants-page)
- [Link states and errors](#link-states-and-errors)
- [Documents](#documents)
- [Notifications](#notifications)

## Staff: initiate

`POST /api/v1/app/{company}/property-management/lease-management/lease-applications/initiate`

Needs `view-lease-application`, on a property the user is allocated to (403 otherwise). That's
the permission the Initiate button uses too. It will move to `create-lease-application` once that
permission can be given to staff roles; the button should move with it.

| Field | Required | Type | Notes |
|---|---|---|---|
| `applicant_type` | yes | `business` \| `personal` | Business is the default on the form |
| `name` | yes | string | Full names; the registered company name for a business |
| `phone` | yes | string | International format, `+254 712 345 678` |
| `email` | yes | email | The link is sent here |
| `tax_pin` | yes | string | KRA PIN, `A` or `P`, 9 digits, a letter (`P051234567X`). Case-insensitive |
| `facility_id` | yes | integer | The property |
| `proposed_start_date` | yes | date `Y-m-d` | |
| `proposed_period_in_years` | yes | integer 0–99 | |
| `proposed_period_in_months` | no | integer 0–11 | Years and months together must be at least one month |
| `fit_out_period_months` | no | integer 0–120 | Tenancy preferences |
| `billing_cycle` | no | `monthly` \| `quarterly` \| `biannual` \| `annually` | "Half-yearly" on the design is `biannual` |
| `spaces` | yes | array, at least 1 | Same items as the spaces picker sends to `POST …/{application}/spaces`: `facility_space_id` and `components[]` (`lease_component_id`, `cost_per_space_unit`, optional `tax_id`) |

The proposed end date is not sent: it follows from the start and the term (start + term − 1 day)
and comes back on the application.

**What it does**, in one transaction:

1. **Resolves the applicant by KRA PIN.**
   - A PIN that already has an account → that account, given the Tenant portal if it lacks it
     (any other portal it holds is kept). If it was newly given the portal, it gets the usual
     welcome email, without a new password.
   - A new PIN → a new account with a **generated password**, sent in the usual tenant welcome
     email (as when staff create a tenant). The applicant never needs it to finish the
     application: the link needs no sign-in.
   - An email or phone that belongs to a *different* account → `422` on `email` / `phone`.
     Nothing is created.
2. Creates the application, `status: pending`, waiting on the applicant (see
   [below](#staff-the-application-afterwards)), with a reference like `LA-2026-0142`.
3. Attaches each space. A space that is not available is refused (`422`), exactly as the spaces
   picker refuses it, so a space already on another open application cannot be taken twice.
4. Issues the link, valid **7 days**, and sends it by email, SMS and WhatsApp. This is a separate
   message from the welcome email.

**Response** (`200`):

```json
{
  "data": {
    "message": "Application created and link sent to jane.kamau@example.co.ke.",
    "application": { "id": 142, "reference": "LA-2026-0142", "link": { "status": { "value": "sent", "color": "info", "label": "Link sent" }, "url": "https://…/apply/lease/7f3a9c2e-…?expires=1761078600&signature=…", "expires_at": { "raw": "…", "formatted": "21 Oct, 2026 20:58", "diff": "6 days from now" }, "opened_at": null, "revoked_at": null }, "...": "the usual application fields" },
    "link": "https://portal.nwrealite.co.ke/apply/lease/7f3a9c2e-51b4-…?expires=1761078600&signature=…",
    "link_expires_at": "2026-10-21T20:58:00+03:00",
    "applicant": { "user_id": 311, "outcome": "created", "tenant_group_added": true, "welcome_email_sent": true, "credentials_sent": true },
    "channels": ["mail", "sms", "whatsapp"]
  }
}
```

For the "What happened" panel:
- `applicant.outcome`: `matched` ("KRA PIN matched an existing user") or `created` ("Account
  created").
- `tenant_group_added`: whether the Tenant group was added.
- `welcome_email_sent`: whether the welcome email went out. False for an existing tenant, or when
  the address can't be emailed.
- `credentials_sent`: whether that email carried a new password, which only happens for a
  `created` account.
- `channels`: the channels the link was **queued** on. Whether each went out is in
  `application.link.deliveries` once the sends have run ([below](#delivery-status-per-channel)).

## Staff: resend and revoke

`POST …/lease-applications/{application}/resend-link`: issues a new link (new 7-day expiry) and
sends it again with "new link" wording. **Every earlier link stops working at once.** Also brings
back an expired or revoked link.

`POST …/lease-applications/{application}/revoke-link`: the link stops working immediately. `422`
on `link` when there is no live link to revoke.

Both need `view-lease-application` (as Initiate does) on an allocated property, and only while the applicant has
not submitted (`403` with the reason otherwise). Response:

```json
{ "data": { "message": "A new link has been sent. Earlier links no longer work.", "link": "https://…", "link_status": { "value": "sent", "color": "info", "label": "Link sent" }, "link_expires_at": "…", "channels": ["mail", "sms", "whatsapp"] } }
```

## Staff: the application afterwards

The application is a normal `pending` application with these additions on the staff resource
(`GET …/lease-applications/{application}` and the list):

| Field | Notes |
|---|---|
| `reference` | `LA-2026-0142`. Every new application gets one; older ones have none |
| `link` | `null` unless it was started with a link. `status` (`sent` → `opened` → `submitted`, or `expired` / `revoked`, with `color` and `label`), `url` while the link is live (for Copy link), `sent_at`, `expires_at`, `opened_at`, `reminded_at`, `revoked_at`, and on the single application only, `deliveries` ([below](#delivery-status-per-channel)) |
| `initiated_by` | The staff member who started it (load: on the show endpoint) |
| `trading_name`, `years_in_operation`, `staff_count`, `intended_use`, `previous_tenancy_period`, `reason_for_moving` | New optional details the applicant may fill |
| `upload_mode` | `individual` or `combined` |
| `declaration_accepted_at` | When the applicant ticked the declaration |

**Until the applicant submits** (`link.status` is anything but `submitted`), the application is
theirs: no review task is raised, it is not vetted, and review is refused ("The applicant has not
submitted this application yet."). Show it as **"Awaiting applicant"** with the link status. On
submit it becomes an ordinary submitted application: the reviewers get their task as usual.

The applicant details staff do not enter here (registration type and number, registered country,
physical and postal address) are empty on these applications. Staff can fill them through the
usual edit.

### Delivery status per channel

`link.deliveries` lists every message sent to the applicant about the **current** link, oldest
first: the link (or the resent link), its reminder, and the confirmation once they submit. It's
returned on `GET …/lease-applications/{application}` only, not on the list.

```json
"deliveries": [
  { "message": "link", "channel": "email", "recipient": "jane.kamau@example.co.ke", "status": "sent", "error": null, "at": "2026-10-14T20:58:03+03:00" },
  { "message": "link", "channel": "sms", "recipient": "+254 712 345 678", "status": "failed", "error": "Insufficient balance", "at": "2026-10-14T20:58:04+03:00" },
  { "message": "reminder", "channel": "email", "recipient": "jane.kamau@example.co.ke", "status": "sent", "error": null, "at": "2026-10-19T21:25:00+03:00" }
]
```

| Field | Values |
|---|---|
| `message` | `link`, `resent`, `reminder`, `confirmation` |
| `channel` | `email`, `sms`, `whatsapp` |
| `status` | `sent` (the provider accepted it) or `failed` (with `error`). `delivered` / `read` will appear once provider delivery webhooks are built. Not yet |

Right after "Generate link & send" the sends are still queued, so `deliveries` can be empty for a
few seconds. Refresh the application to show them.

## The applicant's page

A public route outside the signed-in app, `/apply/lease/{uuid}?expires=…&signature=…` (the link
the applicant receives). The page calls the API with the **same `expires` and `signature`** on
every request, no token or cookie:

| Method | Path (prefix `/api/v1/apply/lease/{uuid}`) | Body |
|---|---|---|
| `GET` | `` | — |
| `PATCH` | `/details` | any of the detail fields below |
| `POST` | `/guarantors` | `full_name`, `id_number`, `phone` (each optional; saved as typed) |
| `PATCH` | `/guarantors/{id}` | same |
| `DELETE` | `/guarantors/{id}` | — |
| `POST` | `/documents` | multipart: `file`, `document_type`, `guarantor_id` (for a guarantor's document) |
| `DELETE` | `/documents/{document_type}` | `?guarantor_id=` for a guarantor's document |
| `POST` | `/submit` | `{ "declaration": true }` |

e.g. `PATCH /api/v1/apply/lease/7f3a9c2e-…/details?expires=1761078600&signature=…`

**Every one of them answers with the whole page** (the `GET` shape below), so the page can
re-render from any reply. The first `GET` marks the link as opened.

### `GET` shape

```json
{
  "data": {
    "reference": "LA-2026-0142",
    "state": "open",
    "expires_at": "2026-10-21T20:58:00+03:00",
    "submitted_at": null,
    "company": { "name": "NW Realite Ltd" },
    "property": { "name": "Westgate Business Park" },
    "applicant": { "type": "business", "name": "Jane Wanjiku Kamau", "tax_pin": "P051234567X", "phone": "+254 712 345 678", "email": "jane.kamau@example.co.ke" },
    "lease": { "start_date": "2026-10-01", "term_years": 6, "term_months": 0, "end_date": "2032-09-30", "billing_cycle": { "value": "monthly", "color": "…" }, "fit_out_period_months": 2 },
    "spaces": [ { "name": "Suite 4B", "size": 1250, "components": ["Rent", "Service Charge"], "monthly": 187500 } ],
    "space_unit": "SqFt",
    "monthly_total": 195500,
    "details": {
      "business": { "trading_name": null, "nature_of_business": null, "years_in_operation": null, "staff_count": null, "intended_use": null },
      "landlord": { "name": null, "phone": null, "email": null, "premises": null, "period": null, "reason_for_moving": null },
      "referral": { "source": null, "referrer_name": null }
    },
    "upload_mode": null,
    "documents_needed": ["Certificate of incorporation / business registration", "KRA PIN certificate (company)", "…", "For each guarantor: National ID or passport, KRA PIN certificate, Proof of income, Signed guarantor form"],
    "documents": [
      { "type": "id", "label": "Certificate of incorporation / business registration", "required": true, "hint": null,
        "uploaded": { "file_name": "certificate.pdf", "size": 1258291, "type": "application/pdf", "status": { "value": "pending", "color": "warning" }, "reason": null } },
      { "type": "cr12", "label": "CR12 (company directors & shareholding)", "required": true, "hint": "Issued within the last 3 months", "uploaded": null }
    ],
    "combined_document": null,
    "guarantor_documents": [ { "type": "id", "label": "National ID or passport", "required": true, "hint": "Both sides", "uploaded": null } ],
    "guarantors": [ { "id": 51, "full_name": "Peter Otieno", "id_number": "12345678", "phone": "+254700111222", "documents": [ { "type": "id", "label": "…", "required": true, "hint": "…", "uploaded": null } ] } ],
    "declaration_accepted_at": null,
    "missing": ["Upload the CR12 (company directors & shareholding).", "Add at least one guarantor."]
  }
}
```

- `documents_needed` is the one-line "Documents needed" list and the combined scan's ordered
  checklist.
- `missing` is what the application still lacks, ready to list as warnings. **Advisory only**:
  the applicant can submit without them, and staff add them before approving.
- `guarantor_form_url` is the blank guarantor form for "Download the form", or `null` until one
  is configured (hide the link then).
- Amounts are monthly, VAT exclusive.

### Details (`PATCH /details`)

Send only what changed (autosave); `null` clears a field.

| Section | Fields |
|---|---|
| Business details | `trading_name`, `nature_of_business`, `years_in_operation` (int), `staff_count` (int), `intended_use` |
| Current or previous landlord | `previous_landlord_name`, `previous_landlord_phone`, `previous_landlord_email`, `previous_landlord_address` (premises), `previous_tenancy_period` (e.g. "Jan 2022 – present"), `reason_for_moving` |
| Referral | `referral_source` (`staff`, `agent`, `social_media`, `website`, `billboard`, `other`), `referrer_name` |

The design's single "Landlord contact" box maps to `previous_landlord_phone` or
`previous_landlord_email`. Send whichever was typed.

### Submit (`POST /submit`)

`{ "declaration": true }`. The declaration is the only requirement: `422` on `declaration` when
it isn't accepted. Missing documents or guarantors **don't** block it (PM decision). Staff
complete the application before approving it.

On success the reply has `state: "submitted"`, and the link becomes **read-only**: `GET` still
answers (show the confirmation with the reference), and every change answers `409` with
`reason: "submitted"`.

Required documents and the guarantor still drive the badges and `missing`, as guidance.

## Link states and errors

Every call checks the link first. A refusal is `403` (or `409` for submitted) with a `reason`
for the page to show its full-page state:

| `reason` | When | Page |
|---|---|---|
| `invalid` | Signature tampered with, or the link was replaced by a resend | "This link is not valid" |
| `expired` | Past its expiry | "Expired, ask the property team to resend" |
| `revoked` | Staff revoked it | "Withdrawn" |
| `submitted` (`409`) | A change after submitting | Confirmation, read-only |

```json
{ "message": "This link has expired. Ask the property team to send you a new one.", "reason": "expired" }
```

The routes are rate-limited (120 requests a minute per application and IP).

## Documents

Upload with `POST /documents` (multipart). The applicant's slots come from `documents` in the
`GET` reply, and the guarantors' from `guarantor_documents`. Render whatever the API returns: the
list is configuration, not code.

| Applicant type | Slots |
|---|---|
| Business | all required: `id` Certificate of incorporation / business registration · `tax_pin` KRA PIN certificate (company) · `cr12` CR12 · `business_permit` Single business permit · `financial_statement` Financial statement · `director_id` ID of signing director / contact person |
| Personal | `id` National ID or passport (required) · `financial_statement` Financial statement (required) · `tax_pin` KRA PIN certificate (optional) |
| Each guarantor | `id` National ID or passport (required) · `tax_pin` KRA PIN certificate · `financial_statement` Proof of income · `guarantor_form` Signed guarantor form. The last three are required for a business applicant's guarantors and optional for a personal applicant's |

The PM's list (2026-10-14). "Required" is advisory, as above. Changing the list is a config
change on our side, with no change on yours.

- **Individual files**: `document_type` = the slot's `type`, plus `guarantor_id` for a
  guarantor's. PDF, JPG or PNG, up to **10 MB**. Uploading into a filled slot replaces it.
- **Combined document**: `document_type` = `combined`, **PDF only**, up to **25 MB**. Guarantor
  cards (name, ID, phone) are still added with `POST /guarantors`, without their document slots.
- **One mode at a time**: the first upload in one mode clears what the other mode left, and
  `upload_mode` says which mode is active. Warn before switching.
- An ID or KRA PIN is checked automatically after upload, as on any application (157a), so a slot's
  `uploaded.status` can move from `pending` to `verified` or `failed` (with `reason`).
- **Combined scan, split automatically (157c):** see [below](#the-combined-scan).
- **Guarantor form template**: `guarantor_form_url` once the PM has uploaded it.

## The combined scan

After a combined PDF is uploaded, it's split into its documents in the background:

1. Claude reads the whole scan once, told exactly what this application expects: the applicant's
   slots for their type, and each guarantor by name with theirs. It says which pages are which
   document.
2. Those pages are cut into their own PDFs and filed into the matching slots, the applicant's or
   the right guarantor's, as if uploaded one by one. Each one's file title says where it came from,
   e.g. "KRA PIN certificate, pages 3-4 of the combined scan". IDs and KRA PINs found this way are
   verified as usual.
3. Pages that match nothing asked for (an application form, a page the model couldn't place) go
   into one `other` document, so nothing is lost. **The combined file stays saved.**

Progress is on the page (and on the staff application) as `combined_scan`:

```json
"combined_scan": {
  "status": "done",
  "error": null,
  "pages": 5,
  "documents": [
    { "type": "id", "label": "National ID or passport", "guarantor": null, "pages": [1], "page_range": "page 1" },
    { "type": "tax_pin", "label": "KRA PIN certificate", "guarantor": null, "pages": [2], "page_range": "page 2" },
    { "type": "id", "label": "National ID or passport", "guarantor": 1, "pages": [4], "page_range": "page 4" },
    { "type": "other", "label": "Other pages", "guarantor": null, "pages": [5], "page_range": "page 5" }
  ]
}
```

| `status` | Meaning | Show |
|---|---|---|
| `processing` | Right after the upload; usually under a minute | "Reading your document…", and poll `GET` (every 5 s) until it changes |
| `done` | The slots above are filled. `guarantor` is the card's position (1 = first) | The filled slots, e.g. "From your combined scan, page 2" |
| `failed` | `error` says why: unreadable, more than **100 pages**, or the reader was unavailable | The error; staff will sort the scan by hand, and the applicant can still submit |

`null` until a combined scan is uploaded. A new combined scan re-runs the split and takes back what
the earlier one filed, but not a slot someone has since replaced with their own file. Add the
guarantor cards (name, ID, phone) **before** uploading the combined scan, so their documents can be
matched to them.

## Notifications

| When | To | Channels |
|---|---|---|
| Link generated / resent | Applicant (the email and phone staff entered) | Email, SMS, WhatsApp |
| Application submitted | Applicant | Email, SMS, WhatsApp (confirmation with the reference) |
| Application submitted | Staff member who started it | Bell + email, linking to the application |
| Reminder, once, when 48 hours are left and it's unsubmitted | Applicant | Email, SMS, WhatsApp, with the same link |
| Link expired unsubmitted | Staff member who started it | Bell. Its data carries `action: "resend_link"` and `lease_application_id`, for a Resend button calling `POST …/{application}/resend-link` |

Reminders and expiry run hourly (`lease-applications:process-links`, at :25). An expired link
shows `link.status` `expired`. Resending issues a new 7-day link with its own reminder.

**WhatsApp** goes through the same sender as every other WhatsApp notification. Meta only
delivers to a number that has messaged us in the last 24 hours unless the message is an approved
template. Template support is for whoever owns the WhatsApp sender to add, and these messages will
use it with no change here. Until then, expect email and SMS to carry the link. 
