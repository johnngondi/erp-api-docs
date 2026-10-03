# Code emails

Two emails carry a one-time code. Both follow the OTP design in
`resources/views/mail/_design/reference.html`: a greeting and an intro, the code on a `code` card
with "Valid until", a "Didn't request this?" `callout`, and a closing `note` that staff never ask
for the code. Both are platform-branded (`EmailMessage::for(null)`, `config('app.name')`): they
are about the person's account, not a company's.

Neither is ever on the database channel: a code in the in-app list would be readable by anyone
looking at the screen it protects. Both are sent synchronously, one channel per notification
instance, so a broken SMS gateway does not take the email down with it. If either is ever
queued, it must also implement `Illuminate\Contracts\Queue\ShouldBeEncrypted`.

| | Temporary session PIN | Password reset code |
|---|---|---|
| Notification | `App\Notifications\Auth\TemporaryPinNotification` | `App\Notifications\Auth\PasswordResetCodeNotification` |
| Sent by | `SessionPinService::issueTemporary()` (`POST auth/pin/temporary`) | `PasswordResetCodeService::issue()` (`POST auth/password/forgot`) |
| Channels | email, SMS, WhatsApp | email, SMS |
| Title | `VERIFICATION CODE` | `PASSWORD RESET` |
| Code card label | Temporary PIN | Reset code |
| Code shown as | `5091` | `482 913` (the SMS carries `482913`) |
| Preview | `/_mail/pin-code` | `/_mail/password-reset-code` |

The session PIN flow itself is documented in `docs/frontend/api/v1/common/session-pin.md`; only
its email changed here.

## Forgot password (API)

Guest routes in `routes/Api/v1/auth-password.php`, required inside the `auth` group of
`routes/Api/v1/common.php`. Frontend contract: `docs/frontend/api/v1/common/forgot-password.md`.

| Route | Controller | Action |
|---|---|---|
| `POST api/v1/auth/password/forgot` (`throttle:5,1`) | `Auth\Password\ForgotPasswordController` | `RequestPasswordResetCodeAction` |
| `POST api/v1/auth/password/reset` (`throttle:10,1`) | `Auth\Password\ResetPasswordController` | `ResetPasswordWithCodeAction` |

Input is `RequestPasswordResetCodeData` (`login`) and `ResetPasswordWithCodeData` (`login`,
`code`, `password` with `password_confirmation`; the password rules come from
`App\Actions\Fortify\PasswordValidationRules`).

### No enumeration

- **Forgot answers the same for every login**, and answers first. The controller validates the
  input, registers a terminating callback and returns. The callback, which runs after the
  response has been sent (under PHP-FPM the connection is already closed), looks the login up,
  issues the code and sends it. So neither the body nor the response time depends on whether the
  account exists. A failure in the callback is logged with `log_exception` and nobody is told.
- The cooldown, the hourly cap and an unreachable account all return quietly from `issue()`,
  rather than throwing as the session PIN's resend does.
- **Reset refuses with one error**, keyed `code`, for a wrong, expired, used or exhausted code
  and for an unknown login. When there is no code to check (no account, no live code), the
  service checks the input against a decoy hash, so that path costs the same bcrypt check as a
  wrong code.

### The code

`user_password_reset_codes`, model `UserPasswordResetCode` (scope `usable()`: not used, not
expired). The service is modelled on `SessionPinService`:

- six digits from `random_int`, stored with `Hash::make`, never readable;
- issuing spends every outstanding code of the account first, so only the latest works;
- `channels` records what went out (`email`, `sms`); when nothing got through, the code is spent
  at once;
- the account is looked up exactly as `LoginUserAction` does: `email`, or `phone` through
  `format_phone_number()`.

Limits, in `config/password_reset.php` (env-tunable):

| Key | Default | Meaning |
|---|---|---|
| `minutes` | 10 | How long a code works |
| `max_attempts` | 5 | Wrong guesses before the code is spent |
| `resend_after_seconds` | 60 | Cooldown between codes for one account (`password-reset:resend:{id}`) |
| `max_per_hour` | 5 | Codes per account per hour (`password-reset:hourly:{id}`) |

Attempts are counted on the row (`attempts`), not in the cache. The reset controller runs the
action in a transaction and locks the code row (`lockForUpdate`), so parallel guesses cannot race
past the limit. **On a validation refusal it commits rather than rolls back**: the only writes
before a refusal are the attempt count and spending an exhausted code, and rolling those back
would allow unlimited guesses. The password is never changed before a refusal.

### On success

The password is set with `Hash::make`, the code (and any other outstanding one) is spent, and
every Sanctum token of the user is deleted, signing them out everywhere. The response carries a
message only; the person signs in again with the new password.

### Tests

`tests/Feature/Emails/PasswordResetCodeTest.php`. The test client terminates the application
after every request, so the terminating callback has run by the time a request returns. The
controller guards its callback to run once, because the application keeps its terminating
callbacks and a test that makes two requests terminates it twice.
