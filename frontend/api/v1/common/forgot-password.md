# Forgot & Reset Password

Base route:

`/api/v1/auth`

Guest endpoints: no token. Someone who forgot their password asks for a 6-digit code, receives it
by email (and by SMS when the account has a phone number), and sets a new password with it.

> **Frontend pages to build.** `/auth/forgot-password` and `/auth/reset-password` do not exist
> yet. The login page already links to a forgot-password page that 404s. Suggested flow: the
> forgot page takes the email or phone and posts `/auth/password/forgot`; on success it moves to
> the reset page carrying the same `login`, which asks for the code, the new password and its
> confirmation and posts `/auth/password/reset`; on success it goes to the login page.

## Endpoints

| Method | Path | What it does | Throttle |
|---|---|---|---|
| `POST` | `/auth/password/forgot` | Send a reset code | 5 requests a minute per client |
| `POST` | `/auth/password/reset` | Set a new password with the code | 10 requests a minute per client |

Past the throttle the answer is a `429` with a `Retry-After` header.

## Ask for a code

`POST /auth/password/forgot`

```json
{ "login": "jane@example.com" }
```

`login` is an email or a phone number, the same as the `username` on `POST /auth/login`
(`0712345678`, `712345678` and `+254712345678` all match the same phone).

**The answer is always the same**, whether or not the login belongs to an account, whether a code
was sent or held back by the cooldown:

```json
{ "data": {
    "message": "If that matches an account, we have sent a reset code to its email and phone. It expires in 10 minutes.",
    "expires_in_minutes": 10,
    "resend_after": 60 } }
```

- Show the message and move on to the reset page. Do not tell the person whether the account
  exists: the server does not know it either, from the client's side.
- `resend_after` is in seconds. Disable a "Send again" button for that long. Asking again sooner
  still returns `200` with the same body, but no new code is sent; the first code still works.
- At most 5 codes per account per hour are sent.
- A new code spends the previous one: only the latest code works.

| Status | Key | When |
|---|---|---|
| `422` | `login` | Missing, not a string, or over 255 characters |

## Reset the password

`POST /auth/password/reset`

```json
{ "login": "jane@example.com",
  "code": "482913",
  "password": "New-password-2026",
  "password_confirmation": "New-password-2026" }
```

- `login`: the same email or phone used to ask for the code.
- `code`: six digits, **as a string** (a number loses its leading zeros). The email shows it as
  `482 913`; strip the space before sending.
- `password`: the app's password rules (at least 8 characters), confirmed by
  `password_confirmation`.

Success:

```json
{ "data": { "message": "Your password has been reset. Sign in with your new password." } }
```

The person is **not** signed in: send them to the login page. Every token they had is deleted, so
any other device or tab that was signed in is signed out.

### Refusals

| Status | Key | When |
|---|---|---|
| `422` | `login` | Missing or over 255 characters |
| `422` | `code` | Not six digits |
| `422` | `code` | `That code is not valid. Check it, or ask for a new one.` — a wrong code, an expired one, one already used, one spent by too many wrong tries, or a login that matches no account |
| `422` | `password` | Missing, too short, or not confirmed |

The `code` refusal is deliberately one message for every reason, so it cannot be used to learn
which logins exist. A code is valid for 10 minutes and allows 5 wrong tries; after the fifth it is
spent and the person has to ask for a new one. Offer a "Send a new code" link next to the error.

```json
{ "message": "That code is not valid. Check it, or ask for a new one.",
  "errors": { "code": ["That code is not valid. Check it, or ask for a new one."] } }
```
