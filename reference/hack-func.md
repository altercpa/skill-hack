# Global Functions

These helpers are global functions, always loaded (including the "back compatibility" polyfills at
its end). They are plain functions, **not** methods — call them directly anywhere,
including in standalone scripts that run without `$core`. Prefer them over hand-rolled
equivalents so hack code stays indistinguishable from platform code (see
[Code Style](hack-style.md)).

## JSON / Serialisation

Use these instead of raw `json_*` — they handle unicode, zero-fraction preservation and
the legacy `serialize`/`null` formats stored in older columns.

| Function | Purpose |
|----------|---------|
| `xxenc( $data, $pretty = false )` | Encode to JSON (unescaped unicode, preserves `.0`). |
| `xxend( $data, $pretty = false )` | `xxenc` + `addslashes` — ready for a SQL literal. |
| `xxdec( $data )` | Decode a JSON string to an array (tolerates `null`/empty input). |
| `xxload( $path )` / `xxsave( $path, $data )` | Load / save a JSON file. |
| `xxvar( $data )` | Pretty `var_export` with short-array syntax (used for generated PHP). |

```php
$core->db->add( DB_XPROXY_ADDR, [ 'proxy_meta' => xxend( $data ) ] );
$data = xxdec( $row['proxy_meta'] );
```

## IP Addresses

| Function | Purpose |
|----------|---------|
| `remoteip( $server )` | Real client IP (CF / X-Real-IP / XFF / `REMOTE_ADDR`). |
| `ip2bin` / `bin2ip` | Packed binary ⇄ IP string. |
| `ip2hex` / `hex2ip` | Hex ⇄ IP. |
| `ip2hql( $ip )` | `X'<hex>'` binary SQL literal for an IP column. |
| `ip2crc( $ip )` | uint32 hash of an IP (for indexed lookups). |
| `ip2int` / `int2ip` | uint32 ⇄ IPv4. |
| `goodip` / `goodip4` / `goodip6` | Validate any / v4 / v6 address. |
| `publicip` / `privateip` | Routable vs. private/reserved. |
| `strictip( $ip )` | Strictly public IPv4: `goodip4()` and `publicip()`, and also outside 100.64.0.0/10, 198.18.0.0/15 and 192.0.0.0/24, which `publicip()` accepts. `false` for IPv6. |
| `ishttps( $server )` | Is the request over HTTPS? |
| `botip( $ip )` / `bothost( $host )` | Heuristic datacenter/bot detection. |

```php
$ip = remoteip( $core->server );
$core->db->add( DB_XPROXY_ADDR, [
	'proxy_ip'  => ip2hql( $ip ),     // binary column
	'proxy_crc' => ip2crc( $ip ),     // indexed uint32 for fast WHERE
] );
```

The `X'<hex>'` convention used by `ip2hql()` is understood by the DB CRUD wrappers — see
[Binary values](hack-db.md#binary-values-the-x-convention).

## Dates and Time

| Function | Purpose |
|----------|---------|
| `form2date` / `date2form` | Form `Y-m-d` ⇄ internal `Ymd`. |
| `date2show( $fmt, $date )` | `date()` over an internal `Ymd` value. |
| `smartdate( $tm )` | `d.m.y H:i`. |
| `tzdelta` / `tzdeltaline` | Minutes between two timezones / `±HH:MM`. |
| `workinghours( $shift, $tz = false )` | Is "now" inside a shift spec like `09:00-18:00@1-5`? |

## Numbers and Strings

| Function | Purpose |
|----------|---------|
| `tostr( $data )` | Force a value to string — arrays become `''`, everything else `(string)`. The standard wrapper for request values fed into string functions (see [Code Style](hack-style.md)). |
| `numinf( $lang, $n )` | Plural inflection from a 3–4 form language array. |
| `numinfi( $n )` | Index (0/1/2) of the plural form for `$n`. |
| `mkb( $bytes )` | Human file size. |
| `wrand( &$weights )` | Weighted random pick from `id => weight` (argument taken **by reference**). |
| `randstr( $len, $type = 0 )` | Random string (0: mixed, 1: lower, 2: with symbols). |
| `randtoken( $bytes = 16 )` | Cryptographically secure random hex token (`random_bytes`) — prefer over `randstr` for security tokens. |
| `sanitize_string( $text )` | `FILTER_SANITIZE_STRING` polyfill (strip tags, escape quotes) — a "back compatibility" helper. |
| `c32u( $data )` | Unsigned CRC32. |
| `fixmail( $email )` | Fix common typos (`gmail` → `gmail.com`). |
| `hasmacro` / `hasmacros` / `setmacro` / `setmacros` | Test / replace `{macro}` and `{ns:key}` tokens. |
| `parset( $params, $p, $v )` / `parmult( $params, $kv )` | Set/drop query params and rebuild the string. |
| `base64url_encode` / `base64url_decode` | URL-safe base64. |
| `jwt( $data, $code )` | Build a signed HS256 JWT. |
| `enc2log` / `log2dec` | Gzip-pack / unpack large log payloads. |

## CURL and Filesystem

| Function | Purpose |
|----------|---------|
| `curl( $url, $post = [], $config = [] )` | Full-featured HTTP request (proxy, auth, JSON, cache, verbose, …). |
| `curload( $url, $file, $config = [] )` | Download a URL to a file; `true` only for a complete answer — HTTP 200, no transfer error, not empty — otherwise the file is removed and the result is `false`. Connect timeout 30 s, at most 10 redirects (4 under `nolocal`); the `$config` keys are listed below. |
| `curvalid( $url, $nolocal = false )` | Syntax-and-lookup test: returns the URL, or `false` unless it is http/https with a valid host name or IP; `$nolocal` adds a lookup — the host must resolve and every IPv4 address pass `publicip()`. One lookup at one moment, not a guard for the fetch: that is `curload( … 'nolocal' => true )`. |
| `findfiles( $path, $deep = false )` | Recursive file list. |
| `cleandir( $path )` / `removedir( $path )` | Empty / delete a directory. |
| `cleancache( $path, $timeout )` | Delete files in `$path` older than `$timeout` seconds (`$timeout` is required). |
| `checkworking( $pid, $file )` | Is a process with `$file` in its cmdline alive? |
| `isroot()` | Is the script running as root? Guard a cron task's background launch with it — see [Task Schedule](hack-cron.md). |
| `removephp( $code )` | Strip PHP out of a template/HTML string. |
| `makecookie( $name, $value, $tm = 0, $path = '', $domain = '' )` | Set a cookie (honours `UNSAFECOOKIE`). |

```php
$resp = curl( 'https://api.example.com/check', [ 'ip' => $ip ], [ 'json' => true, 'timeout' => 10 ] );
$data = xxdec( $resp );

// Verbose mode returns code/time/redirect/data
$r = curl( $url, [], [ 'verbose' => true, 'nofollow' => true ] );
if ( $r['code'] == 200 ) { ... }
```

`curload()` writes to `$file` — an existing file is overwritten, and on any failure none is left behind.
Its `$config` keys, all optional:

- `timeout` — seconds for the whole call, redirects included (default 600).
- `maxsize` — bytes; a longer answer ends the transfer and the file is removed.
- `lowspeed` — `[ bytes, seconds ]`: give up when the transfer stays slower than that.
- `login` + `pass` — HTTP auth, sent to the first URL's scheme, host and port only, never to a redirect target elsewhere.
- `header` (an array of `Name: value` lines), `referer`, `request` (a custom method), `showheader` (the response headers go into the file too).
- `proxy`, `pauth` (`user:pass`) — an outgoing proxy; ignored under `nolocal`.
- `nolocal` — the full SSRF guard, off by default; use it for an address you do not control. The URL must be http/https on port 80 or 443 with a plain host name or dotted IPv4 — no `user@`, IPv6 literal, trailing dot or numeric spelling such as `127.1`. Every IPv4 address the name resolves to must pass `strictip()`, and the request is pinned to the tested address, so there is no second lookup to rebind. Redirects are followed by hand and each hop is tested again from the top, at most 5 requests in all. A source on a LAN is refused.

```php
// An address a user typed: on any failure there is no file left behind
$ok = curload( $url, $path, [ 'nolocal' => true, 'maxsize' => 5242880, 'timeout' => 60 ] );
if ( ! $ok ) return false;
```

`curl()` takes `nolocal` too, but it is only the `curvalid()` test before the request — the redirects `curl()` then follows are not tested again.

## Array Sorting

These sort an array of rows by one of their fields, **in place** (by reference).

| Function | Purpose |
|----------|---------|
| `deepsort( $arr, $field, $rev = false, $nat = false )` | Sort rows by a field. |
| `deepnatsort( $arr, $field )` | Natural-order sort by a field. |

```php
deepsort( $rows, 'proxy_fail', true );   // most failures first
```

## Advertising Click IDs

| Function | Purpose |
|----------|---------|
| `advertids()` | The list of known click-ID parameters (`fbclid`, `gclid`, …). |
| `advertid( $data )` | First click-ID present in a params array. |

## Pro-only helpers

Lite's set of global helpers is a **subset** of Pro's. These helpers exist in
**AlterCPA Pro only** — a hack that targets Lite must not rely on them:

| Function | Purpose |
|----------|---------|
| `params( $core, $list )` | Typed extraction of POST/GET params (`int`/`date`/`email`/`link`/`line` via `$core->text`). |
| `pages( $linkex, $items, $onpage, $page, $f = false )` | Render a Bootstrap pagination block. |
| `select( $data, $cur = null, $fld = null )` | Build `name/value/select` option rows for templates. |
| `array2xml( $array, $prev, $xml = false )` | Convert an array to XML. |
| `stripslashes_deep( $value )` | Recursively `stripslashes` a nested array. |

`msgo( $core, $msg )` (redirect back to the referer with a `message=` flag) exists in
both products, but prefer the method form `$core->msgo( $msg, $url )` documented in
[The Core](hack-core.md#output-and-flow-control).
