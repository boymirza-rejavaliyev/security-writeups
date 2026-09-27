# WordPress Plugin Vulnerability Research — Methodology

How every finding in this folder was produced, end to end.

## 1. Sourcing targets

All targets came from the **official WordPress.org plugin repository** — no private or leaked
code. Plugins were pulled in batches using WordPress.org's own plugin API (tag/category and
"popular plugins" listings, e.g. plugins tagged `booking`, `file-manager`, `backup`, `csv-import`),
then downloaded as the exact `.zip` a real site would install from
`https://downloads.wordpress.org/plugin/<slug>.<version>.zip`.

Plugins were prioritized by two signals:
- **Active install count** — enough real-world exposure to matter, but this research
  focused on small-to-mid plugins (4,000–20,000 installs) rather than the giants, since that's
  where less-audited code tends to live.
- **Suspicious surface area at a glance** — a function name like `nd_booking_final_price_php`, a
  REST route with no input-validation schema next to siblings that have one, or a route registered
  with `permission_callback => '__return_true'` are all signals worth a closer look, not proof of a
  bug by themselves.

## 2. Static analysis first

Each plugin's PHP source was read manually, tracing every `wp_ajax_*` / `wp_ajax_nopriv_*` action
and every `register_rest_route()` call end to end: where does the input come from, what checks run
on it, and — critically — **where does the result get used or saved**. A lot of "looks handled"
code turns out not to be once you actually follow a value from `$_POST` to its final sink.

## 3. Live verification before writing anything up

Every finding below was reproduced against a real, running instance — WordPress + (where relevant)
WooCommerce, in an isolated local Docker container — never against a live production site. A couple
of these took several attempts to get the PoC payload exactly right; those false starts are kept in
each write-up because they were part of how the bug was actually confirmed, not polished away.

## 4. Re-verify against the latest release before reporting

The single most useful habit in this whole process: **before submitting anything**, pull the
current stable release again and re-check the vulnerable code path still exists. This is what
caught the BookIt finding already being patched upstream — see [bookit.md](bookit.md).

## Findings in this folder

| Write-up | What it covers |
|---|---|
| [bookit.md](bookit.md) | Unauthenticated Stripe payment amount manipulation — found, but already fixed upstream by disclosure time |
| [nd-booking.md](nd-booking.md) | Unauthenticated, persistent WooCommerce price override |
| [booking-package.md](booking-package.md) | Predictable booking-cancellation token (weak randomness) |
| [media-library-organizer.md](media-library-organizer.md) | Authenticated path traversal → arbitrary `.zip` file write |
