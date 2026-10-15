# The Core

`$core` (class `Core`) is the platform's god-object. It is
created once per request in `start.php`, exposed as the global `$core`, and passed as
the **first argument to every procedural module function**
(`function hack_proxy_init( $core )`). It owns the request data, lazily loads every
subsystem (database, templates, user, …), and drives output, configuration, caching,
sessions, the event system and routing.

This file covers the Core object itself — the framework plumbing. Its larger subsystems
have their own references (see [Subsystem References](#subsystem-references) at the end).

## Accessing the Core

Three ways, all the same object:

```php
// 1. The global, available anywhere
global $core;

// 2. The first parameter of every module function (the idiomatic way)
function hack_proxy_init( $core ) { ... }

// 3. Inside a library/class — stored on construction as $this->core
public function __construct( $core ) { $this->core = $core; }
```

Never instantiate `Core` yourself, and never call its constructor or destructor —
the lifecycle is owned by `start.php` and the shutdown handler.

## Lazy-loaded Subsystems

Subsystems are read as **properties** (`$core->db`, `$core->user`, …). The first access of each
loads the class and caches the instance for the rest of
the request — so repeated access is free, and a subsystem you never touch is never
loaded. `isset( $core->db )` tells you whether it has been loaded yet (without loading
it).

| Property | Class | Reference |
|----------|-------|-----------|
| `$core->db` | `MySQL` | [Working with database](hack-db.md) |
| `$core->tpl` | `Template` | [Templates and styling](hack-tpl.md) |
| `$core->user` | `User` | [User and request state](hack-user.md) |
| `$core->app` | `Application` | [User and request state](hack-user.md) |
| `$core->text` | `texter` | [Text processing](hack-text.md) |
| `$core->site` | `SiteElements` | [Site elements](hack-site.md) |
| `$core->media` | `MediaLib` | [Media library](hack-media.md) |
| `$core->email` | `eMailSender` | [E-mail sending](hack-email.md) |
| `$core->search` | `SearchWords` | Search-query parsing |
| `$core->lang` | array | Active language strings (`$core->lang['key']`) |
| `$core->langs` | array | Available languages (`code => name`) |

Beyond these fixed names, **any platform library** (or its hack override `hack_<name>`)
is reached the same way — `$core->skin`, `$core->hack`,
`$core->api`, `$core->browser`, `$core->geoip`, `$core->gang`, `$core->set`,
`$core->currency`, `$core->phone`, `$core->crypto`, … The loader resolves the file,
prefers a `hack_<name>` class when a hack provides one, and instantiates it with
`new $name( $core )`. Hack libraries are documented in [Libraries](hack-lib.md).

## Request Input

Incoming request data is exposed as public arrays, **already escaped** with
`addslashes()` at the Core boundary:

| Property | Source |
|----------|--------|
| `$core->get` | `$_GET` |
| `$core->post` | `$_POST` |
| `$core->cookie` | `$_COOKIE` |
| `$core->files` | `$_FILES` |
| `$core->server` | `$_SERVER` |

```php
// Read a request value POST-first, then GET; null when absent
$id = $core->req( 'id' );

// Direct access (remember: never trust the type, cast/sanitize for non-string context)
$page = (int) ( $core->get['page'] ?? 1 );
$name = $core->text->line( $core->post['name'] ?? '' );
```

Because values are pre-escaped you may interpolate them straight into SQL string
literals (`'$value'`), but you must cast to `(int)` or run them through
`$core->text->*` anywhere a value lands in a numeric or identifier position. Full rules
and the second-order-injection caveat (values read back from the DB are **not** escaped)
are in [SQL safety](hack-db.md#sql-safety).

> Per the [code style](hack-style.md), wrap request values in `tostr()` before passing
> them to string functions — it turns arrays into `''` and everything else into a
> string, avoiding the `"Array"` warning of a bare `(string)` cast.

## URLs and Links

| Method | Result |
|--------|--------|
| `u( $path, $param = false, $replace = false )` | Site-relative URL. `$path` is a string or array of segments; `$param` is a query string or array; `$replace` overrides individual params. |
| `uri( $path, $param, $replace )` | Same as `u()` but absolute, built on `config('url','base')`. Use for e-mails, external links, callbacks. |
| `apiu( $app, $func, $param = false, $format = 'json' )` | Full API endpoint URL: `<base>/api/<app>/<func>.<format>`. |
| `url( $id, ... )` | `sprintf` over a built-in template (`css`, `js`, `link`, `meta`, …). Mostly internal. |

```php
$core->u( 'proxy-list' );                          // /proxy-list
$core->u( [ 'proxy', $id ], [ 'tab' => 'log' ] );  // /proxy/123?tab=log
$core->u( '', 'logout=site' );                     // /?logout=site
$core->uri( 'profile' );                           // https://site.tld/profile
```

## Output and Flow Control

These end the request — they call `$core->stop()` internally and never return, so
anything after them is dead code.

| Method | Effect |
|--------|--------|
| `go( $url, $hard = false )` | `Location:` redirect. `$hard` sends a 301. |
| `msgo( $msg, $url, $mt = 'message', $ref = false )` | Redirect back (to referer or `$url`) carrying a `message=` flash code. |
| `json( $data, $nice = true )` | Emit `$data` as `application/json` and stop. |
| `stop( $auto = false )` | Flush output, save the session, render the queued templates, then `die()`. |
| `debug( $raw = true )` | Turn on error display (`text/plain` when `$raw`). Development only. |

```php
if ( ! $id ) $core->msgo( 'badid', $core->u( 'proxy-list' ) );
$core->json([ 'status' => 'ok', 'id' => $id ]);
```

The normal page lifecycle does **not** call `stop()` explicitly in most cases — the
registered shutdown handler unsets `$core`, whose destructor runs `stop( true )`. You
call `stop()` yourself only to finish early (after `json()`/`go()` it is automatic).

## Configuration

Site configuration is stored as named JSON blocks in `DB_CONFIG`. A block is addressed by
`$name` and holds **one associative array carrying every setting of that feature** — the
whole set at once, under one name. A block that exists is cached; one that does not is
**not** (see *Three things about the key* below).

> **The rule: one feature per scope → one array → one block.** `config()` reads the entire
> settings array and `reconf()` writes the entire settings array. Never a config block per
> setting, and — the mistake that keeps being made — **never a database column or a settings
> table per setting**. Settings get no schema of their own: `DB_CONFIG` has exactly two columns
> (`name`, `value`), and a hack's own settings live in the single `hack_config` text column
> (see [Configuration page](hack-config.md)). Turning `token`, `limit`, `on` into columns of
> a `cpa_x<hack>_config` table — or into three separate config blocks — is wrong every
> time: it is the array you already have, spread across DDL you now have to migrate.
>
> *Per scope* is the one legitimate reason to have more than one block for a feature — see
> **Per-user and per-team settings** below. It is still one array per block; only the `$name`
> changes.

| Method | Purpose |
|--------|---------|
| `config( $name )` | Whole config block as an array (`[]` if missing). |
| `config( $name, $field )` | One field of the block (`null` if missing). |
| `reconf( $name, $value )` | Persist `$value` as the block's new content and drop its cache. |

```php
// Read: whole block, then one field
$cfg   = $core->config( 'myhack' );             // [ 'token' => …, 'on' => 1, … ] or []
$token = $core->config( 'myhack', 'token' );    // one field, null if absent

// Write: reconf REPLACES the entire block — there is no per-field setter.
// Read → modify the array → write the whole thing back.
$cfg = $core->config( 'myhack' );
$cfg['token'] = $token;
$cfg['on']    = 1;
$core->reconf( 'myhack', $cfg );
```

**Always model settings as one array**, even when there is a single setting today — you
will almost always grow it, and `reconf()` overwrites the whole block, so keeping the
fields in one array is the only way to add a second setting later without clobbering the
first. Nested structure goes inside that array too: a list of rules is `$cfg['rule']`, an
array of arrays, not a table and not one block per rule.

A **bare scalar block** is not a setting — it is an internal state marker the code writes
and reads itself: a cron timer (`reconf( 'hack-<name>-last', time() )`, see
[Cron tasks](hack-cron.md)), a last-processed id (`reconf( 'lastpaidmail', $tm )`), a
platform flag (`reconf( 'dbversion', DBVER )`). Those are legitimately scalar. Anything a
human edits on a settings page is a **field of the one array**, never a block of its own.

Hacks usually keep their own settings in their **hack configuration** — the per-hack
`hack_config` column reached through `$core->hack->myhack->conf` — rather than in global
`config()` blocks. That store is one array as well; it merely adds a convenience setter
for a single key (`conf( $field, $value )`), which still rewrites the whole stored array.
See [Configuration page](hack-config.md).

> **What must *not* go in `hack_config`:** anything the code generates at runtime — a
> derived secret, a seed, a cached remote id. Saving the hack's Settings form rebuilds that
> column from the `param.json` fields alone and writes it whole, deleting every key you added
> from code. Runtime state belongs in a `$core->config()` block, which is a different table
> and is never touched by that save. See [Configuration page](hack-config.md).

### Three things about the key

`$name` is the whole addressing scheme, and it is thinner than it looks.

- **`cpa_config.name` is `varchar(20)`.** Count the key before writing it. Over-length is not
  an error you will notice: a server without `STRICT_*` in its `sql_mode` **truncates
  silently**, so two keys sharing their first 20 characters collapse into one row that both
  read and both overwrite; a strict server errors instead. `hack-<name>-<what>` runs out fast — a
  nine-character hack name leaves five for everything after it, and any key built by appending
  an id needs the id's widest form counted in.
- **`name` is interpolated raw into the SQL.** `config()` and `reconf()` concatenate it into
  the query unescaped. Build keys from literals and
  `(int)`-cast ids only, never from request text. The cast stops the injection; it does not
  bound the length, which is the separate check above.
- **A missing block is not negatively cached.** `json_decode('')` is `null`,
  `cache( $name, null )` is a read rather than a write, and the `isset()` guard that would have
  short-circuited the query is false for `null` — so a block that has never been written
  re-runs its `SELECT` on **every** `config()` call, within one request as much as across
  requests. Read such a block once into a variable, or keep your own `cache()` layer over it.

### Per-user and per-team settings

A block is install-wide by default, but nothing requires `$name` to be a constant. Where a
feature is configured **per user or per team**, the platform's pattern is one block per scope,
named by a signed id — positive a user, negative a team:

```php
	$auth = $core->app->auth();
	$uga  = $auth['gang'] ? -$auth['gang'] : $auth['user'];
	$cfg  = $core->config( 'tracker' . $uga );	// tracker123 - user 123 ; tracker-45 - team 45
```

That is how the platform's own tracker stores its settings: it reads `'tracker' . $user` for a
user, `'tracker-' . $gang` for a team, and writes either of them back through the one
expression. Each block is still one array holding every setting of the feature — the rule above
is unchanged, only the name carries the scope.

Three things come with the pattern:

- **Cache it yourself.** Most scopes have never saved anything, which is exactly the un-cached
  case above. The tracker keeps its own `cache()` entry per block over the top and drops that
  one key on save — no install-wide flush.
- **Refuse scope 0.** Bail on a zero id, as the tracker does, or every guest and every
  team-less session shares one block.
- **Bound the id.** The length budget now depends on a value from `$auth`, and `$auth['gang']`
  is not always a validated team id — for a system administrator it is whatever `?g=` carried.
  Clamp it to its column's range before concatenating.

## Cache

A key→value store over the best backend available (Memcached → Memcache → APCu →
files, auto-detected). Values are JSON-serialised, so store arrays/scalars, not objects
or resources.

| Method | Purpose |
|--------|---------|
| `cache( $name )` | Read; returns `null` when absent. |
| `cache( $name, $value )` | Write (stored in the engine immediately). |
| `uncache( $name )` | Invalidate a key immediately. |

```php
$data = $core->cache( "xproxy.list.$id" );
if ( ! isset( $data ) ) {
	$data = $core->db->data( "SELECT * FROM ".DB_XPROXY_ADDR." WHERE proxy_id = '$id'" );
	$core->cache( "xproxy.list.$id", $data );
}
```

Because `$value === null` means "read", you cannot cache a literal `null`. Namespace
your keys with the hack prefix (`xproxy.…`) to avoid collisions.

## Sessions

A per-visitor server-side store, loaded automatically from the `ssid` cookie and saved
at request end. Authentication, language and security fingerprinting all ride on it.

| Member | Purpose |
|--------|---------|
| `$core->session` | The session data array (read directly). |
| `sset( $name, $value )` | Set a session value and mark the session dirty. |
| `$core->session_id` | The opaque session token (string), or `false`. |

```php
$core->sset( 'xproxy_step', 2 );
$step = $core->session['xproxy_step'] ?? 0;
```

## Events: Handlers and Filters

The event system is **the** way a hack injects behaviour into the platform without
touching core files. Subscribe in your hack's `init`, and the core fires events at the
right moments.

| Method | Purpose |
|--------|---------|
| `handle( $name, $function )` | Subscribe a callback to an event (multiple allowed). |
| `cando( $name )` | Is anyone subscribed to `$name`? |
| `process( $name, $info = null )` | Fire `$name` as an action: each callback gets `( $core )` or `( $core, $info )`. Returns the OR of their return values. |
| `filter( $name, $data, $info = null )` | Fire `$name` as a filter: `$data` is threaded through every callback (`$data = f( $core, $data[, $info] )`) and the final value returned. |

```php
// In hack_proxy_init():
$core->handle( 'menu', 'hack_proxy_menu' );           // transform the menu
$core->handle( 'texter_line', 'hack_proxy_cleanline' );

// A filter callback returns the (possibly modified) data:
function hack_proxy_menu( $core, $menu ) {
	$menu['tools'][] = 'proxy';
	return $menu;
}
```

Use `process()` for side effects (the boolean tells you whether anything handled it) and
`filter()` to transform a value. A filter callback **must return** the (possibly modified)
data — forgetting to `return $data` blanks it.

The most useful hack-facing events, all of which fire in **both** Lite and Pro:

| Event | Kind | Fires |
|-------|------|-------|
| `startup` | action | very early, once Core and the user are ready |
| `libraries` | action | after libraries load, before routing |
| `actions` | action | before routing — the place to handle state-changing `action=` posts |
| `modules` | action | the routing slot — dispatch your own pages here |
| `menu` / `newmenu` | filter | build / post-process the navigation menu |
| `header` / `footer` | action | just before the page header / footer render |
| `medialibs` | filter | register media libraries (see [Media library](hack-media.md)) |
| `languages` | filter | register languages (see [Language files](hack-lang.md)) |
| `texter_line` / `texter_code` / `texter_link` / `texter_lines` | filter | hook text sanitizers/formatters |
| `showsite-start` / `showsite-stop` | action | around public-site page rendering |

Pro adds many domain events (orders, offers, postbacks, deliveries — e.g. `order-pre-add`,
`postback_make`, `delivery_types`). The set differs by product and version; when you need an
injection point that is not listed here, ask the platform vendor which event fires there.

## Modules

Modules are procedural controllers loaded on demand. `mod()` resolves the file and,
optionally, calls an entry function with `$core`.

```php
$core->mod( 'route-app' );                       // just load the module
$core->mod( 'base-login', 'base_login_page' );   // load and call base_login_page( $core )
```

A name with a dash maps to a folder — `route-app` resolves to `route/app`, falling back to the
flat `route-app` — and a module the application overrides wins over the base one. Your own
modules resolve the same way inside your hack; see [Modules](hack-mod.md) for writing them.

## Locks

Cooperative file locks for serialising work (cron jobs, queue workers). Backed by
`flock` on files under `data/work/`.

| Method | Purpose |
|--------|---------|
| `lock( $name, $id = false, $wait = true )` | Acquire a lock; `true`/`false`. With `$wait = false` returns immediately if busy. |
| `unlock( $name, $id = false, $result = true )` | Release; returns `$result` (handy as `return $core->unlock( $name, $id, false );`). |

```php
if ( ! $core->lock( 'xproxy-check', $id, false ) ) return false;  // already running
// ... exclusive work ...
$core->unlock( 'xproxy-check', $id );
```

Any locks still held are released automatically in `stop()`.

## Languages

| Member | Purpose |
|--------|---------|
| `$core->lang` | Active language array — `$core->lang['proxy_h']`. |
| `$core->language` | Active language code (`'en'`, `'ru'`). |
| `$core->langs` | Available languages as `code => name`. |
| `guesslang()` | Resolve the language (user meta → `LANG` → `Accept-Language` → `en`). |
| `loadlang( $code )` | Load and merge a specific language file set. |

Language file authoring (including the per-hack merge order) is covered in
[Language files](hack-lang.md).

## Subsystem References

The Core's heavier subsystems and the global helper library each have their own file:

| Reference | Covers |
|-----------|--------|
| [Working with database](hack-db.md) | `$core->db` — queries, CRUD, SQL safety, schema |
| [Templates and styling](hack-tpl.md) | `$core->tpl` / `$core->skin` — parts, blocks, macros |
| [User and request state](hack-user.md) | `$core->user` and `$core->app` (routing) |
| [Text processing](hack-text.md) | `$core->text` — sanitizers, validators, formatters |
| [Site elements](hack-site.md) | `$core->site` — header, footer, menus, forms, assets |
| [Media library](hack-media.md) | `$core->media` — uploads, storage, image resizing |
| [E-mail sending](hack-email.md) | `$core->email` — `send()` and mail configuration |
| [Global functions](hack-func.md) | Global helpers — IP, JSON, dates, CURL, … |
