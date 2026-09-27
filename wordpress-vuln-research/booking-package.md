# Booking Package (WordPress plugin) ≤ 1.7.28 — Predictable Cancellation Token → Unauthenticated Booking Cancellation

| | |
|---|---|
| **Target** | Booking Package WordPress plugin, v1.7.28 (latest at time of testing) |
| **Vulnerability class** | CWE-330 (Use of Insufficiently Random Values) |
| **Access required** | None — unauthenticated |
| **Status** | Confirmed via isolated cryptographic PoC. **Reported to vendor. No CVE was issued** — this bug class doesn't cleanly match Patchstack's "significant/sensitive object" bar for broken access control, and the plugin's ~10,000 installs sit below Wordfence's 50,000-install threshold for this bug category. |

> Responsible disclosure notice: tested in an isolated local Docker WordPress environment only.
> The vendor was notified before this write-up was published.

## How I found this

This plugin came from the same WordPress.org batch as several other booking/reservation plugins. A
quick scan flagged one REST route registered with `permission_callback => '__return_true'` — always
worth a second look. It turned out to be fine on its own: the route genuinely serves an anonymous,
public booking form, and every internal `mode` it dispatches to has its own token/nonce check.

That led me to enumerate every `mode` inside the public dispatcher function
(`requestAjaxFrontEnd()`) — `sendBooking`, `cancelBookingData`, `deleteUser`, `updateUser`, and
others. `cancelBookingData` stood out because it relies on a `(key, token)` pair — the classic
"secret token" pattern, and if the token generation itself is weak, the whole access control
collapses regardless of how the check is written.

I confirmed the token was checked correctly against the database (a proper prepared statement — no
SQL injection there), which meant the actual weakness had to be in **how the token is generated**.
That led to `insertPrivateData()`:

```php
$cancellationToken = hash('ripemd160', $timeKey . $scheduleUnixTime . microtime(true));
```

This is the kind of code that looks safe at a glance — a hash function, three inputs, reasonably
long output. The real question is never "is a hash function used," it's "are all the inputs to it
actually secret." Tracing where `$timeKey` comes from led straight to `$_POST['timeKey']` — a
client-supplied value used inside a security token, which is an immediate red flag. Checking the
plugin's own front-end JS (`js/Booking_app.js`) confirmed it: `timeKey` is always set to
`schedule.key`, the calendar time-slot's own database key — visible to every visitor browsing the
public calendar. `$scheduleUnixTime` is equally public. Only `microtime(true)` — the exact
millisecond the booking was created — was ever genuinely unknown.

To prove this concretely rather than just asserting it, I ran the exact vulnerable formula (copied
line-for-line from the plugin) in its own PHP runtime inside the Docker container, simulating an
attacker who only has the two public values and a realistic few-second window on the timing —
recovering the full token by brute force in well under a second.

## Summary

Booking Package lets an anonymous customer cancel their own booking using a `cancellationToken`.
The token is meant to prove "this person made this booking" without requiring an account — but the
formula that generates it leans on two values that are **public**, plus one that's only
moderate-entropy:

```php
// lib/Schedule.php:11328
$cancellationToken = hash('ripemd160', $timeKey . $scheduleUnixTime . microtime(true));
```

- `$timeKey` — taken from client-supplied `$_POST['timeKey']`, and per the plugin's own front-end
  JS (`js/Booking_app.js`), it's always set to `schedule.key`: the calendar time-slot's own database
  key, visible to *any* visitor browsing the public booking calendar.
- `$scheduleUnixTime` — the exact date/time of the booked slot, likewise public on the calendar.
- `microtime(true)` — the only genuinely unknown input, a floating-point timestamp with microsecond
  resolution, captured at the instant the server processed the request.

Because the two "secret" inputs aren't secret at all, and the timing window can be narrowed to a
few seconds (e.g. by watching the live calendar for a slot becoming unavailable, or from a
response's `Date` header), the entire token can be brute-forced **offline, locally, against no
server at all** — RIPEMD-160 is fast and has no memory-hardness to slow a guesser down.

This token, together with the booking's own sequential/enumerable database ID, is the *only*
authorization check on the unauthenticated `cancelBookingData` endpoint. Recovering it lets an
attacker cancel any other customer's booking.

## Proof of Concept

To validate the core weakness without needing to drive the full multi-step booking/payment UI, the
exact vulnerable formula from `lib/Schedule.php:11328` was run in the plugin's own PHP runtime
(WordPress Docker container, plugin v1.7.28), simulating a realistic attacker who knows only the
public `timeKey` / `scheduleUnixTime` and a 4-second creation-time window:

```
=== ATTACKER SIDE (only public timeKey/scheduleUnixTime + a few seconds' window) ===
Search space: 4 seconds x 1,000,000 microseconds = 4,000,000 candidates
MATCH FOUND after 636,451 attempts in 0.48s
Recovered token matches real token exactly: 35e36c39c9d821460d72e4b2b84491f58940c995
```

The exact `cancellationToken` was recovered in under half a second using a plain PHP script — no
GPU, no specialized hardware, because RIPEMD-160 is computationally cheap.

## Impact

- Any unauthenticated attacker who can approximate *when* a booking was made — a realistic
  condition given the plugin's own public, real-time calendar — can cancel any other customer's
  booking.
- For a business relying on this plugin (restaurant, clinic, rental, etc.), this enables systematic,
  repeatable sabotage of other people's bookings with zero authentication and zero interaction from
  the legitimate customer.

## Suggested fix

Generate the token from a cryptographically secure random source that has no dependency on any
client-supplied or externally observable input:

```php
$cancellationToken = bin2hex(random_bytes(32));
```

Store and require it exactly as before, but make sure nothing derived from `$_POST` (or anything
else visible to a site visitor) ever contributes to its value.

## Why this didn't get a CVE

Patchstack's accepted "Broken access control" category requires the access gained to reach a
significant or sensitive object — the examples given are things like API keys, password hashes, or
backup/SQL files. An unauthenticated booking cancellation doesn't map cleanly onto that list, even
though it's a genuine, exploitable authorization failure with real business impact. Wordfence's
program would fold this into its general "All Other Vulnerabilities" bucket, which requires
≥50,000 active installs for a new researcher; this plugin sits at roughly 10,000. Both are triage
thresholds set by the vendors, not a verdict on whether the bug is real — it was reported to the
plugin author directly regardless.
