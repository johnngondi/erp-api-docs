# Backend Developer Take-Home Assessment

**Stack:** Laravel 10 · PHP 8.1 · Pest · `spatie/laravel-data` v4 · `spatie/laravel-query-builder` v5.8 · `spatie/laravel-permission` v6

**Time budget:** ~4–6 focused hours. We are evaluating *how you tie our core building blocks together*, not how much you build. A small, clean, idiomatic slice beats a large, messy one.
Delivery Due: 30/06/2026 at 18:00 HRS (EAT)

> You will build this in a **fresh Laravel project** of your own. We are not handing you our codebase — the point is to see whether you can structure a feature out of the patterns described below from scratch. Set up the listed packages yourself.

---

## 1. The scenario

You are building a small slice of an **events platform**. An *organiser* (the tenant) runs several *events*, and each event sells *tickets* in named **ticket tiers** (e.g. "Early Bird", "VIP", "General Admission").

You will build the **persistence + API slice** for ticket tiers: model, migration, the full CRUD endpoint, plus one custom action. No frontend, no payment processing, no PDF — just a clean, idiomatic, multi-tenant API resource.

The whole point is the *structure*: we want to see you wire a Laravel Data class, an Action class, a Spatie Query Builder index, an API Resource, a policy, and tenancy scoping into one coherent feature the way a senior Laravel dev would.

---

## 2. The concepts you must demonstrate

These are the patterns we live by. Your submission is judged on getting each one right and making them work *together*.

| Concept | What we expect to see |
|---|---|
| **Laravel Data class** (input + validation) | A `CreateTicketTierData` and `UpdateTicketTierData`. Validation declared in a `rules(ValidationContext $context)` method. Use `Optional` for non-required fields. Tenant-scoped `exists` / `unique` rules must resolve the organiser from the **route or authenticated context — never from the request body**. |
| **Action class** | One `Create`, one `Update`, one `Delete`, and one `Publish` action. Each has exactly **one public method `execute(...)`**; every other method is `private`. The action owns the write logic and returns the model (or `void`). Controllers must not contain write logic. |
| **API Resource** (`toArray` envelope) | A `TicketTierResource`. Always include `id`; expose every other field via `whenHas`; expose relations via `whenLoaded`. No business logic beyond shaping output. |
| **Spatie Query Builder** (index) | `QueryBuilder::for(...)` with `allowedFilters`, `allowedSorts`, `allowedIncludes`, a default `latest()` ordering, and pagination via a `per_page` query param with a sane default. |
| **Controller conventions** | An `--api` resource controller (no `create`/`edit`). Call `$this->authorize(...)` first. Wrap every write in a `DB::beginTransaction()` / `try` / `catch` block — re-throw `ValidationException`, otherwise log the exception and throw a single generic, translated message. Return a consistent response envelope (see below). |
| **Consistent response envelope** | Mutating endpoints return a uniform JSON envelope carrying a translated `message` and the resource payload — define one small wrapper Resource and use it everywhere, rather than ad-hoc `response()->json([...])`. |
| **Multi-tenancy** | An `organiser_id` boundary applied via a **global scope (an Eloquent trait)** so queries are tenant-bound by default. `organiser_id` is set server-side on create, never accepted from the client. One organiser must never be able to read or mutate another's rows. |
| **Authorization** | A `TicketTierPolicy` with `viewAny`/`view`/`create`/`update`/`delete` backed by `spatie/laravel-permission` (`$user->hasPermissionTo('...')`). State any permission-name assumption you make. |
| **Routing** | `Route::apiResource(...)` nested/scoped under the tenant, plus one non-resourceful route for the `publish` action. |
| **Pest feature tests** | `tests/Feature` Pest tests with a `beforeEach` that bootstraps a user + organiser + permissions. Assert HTTP status, DB state, **and** response payload. |
| **Localization** | Every user-facing string wrapped in `__()`. |

---

## 3. Scope — exactly what to implement

### 3.1 Migration + model

Create `ticket_tiers`:

- `id`
- `event_id` (FK)
- `organiser_id` (FK, **not nullable**) — the tenant boundary
- `name` (string)
- `price` (decimal) — money, stored in currency units
- `quantity` (unsigned int) — total tickets available in this tier
- `sales_channels` (json, **nullable**) — `NULL` = sold on all channels; a JSON array (e.g. `["web", "box_office"]`) = only those channels
- `is_published` (boolean, default false)
- `is_active` (boolean, default true)
- timestamps + soft deletes

Index: composite `(organiser_id, event_id, is_active)`.

`TicketTier` model:
- Apply your tenancy trait/global scope (`organiser_id`).
- Cast `sales_channels` to array; cast `price` appropriately.
- Query scopes: `forEvent($eventId)`, `active()`, `availableOnChannel($channel)` (matches `sales_channels IS NULL` **OR** the JSON array contains `$channel`).

### 3.2 The CRUD endpoint

A tenant-scoped `apiResource` (`index`, `store`, `show`, `update`, `destroy`) **plus** a `publish` action route. Build it with the full convention stack from Section 2:

- `index` via Query Builder — filterable by `event_id` and by `channel` (the latter via an `AllowedFilter::callback` using `availableOnChannel`), sortable by `name`/`price`/`created_at`, paginated.
- `store` / `update` via Data classes + Action classes inside a DB transaction, returning the response envelope.
- `destroy` is a soft delete.
- `publish` flips `is_published` to `true` — single-action style.

**Validation rules that matter (put them in the Data class):**
- `name` must be unique **per event** (scoped to the tenant — not globally).
- `event_id` must reference an event that **belongs to the current organiser**.
- `price` ≥ 0; `quantity` ≥ 1.
- `sales_channels` nullable; when present, an array whose values are all within a fixed allowed set.
- `organiser_id` is set server-side — reject or ignore it from the request body.

### 3.3 Tests (Pest, `tests/Feature`)

Cover at minimum:
1. Store creates a tier scoped to the current organiser; a client-supplied `organiser_id` cannot override it.
2. Creating a tier whose `event_id` belongs to another organiser is rejected.
3. `name` uniqueness is enforced per event, but the same name is allowed across two different events.
4. `availableOnChannel` returns all-channel (`NULL`) + matching-channel tiers and excludes a tier restricted to a different channel.
5. A user from organiser B cannot read or update organiser A's tier (tenancy boundary).

---

## 4. Ground rules

- **Demonstrate the patterns working together** — that's the whole exercise. We are reading how the Data class, Action, Resource, Query Builder, policy, and tenancy scope hand off to each other.
- **No frontend, no payments.** `sales_channels` and `price` are simple stored values here.
- Keep `git` history clean and incremental — we read your commits.
- If a requirement is ambiguous, make a reasonable assumption, **state it in your README**, and proceed.
- Run the suite before submitting: `php artisan test`.

---

## 5. How we score (100 pts)

| Area | Pts | What earns the marks |
|---|---|---|
| **Data classes & validation** | 20 | Correct `spatie/laravel-data` v4 usage; `Optional`; server-resolved tenant scoping; no client-trusted `organiser_id`; per-event unique rule done right. |
| **Action classes** | 15 | Single `execute`, private helpers, write logic lives here — not in the controller. |
| **Controller & envelope** | 15 | `--api`, authorize-first, transactions + correct exception handling, consistent response envelope. |
| **Query Builder index** | 10 | Proper `allowedFilters`/`allowedSorts`/`allowedIncludes`, pagination + `latest()`, the `availableOnChannel` filter. |
| **Resource** | 10 | `id` + `whenHas` + `whenLoaded`, no leakage of internals. |
| **Tenancy correctness** | 15 | Global scope applied correctly; cross-tenant access impossible. |
| **Tests** | 10 | Meaningful assertions on status, DB, and payload; all five scenarios covered and passing. |
| **Conventions & polish** | 5 | `__()` everywhere, sensible exception logging, routing placement, policy, clean commits, README with assumptions. |

**Automatic red flags:** trusting a client-supplied `organiser_id`; business logic in the controller instead of an Action; ad-hoc `response()->json` instead of a consistent envelope; tenancy enforced by hand in each query instead of a global scope; missing transactions on writes; untranslated user-facing strings; global-instead-of-per-event uniqueness.

---

## 6. Deliverable

A git repo (or zip) containing your migration, model, Data classes, Action classes, controller, Resource, policy, routes, and tests — plus a short `README` listing your assumptions and how to run the tests and postman collection to test api. Good luck.

----

## 7. NOTE

We encourage use of AI coding agents like claude and codex as we believe that is the future of software development and we expect you to use them in the project.
Neverthe less, we expect you to be able to defend the architectural choices you made and explain why specific lines of code or decisions made came to be.
