# Supplier emails

Three emails reach a supplier (a `users` row in the `vendor` group) outside the document emails.
All three use the shared shell described in [../emails.md](../emails.md), and are branded with
the company the event belongs to.

## Prequalified categories

A supplier is prequalified for expense sub-types (`facility_expense_sub_type_vendors` rows with
`is_prequalified` true). Both emails below list them the same way, through
`App\Support\Mail\Supplier\PrequalifiedCategoriesTable`:

- a table with two columns, **Expense type** and **Category** (the sub-type), one row per
  prequalified sub-type, sorted by type and then category;
- rows with `is_prequalified` false are left out, and so are deleted sub-types;
- with a company, only that company's sub-types (and global ones with no company) are listed;
- properties are never listed, even when a row is limited to some properties.

### In the welcome email

When a supplier is onboarded, the welcome email's supplier section
(`App\Support\Mail\Welcome\SupplierWelcomeSection`) adds a "Your prequalified categories"
headline, one line saying they will be invited to quote for work in those categories, and the
table. A supplier with no prequalified categories gets no section at all.

### When the categories change

`App\Notifications\Vendor\PrequalifiedCategoriesChangedNotification`, queued, on the database and
mail channels (no SMS: the list does not fit).

**When.** `UpdateVendorAction` (`PUT app/{company}/users/vendors/{vendor}`), only when the request
carries `expense_sub_types`. The set of prequalified sub-type ids is read before and after the
rows are replaced, and the email goes only when the two sets differ. So:

| Request | Email |
|---|---|
| A category added, removed, or switched between prequalified and not | Sent |
| The same set again (other order, other properties or auto-assign settings) | Not sent |
| `expense_sub_types` left out | Not sent |
| `expense_sub_types: []` on a supplier who had categories | Sent ("no categories") |

It is sent through `OutboundEmail::send()`, so it waits for the update's transaction to commit.
Supplier creation, activation, termination and deletion send nothing from here: creation is
covered by the welcome email.

**Content.** Title `PREQUALIFICATION`; "Dear {name}"; a line saying their prequalified categories
with the company were updated; then either "You are now prequalified for the categories below"
and the full current table, or, when the set is now empty, a plain line saying they are not
prequalified for any category and will not be invited to quote until one is added. Button
**Open supplier portal** → `FrontendUrl::to('vendor.prequalification')`. A closing note says
whom to contact if a category is wrong.

**In-app payload.** `title`, `message`, `expense_sub_type_ids` (the current set) and the usual
`resource_type`/`resource_id`/`resource_url` (all null: there is no single resource).
`company_id` is the route's company.

## Bill rejected

`App\Notifications\PropertyManagement\Finance\BillInvoiceRejectedNotification`, sent by
`RejectBillInvoiceAction` when staff reject the invoice a supplier submitted against a pending
bill. Its channels are database and mail. It sent SMS and WhatsApp until 2026-10-06; a rejection
is not one of the messages allowed out by text (see [messaging.md](../messaging.md#which-notifications-text)).

The email:

- title `BILL REJECTED`, subject "Invoice rejected for bill #{id}";
- "Dear {name}" and a line saying the invoice for bill #{id} at {property} was rejected and
  removed from the bill, and must be corrected and submitted again;
- the rejection reason in a callout ("Reason for rejection");
- the bill's reference details: bill number, property, expense type, rejection date and bill
  total (in the bill's currency);
- button **Open bill** → `FrontendUrl::to('vendor.bill', ['id' => $bill->id])`, with a side note to
  upload the corrected invoice there;
- a note that payment is on hold until a valid invoice is submitted.

The invoice number and CU number are not shown: the rejection clears them from the bill before
the notification is built.

## Previews

Locally, `GET /_mail/prequalified-categories` renders the changed-categories email for the first
supplier with a prequalified category, and `GET /_mail/bill-rejected` the bill-rejected email for
the most recently rejected bill (or any bill with a supplier, with a sample reason).
