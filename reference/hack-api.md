# Creating API

Hacks can provide new API applications or extend existing ones with new functions. Overwriting existing functions in applications is not supported.

## API File

API functions are stored in the `mod/` folder, one file per application. The file name must start with `api` followed by the application name separated by a hyphen: `api-proxy.php`.

Application names: lowercase letters and numbers only, starting with a letter or number — no capitals, underscores, or hyphens in the name itself.

### Reserved words

An app or function name may never **start** with `core`, `data`, `hack` or `skin` — each is a
real top-level folder in the webroot, so a request whose first path segment matches one is
served as that folder by the webserver before PHP ever runs, never reaching the router.

`tasks` may not appear **anywhere** in an app or function name — in the app segment or the
function segment of the path alike. `tasks/` is the folder of the platform's cron launchers,
and the nginx host configuration that AlterCPA **Pro** requires keeps it shut to web requests
with `location ~ /tasks { deny all; return 403; }`. The pattern is unanchored: it matches
`/tasks` anywhere in a request path, so `api/proxy/tasks.json` is refused just like a web
request for a launcher script. The webserver answers with a bare HTTP 403 before PHP starts —
the client never sees a JSON `error`, your function is never reached, and the call is not
written to the API log. Use another word; the singular `task` does not match.

## Function Convention

Naming: `api_<app>_<func>` — e.g., `api_proxy_check`.

Access path: `api/[appname]/[funcname].[format]` — e.g., `api/proxy/check.json`.

**You cannot create functions with the names of existing system functions.**

Each function accepts two parameters: `$core` and `$user` (user ID). Functions must return an array following the [Response Format](#response-format) below.

```php
function api_proxy_check( $core, $user ) {
	$uid = (int) $core->get['uid'];
	if ( $core->hack->proxyland->check( $uid ) ) {
		return [ 'status' => 'ok', 'data' => [ 'id' => $uid ] ];
	} else return [ 'status' => 'error', 'error' => 'failed' ];
}
```

## Helper Functions

When an API needs shared helpers — e.g. to fetch standard data, normalize input, or
reuse logic across several endpoints — **do not name them with the `api_` prefix**. The
`api_` prefix is reserved for public API functions: every `api_<app>_<func>` is reachable
over `api/[appname]/[funcname]`, so a helper named that way becomes an unintended public
endpoint.

For helpers use the `apic_` prefix instead: `apic_<app>_<func>` — e.g.
`apic_proxy_getdata`. These functions are **not** publicly reachable and can be freely
called by the API functions themselves.

```php
// Private helper — not exposed as an endpoint.
function apic_proxy_load( $core, $uid ) {
	return $core->db->row( "SELECT * FROM proxy WHERE proxy_id = ?i", $uid );
}

function api_proxy_check( $core, $user ) {
	$row = apic_proxy_load( $core, (int) $core->get['uid'] );
	if ( $row ) return [ 'status' => 'ok', 'data' => [ 'id' => (int) $row['proxy_id'] ] ];
	else return [ 'status' => 'error', 'error' => 'not-found' ];
}
```

## Response Format

Every API function returns an array that the platform serializes into the requested format (JSON, etc.). All responses share the same fixed set of top-level fields, so any client can parse any of the platform's APIs the same way.

| Field | When | Description |
|-------|------|-------------|
| `status` | always | `ok` on success, `error` on failure. |
| `error` | on error | Machine-readable error code. Lowercase letters and digits, hyphen as separator. |
| `message` | optional | Human-readable text describing the result or error. Add it only when useful; clients should rely on `error`/`status`, not on `message`. |
| `data` | on success | The response payload — a single entity or a list (see below). Absent on errors. |
| `nav` | paginated lists | Pagination metadata. Omitted when the request is not paginated. |
| `dataset` | optional | Reference / lookup data that accompanies the payload but is not itself the payload (option labels, allowed values, dictionaries). A sibling of `data`, never nested inside it. |

### Success

```php
return [
	'status'	=> 'ok',
	'data'		=> [ /* payload */ ],
];
```

### Error

```php
return [
	'status'	=> 'error',
	'error'		=> 'no-id',
];
```

Error codes are always lowercase, words separated by hyphens (`no-id`, `bad-url`, `not-found`). A human-readable `message` may be added optionally:

```php
return [
	'status'	=> 'error',
	'error'		=> 'no-id',
	'message'	=> 'Entity identifier is required',
];
```

### Standard error codes

Reuse these common codes across all APIs for consistency. Define API-specific codes only when none of the standard ones fits, keeping the same lowercase-hyphen style.

| Code | Meaning |
|------|---------|
| `auth` | Authentication required or failed. |
| `scope` | Missing or invalid access scope / token. |
| `access` | Authenticated, but the action is not allowed for this user. |
| `unpaid` | Feature requires an active subscription or limit. |
| `no-id` | Missing entity identifier. |
| `no-<param>` | A required parameter is missing (`no-url`, `no-user`). |
| `bad-<param>` | A parameter has an invalid value (`bad-url`, `bad-domain`). |
| `bad-request` | The request is malformed in general. |
| `not-found` | The requested entity does not exist. |
| `duplicate` | The entity already exists. |
| `limit` | Rate or quota limit exceeded. |
| `db` | Database error. |
| `internal` | Unexpected internal error. |

These are **conventions**, not constants — your function returns them as plain strings.
The engine itself emits a few error codes **before** your function is ever reached:
`request` (incomplete app/func/format), `func` (unknown function), `key` / `auth` (bad
or missing credentials), `ban` (banned/inactive user), and `bot` (known crawler User-Agent
→ HTTP 418 in Lite; see the note below for the HTTP status in Pro).

> **The HTTP status differs by product.** AlterCPA **Lite** always maps the error code to
> the HTTP status (`auth`/`key`→401, `access`/`ban`→403, `unpaid`→402,
> `not-found`/`no-id`/`bad-id`→404, `app`/`func`/`db`→500, `bot`→418; any other error is
> 400, ok is 200) and ignores `?statuscode`. AlterCPA **Pro** answers **200** whatever the
> result; only when `?statuscode` is present (any value) does it answer 400 (error) /
> 200 (ok) — no per-code mapping, `bot` included. Both send `X-API-Status` (`ok` or
> `error`) and, for an error, `X-API-Error` (the code); Pro adds `X-Status`, the 400/200
> that `?statuscode` would apply. Lite's 406 keys off the **misspelled** `dublicate`, so
> `duplicate` answers 400 — don't rely on a specific HTTP code; clients should read the
> `status`/`error` fields.

### Data

The `data` field holds the actual payload and comes in two shapes.

**Single entity** — a flat associative array of the entity's short fields:

```php
'data' => [
	'id'	=> 42,
	'url'	=> 'example.com',
	'user'	=> 7,
],
```

**List** — an associative array keyed by the entity ID, each value being that entity's short fields:

```php
'data' => [
	42 => [ 'id' => 42, 'url' => 'example.com', /* ... */ ],
	43 => [ 'id' => 43, 'url' => 'other.com',   /* ... */ ],
],
```

### Navigation (nav)

When a list is returned page by page, add a `nav` block. Omit `nav` entirely when the request returns everything at once without pagination (`show = 0`).

| Field | Description |
|-------|-------------|
| `total` | Total number of matching records. |
| `page` | Current page number (1-based). |
| `pages` | Total number of pages. |
| `show` | Number of records per page. |

```php
'nav' => [
	'total'	=> 137,
	'page'	=> 2,
	'pages'	=> 7,
	'show'	=> 20,
],
```

Full paginated list example:

```php
$page = max( 1, (int) ( $req['page'] ?? 0 ) );
$show = max( 0, (int) ( $req['show'] ?? 0 ) );

if ( $show ) {
	$st		= ( $page - 1 ) * $show;
	$total	= $core->db->field( "SELECT COUNT(*) FROM ... WHERE $where" );
	$rows	= $total ? $core->db->data( "SELECT * FROM ... WHERE $where LIMIT $st, $show" ) : [];
} else {
	$rows	= $core->db->data( "SELECT * FROM ... WHERE $where" );
	$total	= count( $rows );
}

$data = [];
foreach ( $rows as $r ) $data[ (int) $r['id'] ] = [ 'id' => (int) $r['id'], /* ... */ ];

$out = [ 'status' => 'ok', 'data' => $data ];
if ( $show ) $out['nav'] = [
	'total'	=> (int) $total,
	'page'	=> (int) $page,
	'pages'	=> (int) ceil( $total / $show ),
	'show'	=> (int) $show,
];
return $out;
```

> Note: some older platform endpoints still return the legacy flat layout with `total`/`page`/`show` in the response root and no `pages`. New hacks must use the `nav` block described above.

### Dataset (dataset)

Some responses need to ship **reference data** alongside the entity — the option
labels behind a numeric field, the list of allowed values, small dictionaries the
client needs to render or interpret `data` without a second round-trip. This goes
in a top-level `dataset` block, a **sibling of `data`**, never mixed into it:

```php
return [
	'status'	=> 'ok',
	'data'		=> [ /* the entity / list */ ],
	'dataset'	=> [
		'auths'		=> [ 'api', 'key' ],                              // list of allowed values
		'tags'		=> [ 'utm_content', 'utm_term', 'utm_medium' ],   // index → label
		'repeats'	=> [ 0 => 'Once a day', 1 => 'Each hour' ],       // value → label
		'periods'	=> [ 0 => 'Today', 1 => 'Yesterday' ],            // value → label
	],
];
```

Keep the split strict: `data` is *what you asked for*, `dataset` is *what you need
to make sense of it*. Numeric enum fields in `data` (e.g. a `field` index) resolve
against the matching `dataset` array by key. Include `dataset` only when the
endpoint actually has such reference data to offer; omit it otherwise. It combines
freely with `nav` — a paginated list may carry both.

## Registering in start.php

```php
return [
	'api' => [ 'proxy', 'wm' ],
	'api-auth' => [
		[ 'proxy', [ 'check', 'add', 'del', 'edit', 'list' ] ],
		[ 'wm', 'getproxy' ],
	],
];
```

`api` — list of application names to register (loads `mod/api-<app>.php`).
`api-auth` — list of `[appname, funcname(s)]` pairs that **require API-key
authentication**. The function value may be a single name or an array of names.

> **A function not listed in `api-auth` is public.** Registering a function under
> `api-auth` is exactly what makes the engine run authentication for it; any function
> you do **not** list is reachable without a key. List every endpoint that touches
> user-scoped data or mutates state.

## Authentication

An authenticated call carries its credentials in the `id` parameter (query string
`?id=` or POST `authid`), formatted `<userid>-<apikey>` — the numeric user ID, a hyphen,
then that user's `api` key (see [`$core->user->api`](hack-user.md)). The API engine
accepts the request when either:

- the supplied key equals the user's stored `user_api`, **or**
- it is a valid **HMAC-SHA1 signature** of the request body —
  `hash_hmac( 'sha1', http_build_query( $post ), $user_api )` — which lets a client
  authenticate without sending the raw key.

The resolved user ID is passed to your function as the second argument (`$user`). Banned
or inactive users are rejected (`ban`). On Pro, an administrator may act on behalf of
another user via a `uu` parameter. If the request body arrives as
`Content-Type: application/json`, the engine decodes it into `$core->post` for you.

## Request and response formats

The format is the extension on the function (`check.json`). The engine serializes your
returned array accordingly:

| Format | Output |
|--------|--------|
| `json` | Compact JSON (`check.json`). |
| *(none)* | Pretty-printed JSON (default). |
| `serial` | PHP `serialize()`. |
| `text` / `raw` | The value as-is (return a string for these). |
| `xml` | XML — **AlterCPA Pro only**. |

Every API call is logged to `DB_LOG_API` (request, result, IP, geo, UA) unless disabled
by the `apilog` config or a per-call `nolog` flag.
