# User and Request State

Two Core subsystems describe *who* is making the current request and *what* they asked
for: `$core->user` (the authenticated user) and `$core->app` (the parsed request /
routing). Both are loaded lazily on first access — see [The Core](hack-core.md).

## The Current User — `$core->user`

The `User` class is constructed once, runs authentication (session
cookie, login form, password recovery, AlterCPA SSO), and exposes the result as
read-only properties.

| Property | Meaning |
|----------|---------|
| `id` | User ID (`0` for a guest). |
| `name` / `email` | Display name and login e-mail. |
| `level` | Privilege scale (`0` = ordinary user; **any non-zero value = staff/admin**; `1` is the full administrator the admin routes/menus check for). |
| `auth` | `true` when authenticated. |
| `ban` / `inactive` / `warn` | Account state flags. |
| `vip` | VIP flag. |
| `gang` | Team (gang) ID, `0` if none. |
| `tz` | User timezone, or `false`. |
| `paid` | Paid-until timestamp. |
| `reg` | Registration timestamp. |
| `api` | The user's API key (used to authenticate API calls). |
| `ip` | Remote IP of the request. |
| `meta` | Decoded user meta array (`$core->user->meta['lang']`). |

The table above lists the properties present in **both** products. AlterCPA **Pro**
adds many more — most importantly `work` (account role/state that drives Pro's routing),
the balance fields `cash` / `bonus` / `withdraw` / `purse`, the referrer `ref`, and
company/team fields (`comp`, `team`, `teamlead`, `oacl`, …). A hack that targets Lite
must not rely on the Pro-only fields; check the live user object of your target
product before using anything beyond the shared set.

```php
// Admin-only guard at the top of a module
if ( ! $core->user->level ) $core->go( $core->u( '' ) );

// Greet the current user
$name = $core->user->auth ? $core->user->name : $core->lang['guest'];
```

### Methods

| Method | Purpose |
|--------|---------|
| `get( $id, $field = false )` | Load any user's row (cached). |
| `getbyname( $email, $field = false )` | Load a user by e-mail. |
| `set( $id, $data )` | Update DB columns for a user and drop caches. |
| `edit( $data )` | Update the *current* user; keys auto-prefixed `user_`. |
| `meta( $id, $data )` | Merge values into a user's meta block. |
| `reset( $id )` | Drop a user's cached row. |
| `pass( $text, $sid = 1 )` | Hash a password (HMAC-SHA256 with `SALTKEY*`). |
| `checkpass( $input, $check, $sid = 1 )` | Verify a password / token. |
| `avatar( $id, $min = false, $uri = false )` | URL of the user's avatar. |
| `secret( $user = 0, $len = 10 )` | Deterministic secret token for the user. |

```php
// Update the current user, store a hack flag in meta
$core->user->edit( [ 'tz' => 'Europe/Moscow' ] );
$core->user->meta( $core->user->id, [ 'xproxy_seen' => time() ] );

// Look up another user's e-mail
$mail = $core->user->get( $wm, 'user_mail' );
```

`get()`/`getbyname()` return the raw DB row (columns prefixed `user_`) plus a decoded
`meta` key; guard with `empty()` and remember the values are **not** re-escaped if you
feed them back into SQL.

## Routing State — `$core->app`

The `Application` class parses `REQUEST_URI` into the parts a module
needs. The URL `/<div>/<item>/<subdiv>/<subitem>` maps to:

| Property | Meaning |
|----------|---------|
| `div` | First path segment — the module. |
| `item` / `id` | Second segment, symbolic and `(int)`. |
| `subdiv` | Third segment. |
| `subitem` / `subid` | Fourth segment, symbolic and `(int)`. |
| `action` | `action=` from POST/GET (referer-checked for CSRF). |
| `message` | `message=` flash code from POST/GET. |
| `page` | `page=` number, minimum `1`. |
| `path` / `pp` | Raw segment array / cleaned path string. |
| `api_app` / `api_func` / `api_fmt` / `api_id` | API request parts when `div == 'api'`. |

```php
// Dispatch inside a module entry function
if ( $core->app->div == 'proxy' && $core->app->id ) {
	$id = $core->app->id;                       // /proxy/123
	if ( $core->app->action == 'delete' ) {     // /proxy/123?action=delete
		// ... delete, then redirect with a flash message ...
	}
}
```

> When an `action` is present but the request's `Referer` does **not** match the site
> host, the platform's CSRF guard not only unsets `action` — it **discards
> the entire `$core->post` array**. So a cross-site POST reaches your code with no
> `action` *and* no POST body. Trigger state-changing operations through `action=`
> (a form post or a same-site link), never through arbitrary GET params.

### Viewing scope — `auth()`

`$core->app->auth()` resolves the webmaster/gang scope for statistics pages (who the
admin is viewing, team access, "all items" mode) and returns an array with
`wm`, `user`, `gang`, `all`, `can` (`wm` is `null` when there is no webmaster scope).
Use it on stats/report pages.

`$core->app->gauth( … )` is a companion boolean helper that tests whether the current
user may access a given gang's data — use it to gate team-scoped features.
