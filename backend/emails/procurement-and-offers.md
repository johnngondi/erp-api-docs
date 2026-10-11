# Procurement and offer emails

Eight emails built on the shared shell in [emails.md](../emails.md): the RFQ invitation, the
quote reminder, the three RFQ round emails (cancelled, reopened, deadline extended), the Letter of
Offer, the returned lease application and the invitation to apply. All are notifications marked
`SendsEmail` and `HasNotificationCompany`, so they carry the company's branding and go through its
mailer.

| Email | Class | Recipient | Sent when | Button |
|---|---|---|---|---|
| Request for quotation | `RfqInvitationNotification` | each invited, active supplier | the procurement step of a work request is approved and the request opens for quotes | "Submit quote" → `vendor.bid` |
| Quote reminder | `QuoteSubmissionReminderNotification` | each invited supplier who has not quoted | once per working day, by `RemindPendingQuotesService`, until they quote or the deadline passes | "Submit quote" → `vendor.bid` |
| RFQ cancelled | `RfqCancelledNotification` | each active supplier whose invitation a rebase cancelled | `RebaseProcurementRequestAction` | none |
| RFQ reopened | `RfqReopenedNotification` | each active supplier whose quote a rebase shelved, carried into the next round | `ReviewStepAction` sends a rebased request out again | "Submit revised quote" → `vendor.bid` |
| Quote deadline extended | `RfqDeadlineExtendedNotification` | each active supplier whose invitation is open | `RequestMoreProcurementQuotesAction` extends the deadline after a quote shortfall | "View request" → `vendor.rfq` |
| Letter of Offer | `LooSentNotification` | the tenant or applicant, or the address the sender typed | `SendLooAction` sends the offer | "Read and respond" → `tenant.loo`, or "Sign in" → `login` |
| Lease application returned | `LeaseApplicationReturnedNotification` | the staff member who approved the application, else the reviewer-role holders | `ReturnLeaseApplicationAction` sends an approved application back for review, directly or through a LOO rebase | "Review application" → `app.lease_application` |
| Invitation to apply | `LeaseApplicantInvitationNotification` | a prospective tenant, on demand (no account yet) | staff send it from the lease applications list (`InviteLeaseApplicantAction`) | "Start application" → `register` with `apply=lease` and the applicant's details |

## Request for quotation

Sent from `ReviewStepAction::sendVendorNotification()`. That point is reached only for a `work`
request whose procurement step is approved without a quote change, which moves the request to
`order` and the step to `waiting-quotes`. Purchase requests never invite. The emergency
default supplier is invited the same way.

- **Recipients** are read fresh from `$procurementRequest->vendors()` (on the first round the
  invite rows are deleted and recreated by `AttachVendorsToRequestAction`; later rounds append),
  with the supplier loaded. Cancelled invitations are skipped, and so is a supplier whose account
  is not `active`, the same filter as the quote reminder. A supplier carried over after a rebase
  gets `RfqReopenedNotification` instead.
- **Once per supplier per request.** Before sending, the action looks for a stored
  `RfqInvitationNotification` in `notifications` with this request as its subject
  (`subject_type`/`subject_id`, stamped by `DatabaseChannel`) and the supplier as its
  notifiable. When one exists the supplier is skipped. A step sent back for review and
  approved again reaches the same point; only suppliers new to the request are invited then.
- **After commit.** The send goes through `OutboundEmail::send()`, so it waits for the step
  controller's transaction and nothing is sent when it rolls back.
- **Channels:** database and mail (mail only when the supplier has an address). Queued.

Content: title `REQUEST FOR QUOTATION`; eyebrow (property name) over the request title; an
invitation paragraph naming the company, property and deadline; a "Request details" card
(reference `RFQ #{id}`, property, category as expense type · sub-type, priority, quote
deadline, site visit with its instructions when one is required); the button; a note.

Database payload: `title`, `message`, `procurement_request_id`, `request_title`,
`facility_name`, `deadline`, plus the `resource_*` keys from `ResolvesRecipientResource`
(the supplier's open-job page).

## RFQ round emails

`RfqCancelledNotification`, `RfqReopenedNotification` and `RfqDeadlineExtendedNotification` share
`RfqSupplierNotification` (abstract): database and mail (never text), queued on `notifications`, the
request as the subject, sent through `OutboundEmail::send()` after commit. Inactive suppliers are
skipped.

- **Cancelled**: title `RFQ CANCELLED`; "{company} has cancelled the request for quotation for
  {title} at {property}. Any quotation you sent for it will not be considered."; a "Cancellation"
  card (reference, property, reason); a note. Payload adds `reason`.
- **Reopened**: title `RFQ REOPENED`; the job is open again, the scope may have changed, send a
  revised quotation before the deadline; a "Request details" card (reference, property, reason for
  the change = the rebase reason, quote deadline); the button with the request-page link in its note.
  Payload adds `deadline`.
- **Deadline extended**: title `QUOTE DEADLINE EXTENDED`; "{company} has extended the quote
  deadline for {title} at {property} to {deadline}."; a card (reference, property, new quote
  deadline); the button; a note that a supplier who already quoted need do nothing. Payload adds
  `deadline`.

Suppliers newly asked in a later round, or by "request more quotes", get the ordinary request for
quotation. On a round after a rebase, the "once per supplier per request" check only looks at
invitations and reopen notices sent since the rebase.

## Quote reminder

Unchanged except for the email: title `QUOTE REMINDER`, the reminder text with the deadline, a
"Request details" card (reference, request, property, category, quote deadline), the button and
a note that reminders stop once the supplier quotes or the deadline passes. The subject stays
"Quotation due for {title}". SMS, WhatsApp and the database record are as before; the database
record's `resource_url` still points at the open job.

The button used to be the in-app record's `resource_url`, which the resource-URL resolver builds
as an API route. It is now the frontend's submit-quote page.

## Letter of Offer

`LooSentNotification` keeps its channels: an on-demand recipient gets mail only; the tenant's
copy when the offer was addressed elsewhere (`databaseOnly`) gets the database record only;
otherwise database, plus mail when the tenant has an address.

Content: title `LETTER OF OFFER`; eyebrow (property name) over "Your Letter of Offer is ready";
"{company} has issued a Letter of Offer for {property}", the sender's optional note, and "Sign in
to your tenant portal to read it and record your acceptance"; a card with the reference
(`our_ref`) and property; the button; a closing note.

The offer is **attached**: the PDF `SendLooAction` filed as the Loo's `document_upload_id` (the
same file the tenant portal's download serves, never a fresh render), under its stored file name.
Every recipient gets it, an on-demand one included. When that file cannot be read the email
still goes without it, and the note sends the reader to the portal instead of saying it is
attached.

- An account holder's button is **Read and respond** → `tenant.loo` with the offer's id.
- An on-demand recipient (`AnonymousNotifiable`: an applicant with no portal login, or a copy
  addressed to an agent) gets **Sign in** → `login`, since there is no portal page of theirs to
  open.

There is no PDF attachment and no public link: the offer lives behind the authenticated tenant
portal, where `Loo::visibleToTenant()` still applies.

## Lease application returned

Staff, not tenants: the person asked to review an approved application again because it is not
fit for a Letter of Offer yet. Sent through `OutboundEmail::send()` after the return commits, to
the same people who get the "Re-review returned lease application" pending task
(`LeaseApplicationResponsibleUsers::returnedReviewers()`).

- **Channels:** database, mail when the reviewer has an address, and SMS when they have a phone
  number. Queued on `notifications`. This is one of the few staff messages allowed out by text;
  the user asked for it on 7 October 2026 (see [messaging.md](../messaging.md)).
- **Content:** title `APPLICATION RETURNED`; "{returner} returned the lease application from
  {applicant} for {property}. It is not ready for a Letter of Offer yet."; the reason in a
  callout; a card with the applicant, property and returned date; the button.
- **SMS:** one line with the applicant, property and reason.

Database payload: `title`, `message`, `lease_application_id`, `return_reason`, plus the
`resource_*` keys from `ResolvesRecipientResource`.

## Links

| Key | Path |
|---|---|
| `vendor.bid` | `/vendor/procurement/rfq-quotes/create?job={job}`; `job` is the procurement request id |
| `vendor.rfq` | `/vendor/procurement/open-jobs/{id}`; linked from the invitation's button note |
| `tenant.loo` | `/tenant/lease-management/loos/{id}` |
| `app.lease_application` | `/app/property-management/lease-management/lease-applications/{id}` |
| `login` | `/auth/login` |

## Previews

`/_mail/rfq-invitation`, `/_mail/quote-reminder` and `/_mail/loo` render the latest work request
(with a deadline, for the reminder) or the latest offer. They show "no sample data" when the
database has none.
