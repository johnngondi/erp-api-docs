# Activity log — the backend contract

**Audience: backend developers.** This is an internal contract, not an API endpoint. The audit
trail report (14.5.5) is built on top of what this describes; nothing here is exposed to the
frontend yet.

Every model in `app/Models` is audited. A create, an update or a delete on any of them writes a
row to `activity_log` carrying:

| Column | What it holds |
|---|---|
| `description` | The sentence a person reads. Never a dump of attributes. |
| `properties` | The raw `attributes` / `old` values, for anyone who needs the exact before and after. |
| `subject_type` / `subject_id` | The record that changed. |
| `causer_type` / `causer_id` | The person responsible — including for work done on a queue. |
| `event` | `created` \| `updated` \| `deleted` \| `restored`. |
| `log_name` | The kind of record, slugged from its label (`invoice`, `property`, …). |
| `resource_url` | Where to send a reader who wants to see the record. **Relative.** |

The package is [`spatie/laravel-activitylog`](https://spatie.be/docs/laravel-activitylog). Nothing
below replaces its documentation — it covers the parts this codebase does differently.

---

## How a model becomes auditable

It already is. `App\Models\Concerns\Auditable` is applied to every model in `app/Models`, and a new
model should carry it too:

```php
use App\Models\Concerns\Auditable;

class FacilityInvoice extends Model
{
    use Auditable;
}
```

That is the whole change. The trait supplies `getActivitylogOptions()`, so a model never writes
one, and a test (`ActivityLogFoundationTest`) fails if a model in `app/Models` is missing it.

**Nothing else is required.** A model with no configuration logs with a conventional label (its
class name, headlined), a conventional title (the first of `cu_invoice_number`, `invoice_number`,
`name`, `title`, … that it carries), conventional wording, and no resource URL.

Everything that cannot be derived from the model lives in **`config/audit.php`**:

```php
App\Models\FacilityInvoice::class => [
    'label' => 'Invoice',                                   // what to call it in a sentence
    'title' => 'cu_invoice_number',                         // what names one record
    'url' => 'lease-management/billing/invoices/{id}',      // its page, relative
    'describer' => FacilityInvoiceDescriber::class,         // its wording
],
```

To stop a model logging entirely, add it to `audit.ignore`. That list is enforced at runtime, so a
model keeps the trait and simply stays quiet — the activity model itself is there, because logging
the log is a loop.

---

## Writing a description resolver

**The description is produced by a service, never by a closure on the model.** Two reasons: the
wording is the part most often argued over and changed, and a closure inside a model cannot be
tested without saving a record.

A describer implements `App\Services\Audit\Contracts\DescribesActivity`:

```php
class FacilityInvoiceDescriber implements DescribesActivity
{
    public function __construct(private readonly DefaultActivityDescriber $default) {}

    public function describe(ActivityContext $context): string
    {
        if ($context->isUpdated() && $context->changed('status')) {
            return "Marked {$context->subjectName()} as {$context->readable($context->to('status'))}";
        }

        return $this->default->describe($context);   // fall back for everything else
    }
}
```

Register it under the model in `config/audit.php`. A model without one gets
`DefaultActivityDescriber`, which produces:

```
Created Invoice INV-0012
Updated Property Westlands Plaza: name from Westlands Tower to Westlands Plaza
Deleted Invoice INV-0012
```

### What the `ActivityContext` gives you

Everything is resolved before the describer runs, so a describer never reaches into a mid-event
model:

| Member | What it is |
|---|---|
| `subject` | The model that changed. |
| `event` | `created` \| `updated` \| `deleted` \| `restored`, with `isCreated()` / `isUpdated()` / `isDeleted()` helpers. |
| `label` / `title` / `subjectName()` | "Invoice", "INV-0012", "Invoice INV-0012". |
| `changes` | `attribute => ['from' => …, 'to' => …]`, with ignored and hidden columns already dropped. Empty for anything but an update. |
| `changed($attr)`, `from($attr)`, `to($attr)`, `only(…)` | Reading one change. |
| `phrase($attr)` | One change as prose: `"status from draft to approved"`. |
| `attributeName($attr)` | `facility_id` → "facility". |
| `readable($value)` | `"1200.00000"` → `1,200`; `true` → `yes`; an enum → its value; a date → a readable date; `null` → `null`. |

### Rules

- **Return a sentence, not a dump.** The raw values are already in `properties`.
- **Name the record as it is now.** A renamed property is described by its new name; the old one is
  in the change phrase.
- **Keep it short.** The default names at most four changes and summarises the rest as "and N
  more"; do the same.
- **Never query in a loop.** A describer runs inside a model event, on every save.

Describers are tested directly — build a model with `make()`, call
`app(ActivityDescriber::class)->describe($model, 'updated')`, assert the string. No database
needed.

---

## Parent and child

**Where a child changes, the child's change is recorded and the parent is touched.** Editing an
invoice line writes an activity for the line and bumps the invoice's `updated_at`, so the invoice
tells the truth about work done underneath it.

A child names its owner in one of two places — the property wins, so the fact can sit beside the
relation it names:

```php
// config/audit.php
App\Models\FacilityInvoiceItem::class => ['parent' => 'invoice'],

// or on the model
protected $auditParent = 'invoice';
```

Two things follow from that one declaration:

1. **The parent is touched** on every save and delete of the child. `touch()` is itself a save, so
   the chain continues: a line on a line reaches the record that owns both. A cycle in the
   configuration is guarded against rather than recursed into.
2. **The child's activity links to the parent.** See below.

Laravel's own `$touches` does the first half for models that declare it; this covers the rest from
the audit declaration so there is one place to look.

> A pure touch does not write an activity of its own. `updated_at` is in `audit.ignore_attributes`
> and the options use `logOnlyDirty()` with `dontSubmitEmptyLogs()`, so the parent's timestamp moves
> without adding a row that says nothing. The child's row is the news.

---

## The resource URL

`resource_url` is where the audit trail sends a reader who wants to see the record. Two rules:

**1. A child resolves to its parent.** An invoice item has no page — its change happened on the
invoice's page — so its row stores the invoice's URL. The walk continues up the ownership chain and
stops at the first ancestor that has a page. A record with no page, and no ancestor with one,
stores `null`; that is a valid answer, and the row still records what happened.

**2. It is relative.** No scheme, no host, no `/api/v1/app/{company}` prefix, and no leading slash —
a path fragment the client joins to its own base. The same row is read through whatever host the
app is served from, by a client that already knows which company it is in.

```
lease-management/billing/invoices/42     ✅
/lease-management/billing/invoices/42    ❌ leading slash
https://app.example.com/invoices/42      ❌ absolute
api/v1/app/3/…                           ❌ carries the API prefix and a company
```

Patterns live in `config/audit.php` as `url`, with `{attribute}` placeholders filled from the
record (`{id}` is the primary key). A placeholder that resolves to nothing voids the URL rather
than storing a half-built path.

> **Not the `relative_url` column.** Several tables carry a `relative_url` column of their own; that
> is the path to the record in the **legacy EPMAS system** (see
> `App\Services\Migration\Epmas\Support\LegacyUrl`) and has nothing to do with this.

---

## Queue causer attribution

A queue worker is authenticated as nobody. Without intervention, everything a job changes is logged
against `null`: invoices generated, budgets monitored, statements rebuilt, all apparently by no
one — and an audit trail that cannot name a person is not much of an audit trail.

So the user is captured **when the job is queued** and restored **while it runs**. Both halves live
in `App\Providers\AuditServiceProvider`:

1. `Queue::createPayloadUsing()` stamps the signed-in user onto every job payload at dispatch, under
   the `audit_causer` key.
2. A `JobProcessing` listener reads it back and hands it to spatie's `CauserResolver`; the
   completion listeners (`JobProcessed`, `JobFailed`, `JobExceptionOccurred`) clear it, so a worker
   running one job after another never attributes one person's work to the next.

**There is nothing to do when writing a job.** Attribution is wired at the queue rather than in a
base class, so every job gets it — including ones dispatched by packages — and a new job needs no
trait, no constructor argument and no remembering.

```php
// This is a complete, correctly-attributed job.
class RenameFacilityJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(private readonly int $facilityId, private readonly string $name) {}

    public function handle(): void
    {
        Facility::find($this->facilityId)?->update(['name' => $this->name]);
    }
}
```

A job dispatched by the scheduler or a console command has no user to capture, and its activities
are logged with no causer. That is the honest answer, not a bug — do not invent a system user for
it.

A stamp naming a user who has since been deleted resolves to `null` rather than failing the job.

---

## Volume

Every model being auditable means this table grows faster than any other in the schema. Four
indexes are in place from the first migration, because adding them later is an `ALTER` on a table
already holding millions of rows:

| Index | Columns |
|---|---|
| `activity_log_causer_created_index` | `causer_id`, `causer_type`, `created_at` |
| `activity_log_subject_type_created_index` | `subject_type`, `created_at` |
| `activity_log_event_created_index` | `event`, `created_at` |
| `activity_log_created_at_index` | `created_at` |

Each leads with the column the audit endpoint filters on and carries `created_at` behind it: the
filter narrows, the index supplies the newest-first order without a sort.

The package's own retention command deletes rows older than
`config('activitylog.delete_records_older_than_days')` (365). Schedule
`activitylog:clean` before this table becomes a problem.

---

## Turning it off

`ACTIVITY_LOGGER_ENABLED=false` stops all logging, including the parent touching. Useful for bulk
imports and data migrations, where a per-row audit trail is noise and the volume is real. Prefer
spatie's `activity()->withoutLogs(fn () => …)` for a single operation.
