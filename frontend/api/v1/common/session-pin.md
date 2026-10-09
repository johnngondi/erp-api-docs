# Session PIN & Idle Lock

Base route:

`/api/v1/auth`

A session locks after a period without activity. Locked means **the API refuses** — `423` on
everything except the lock screen's own reads — so the lock is not something the client can be
talked out of.

The PIN is stored hashed and is never returned in any form. The only thing a client ever learns is
`has_pin`.

## Where the lock's settings come from

The server owns every number the lock screen needs. Three endpoints return them, in one shape, so a
client holds no defaults of its own:

| Endpoint | Returns |
| --- | --- |
| `POST /auth/login` | `session` — settings only |
| `GET /profile` | `session` — settings only |
| `GET /app/{company}/inbox/summary` | `session` — settings **plus** the live state |

```json
{ "session": { "idle_minutes": 15, "max_attempts": 3, "lockout_minutes": 15 } }
```

and on the summary, the same plus:

```json
{ "locked": false, "locks_in_seconds": 842 }
```

| Field | Meaning |
| --- | --- |
| `idle_minutes` | How long a session may sit idle before it locks |
| `max_attempts` | Wrong PINs allowed before a lockout — show this on the lock screen rather than waiting to learn it from a failure |
| `lockout_minutes` | How long a lockout lasts |
| `locked` | Whether this session is locked **now** |
| `locks_in_seconds` | Seconds left, or `null` when the server will never lock this session |

**Settings are on login and profile; live state is not.** At login there is no session to be idle
yet, and on a profile read a countdown is stale the moment it is read. Poll the summary for state.

**Tie your keep-alive interval to `idle_minutes`.** The floor is one minute, so a fixed two-minute
poll would lock a working user if the window were ever set below that. Half the window is a safe
rule.

**`retry_after` is always present on a `429`.** The status is chosen from the value itself, so a
lockout always carries it and a response without it is a `422` — an ordinary wrong PIN, not a
lockout. There is no need to assume a lockout length.

## Endpoints

| Method | Path | What it does |
|---|---|---|
| `POST` | `/auth/pin` | Set a PIN |
| `PUT` | `/auth/pin` | Change it (needs the current one) |
| `POST` | `/auth/pin/verify` | Unlock |
| `POST` | `/auth/pin/temporary` | Send a one-time PIN |
| `POST` | `/auth/pin/temporary/verify` | Unlock with it, clearing the old PIN |
| `POST` | `/auth/touch` | Keep-alive — "the person is still here" |
| `POST` | `/auth/lock` | Lock now, for someone walking away |

## has_pin

On the login response and on `GET /profile`, under `data.user`:

```json
{ "id": 1, "name": "Admin Strator", "has_pin": false }
```

`false` keeps the set-a-PIN screen up. There is no way to read the PIN itself.

## Set a PIN

`POST /auth/pin`

```json
{ "pin": "4728", "pin_confirmation": "4728" }
```

Four digits, confirmed. Returns `data.has_pin: true`.

Each refusal has its own key, so the key is the answer and the message is only there to display:

| Key | When |
|---|---|
| `pin` | Not four digits |
| `pin_confirmation` | The two do not match |
| `pin_too_obvious` | Every digit the same (`0000`), a run (`1234`, `6789`), a repeated pair (`1212`), or a keypad shape (`2580`) |

On `PUT`, a wrong current PIN is keyed `current_pin`.

## Change a PIN

`PUT /auth/pin`

```json
{ "current_pin": "4728", "pin": "5091", "pin_confirmation": "5091" }
```

`current_pin` is required: without it, anyone at an unlocked screen could set a PIN of their own
and lock the owner out of their own session. A wrong one is a `422` keyed `current_pin`.

## Unlock

`POST /auth/pin/verify` with `{ "pin": "4728" }`.

A wrong PIN is a `422` carrying two extra fields:

```json
{ "message": "That PIN is not right.",
  "errors": { "pin": ["That PIN is not right."] },
  "attempts_remaining": 2,
  "retry_after": null }
```

- `attempts_remaining` counts down, so you can show "2 tries left" without hardcoding a limit the
  server owns.
- `retry_after` is `null` until the lockout actually trips. Showing "try again in 15 minutes"
  after one typo would read as locked when it is not.

**On the last wrong try the response becomes a `429`**, keyed `pin_locked`, with `retry_after` in
seconds and a `Retry-After` header:

```json
{ "message": "Too many attempts. Use a temporary PIN to get back in.",
  "errors": { "pin_locked": ["Too many attempts. Use a temporary PIN to get back in."] },
  "attempts_remaining": 0,
  "retry_after": 900 }
```

A wrong PIN is a `422` like any other rejected input; the lockout is a `429` because it is not a
judgement about what was sent, it is a refusal to keep answering.

The lockout arrives *with* the final failure rather than on the next attempt, so you can go
straight to the temporary-PIN screen without another round trip.

While locked, even the right PIN is refused — otherwise the lockout is bypassed by guessing
correctly on the next try.

**Signing in again restores the three tries.** The counter is per person rather than per session,
so it outlives the session that earned it; without this, a fresh login inherited a spent counter
and the next single mistake locked on contact. A password is more than the PIN proves, so it puts
the count back. It does not unlock anything — the PIN still stands and a wrong one is still
refused.

## Temporary PIN

`POST /auth/pin/temporary`

Sends a one-time PIN on every channel the person can be reached on.

```json
{ "message": "Temporary PIN sent.",
  "channels": ["email", "sms", "whatsapp"],
  "phone": "•••• ••• 678",
  "email": "d•••••••••s@example.com",
  "expires_at": "2026-10-02T14:42:00+03:00",
  "resend_after": 60 }
```

`channels` lists where the PIN was queued. Sending happens on the `otp` queue just after the
response, so a provider that refuses the message is logged by the worker, not reported here. The
phone and email are masked — enough to recognise your own, not enough to learn someone else's.

Asking again inside the cooldown is a `422` keyed `temporary_pin_resend`. That exists so the
resend button cannot be used to spray someone's phone.

**When no channel takes it** (the person has neither an email nor a phone, or the PIN could not
be queued), the response is a `422` keyed `temporary_pin_send_failed` and no PIN is issued, so
there is nothing to type. Surface the message rather than offering a retry: the usual reason is
a missing address, not a transient failure:

```json
{ "message": "We could not send a temporary PIN. Check your phone number and email, or ask an administrator.",
  "errors": { "temporary_pin_send_failed": ["We could not send a temporary PIN. Check your phone number and email, or ask an administrator."] } }
```

### Using it

`POST /auth/pin/temporary/verify` with `{ "pin": "3334" }`.

```json
{ "message": "Unlocked. Set a new PIN to continue.", "has_pin": false }
```

Three things happen at once, all server-side:

1. The session unlocks.
2. The one-time PIN is spent — a second use is refused.
3. **The old PIN is cleared**, so `has_pin` comes back `false` and the set-a-PIN screen follows
   immediately. Whoever asked for this could not remember their PIN, so leaving it in place would
   lock them out again on the next idle screen.

| Key | When |
|---|---|
| `temporary_pin_expired` | Past its ten minutes |
| `temporary_pin_used` | Already spent — a different thing to tell someone |
| `pin` | Simply the wrong digits |

The ten-minute life and the single use are both the server's: a client clock decides nothing
here.

## The lock itself

Once a token has been idle past the window, every request outside the lists below returns **423**:

```json
{ "message": "Your session is locked. Enter your PIN to continue.",
  "locked": true,
  "errors": { "session_locked": ["Your session is locked. Enter your PIN to continue."] } }
```

### What still answers while locked

The lock screen has to draw itself and the person has to get back in:

- `GET /profile`
- `GET {portal}/inbox/summary`
- `GET {portal}/pending-tasks`
- `GET {portal}/notifications`
- `POST /auth/pin`, `/auth/pin/verify`, `/auth/pin/temporary`, `/auth/pin/temporary/verify`
- `POST /auth/logout`, `/auth/logout-all` — someone who cannot unlock must still be able to leave

### The meter reading app

The meter reading app has no lock screen, so the routes it calls are outside the lock altogether:
they answer whether or not the session is idle, and they **do not count as activity** either, so
using the meter app never keeps an idle main-app session open.

| Method | Path |
|---|---|
| `POST` | `/auth/login` (unauthenticated, so never locked) |
| `GET` | `/app/{company}/property-management/facilities` (index only) |
| `GET` | `/settings/lease-management/lease-components` (index only) |
| `GET` | `/settings/skus` (index only) |
| `GET` | `/app/{company}/property-management/facilities/utility-meters` and `/{meter}` |
| `POST` | `/settings/file-management/uploads` — the meter photo |
| `POST` | `/app/{company}/property-management/facilities/utility-meters/{meter}/extract-reading` |
| `GET`, `POST` | `/app/{company}/property-management/facilities/utility-meters/{meter}/readings` |
| `POST` | `/app/{company}/property-management/facilities/utility-meters/submit-readings` |

Only those. Viewing one property, creating, editing, deleting or (de)activating a meter, viewing,
editing or deleting a reading, writing lease components or units, and listing, viewing, editing or
deleting uploads are still refused with a `423` while locked.

The app needs one permission for all of it: `submit-meter-readings` (see
[utility-meters.md](../app/property-management/facilities/utility-meters.md#permissions)).

### What counts as activity

Everything **except** the lock screen's own reads and background polling. So ordinary work keeps
the session alive on its own and nobody is locked mid-task.

Polling is deliberately excluded. If `inbox/summary` reset the clock, an open tab would never go
idle and the feature would do nothing.

`POST /auth/touch` is the keep-alive for when someone is reading rather than clicking. It is the
only route that moves the clock without doing anything else, and it is **refused while locked** —
a locked tab that could touch its own session would free itself without the PIN. It is not a
poll: send it at most every couple of minutes, and only when there has been real activity since
the last one.

### Knowing when to lock

The server only sees requests, so a tab nobody is near sends nothing and is never told it has
been locked - until something it does send comes back `423`. The inbox summary closes that gap:
it is polled every thirty seconds, it answers behind the lock, and it does not count as
activity, so it carries the lock state on every portal:

```json
{ "session": { "locked": false, "locks_in_seconds": 212, "idle_minutes": 5 } }
```

- `locked: true` — put the lock screen up now. The next ordinary request would be refused anyway.
- `locks_in_seconds` — how long the server gives the session before it locks; `0` while locked.
  `null` means the server will never lock this session (enforcement off, or no PIN set), so
  there is nothing to wait for.
- `idle_minutes` — the window, for the "locked after N minutes" label.

A client may still lock on its own clock sooner (five minutes without a mouse or key is a fair
signal); the summary is the authority for the case where its clock is wrong, cleared, or never
started.

### Who never gets locked

Anyone with no PIN set. Locking them would be a closed door with no key; getting them to set one
is the client's job, driven by `has_pin`.

A session with no recorded activity is treated as fresh, so a cache flush does not lock everyone
out of the product at once.

## Locking on request

`POST /auth/lock`, with no body.

The idle window is for the times nobody decided anything. This is the other case — someone knows
they are walking away and should not wait five minutes for the screen to catch up. Put it behind
the profile dropdown.

```json
{ "message": "Locked.", "locked": true }
```

The next request is refused exactly as an idle one would be, and the same PIN unlocks it. Calling
it twice is harmless.

**It is refused when the caller has no PIN**, keyed `pin_required` — locking someone who cannot
unlock would strand them in their own session. Only offer the button when `has_pin` is `true`.

## Switching it on

The lock ships **off**. `SESSION_PIN_ENFORCE` turns it on.

With it off, every PIN endpoint still works — a PIN can be set, changed, verified — and nothing is
ever refused with a `423`. So the backend can merge and deploy before the lock screen exists,
without anyone being locked out of a product that has nothing to show them.

Turning it on is an environment change, not a deploy. So is turning it off again, which matters
more.

## Settings

All server-side, and all environment variables, so they change without a deploy:

| Variable | Default | |
|---|---|---|
| `SESSION_PIN_ENFORCE` | `false` | Whether idle sessions are refused at all |
| `SESSION_PIN_IDLE_MINUTES` | 5 | Idle time before locking |
| `SESSION_PIN_MAX_ATTEMPTS` | 3 | Wrong tries before lockout |
| `SESSION_PIN_LOCKOUT_MINUTES` | 15 | How long the lockout lasts |
| `SESSION_PIN_TEMPORARY_MINUTES` | 10 | Life of a one-time PIN |
| `SESSION_PIN_RESEND_AFTER_SECONDS` | 60 | Resend cooldown |

Read `attempts_remaining` from the response rather than hardcoding the limit.

## Frontend Error Handling

Apply shared rules in `docs/frontend/app/README.md`.
