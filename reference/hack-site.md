# Site Elements

`$core->site` (class `SiteElements`) builds the chrome around your page
content: header, footer, breadcrumbs, menus, forms, pagination and asset inclusion. It
is loaded lazily on first access — see [The Core](hack-core.md).

`$core->site` decides *which* assets and page parts to emit; the actual rendering goes
through the template engine. For template parts, blocks, directives and skin macros see
[Templates and styling](hack-tpl.md).

## Page Flow

| Method | Purpose |
|--------|---------|
| `header()` | Render the site header (menus, title, queued messages). Connect assets/sets **before** calling it. |
| `footer()` | Render the footer (follow with `$core->stop()`). |
| `bc( $name, $url = false )` | Add a breadcrumb (also extends the page title). |
| `ht( $name )` / `pt( $name )` | Set the full header title / the page (`<h1>`) title. |
| `info( $type, $text )` | Queue a flash message. `$text` may be a `lang` key. See the `$type` note below. |

`info()` maps only four `$type` values to a colour: `ok` and `info` both
render **green** (`success`), `error` renders **red** (`danger`), and `text` renders a
plain note. Any other string (e.g. `warning`, `primary`) is passed through verbatim as
the alert's CSS class via the `{tpl:alert-<type>}` macro — so `warning` "works" only
because a matching macro exists, it is not a first-class type. Stick to `ok`/`error`/`info`/`text`.

The standard page skeleton:

```php
$core->site->bc( $core->lang['proxy_h'], $core->u( 'proxy-list' ) );
$core->site->info( 'ok', 'proxy_saved' );      // optional flash message
$core->site->header();

// ... queue the body template — see hack-tpl.md ...

$core->site->footer();
$core->stop();
```

## Builders

| Method | Purpose |
|--------|---------|
| `form( $name, $action, $title, $fields, $button = false )` | Render a full form from a field-definition array. |
| `page( $url, $start, $perpage, $count )` | Render a pagination block plus a "shown X–Y of Z" line. |
| `config( $type, $list )` | Render a settings-index grid of config sections. |

`form()` builds the markup from an array of field definitions — each entry has a `type`
and the relevant keys. Supported types: `head`, `line`, `hidden`, `text`,
`email`, `number`, `pass`, `textarea`, `checkbox`, `vcheckbox` (alias of `checkbox`),
`select`, `mselect`, `radio`, `radioline`, `file`, `date`, `datetime`, `button` (an
action button with optional `confirm`), `code` (CodeMirror) and `mces`/`mcea` (WYSIWYG).
The editor/date/select2 assets are pulled in automatically when a field needs them.

Per-field keys the builder understands, beyond `type`:

| Key | Applies to | Meaning |
|-----|------------|---------|
| `name` | most | Field name (`name="..."`). |
| `head` | most | Label shown above the field. |
| `value` | most | Current value. |
| `descr` | most | Help text under the field. |
| `req` | inputs | Mark required (`*`). |
| `ro` | text/email/number/textarea | Read-only. |
| `ph` | text inputs | Placeholder. |
| `length` | text/pass | `maxlength`. |
| `min` / `max` / `step` | number | Numeric bounds. |
| `rows` | textarea/code | Row count (code defaults to 15). |
| `options` | select/mselect/radio | `value => label` choices. |
| `checked` | checkbox | Pre-checked. |
| `s2` | select | Render as a select2 searchable box. |
| `raw` | mselect | Skip select2. |
| `lang` / `mime` | code | CodeMirror mode. |
| `text` / `confirm` | button | Button caption / `confirm()` prompt. |
| `skip` | any | Skip rendering this entry. |

```php
$core->site->form( 'body', $core->u( 'proxy', $id ), $core->lang['proxy_edit'], [
	[ 'type' => 'text',   'name' => 'name',   'head' => $core->lang['name'],   'value' => $d['proxy_name'], 'req' => true ],
	[ 'type' => 'number', 'name' => 'port',   'head' => $core->lang['port'],   'value' => $d['proxy_port'] ],
	[ 'type' => 'select', 'name' => 'status', 'head' => $core->lang['status'], 'value' => $d['proxy_status'],
	  'options' => $core->lang['proxy_status'] ],
] );
```

For the full field-type reference, the add/edit page pattern, buttons, choice fields,
checkbox groups and dynamic field building, see the dedicated
[Building forms](hack-form.md) guide.

For the pager (`page()`), see the dedicated [Pagination](#pagination) section below.

## Pagination

`page( $url, $start, $perpage, $count )` renders the pager for a `LIMIT`ed list. It
**returns an HTML string** — a `<ul class="pagination">` plus a "Shown X–Y of Z" line —
which you hand to the template as a variable; it does not emit anything by itself.

| Argument | Meaning |
|----------|---------|
| `$url` | Base URL of the list (from `$core->u()`); page links append `?page=N` or `&page=N`. |
| `$start` | Row offset of the current page — the `LIMIT` **start**, not the page number. |
| `$perpage` | Rows per page (the `LIMIT` length). |
| `$count` | Total number of matching rows (`COUNT(*)`). |

The current page number is **not** an argument — `page()` reads it from `$core->app->page`,
which is taken from `$_GET['page']` (defaults to `1`, clamped to `>= 1`). You only pass the
derived offset. The full pattern is: turn the page into an offset, run `COUNT(*)` and a
`LIMIT`ed query on the **same** `WHERE`, then call `page()`:

```php
// Current page → LIMIT offset. $sh = rows per page, $st = offset.
$page = $core->app->page; $sh = 20; $st = $sh * ( $page - 1 );

$cc   = $core->db->field( "SELECT COUNT(*) FROM ".DB_XPROXY_ADDR." WHERE $where" );
$rows = $core->db->data(  "SELECT * FROM ".DB_XPROXY_ADDR." WHERE $where ORDER BY proxy_id DESC LIMIT $st, $sh" );

$core->tpl->vars( 'body', [
	'pages' => $core->site->page( $core->u( 'proxy-list', $params ), $st, $sh, $cc ),
] );
```

Place the variable where the pager should appear in the skin — the platform convention is
inside the list wrapper:

```html
<div class="lister">{pages}</div>
```

Notes:

- **Return values.** `$count == 0` → empty string (nothing to page). A single page → only
  the "Shown …" line, no navigation. Two or more pages → navigation plus the line.
- **The "Shown X–Y of Z" line** is built with `sprintf` from the `shown` language string
  (`'Shown %d to %d of %d'`), which core already defines — supply your own `shown` key only
  if you localise it.
- **Preserve filters.** Pass any active search/filter GET params through `$core->u()` (the
  `$params` above) so page links keep them; `page()` detects the existing query string and
  adds `page=N` on top with the right separator (`?` or `&`).

## Navigation menu

`header()` renders the main menu from the public `$core->site->menu` / `->menus` /
`->counter` arrays, but a hack does **not** write them directly — it transforms the menu
through the `menu` filter (and post-processes via `newmenu`). Subscribe in your hack's
`init` and append your section's item key:

```php
// In hack_proxy_init():
$core->handle( 'menu', 'hack_proxy_menu' );

function hack_proxy_menu( $core, $menu ) {
	$menu['tools'][] = 'proxy';   // add 'proxy' under the Tools group
	return $menu;                 // a filter callback MUST return the (modified) data
}
```

The item key (`proxy`) resolves to a `menu_proxy` language string and a `$core->u()`
link. See [Events](hack-core.md#events-handlers-and-filters) and
[Language files](hack-lang.md).

## Assets

Each file is added only once, even if requested repeatedly. Connect everything **before**
`header()` — the header template renders the `<head>` block. The bigger picture
(extension-less names, ready-made sets, skin resolution) is in
[Including assets](hack-tpl.md#including-assets).

| Method | Purpose |
|--------|---------|
| `css( $name )` / `js( $name )` | Include a system (or `[mod, name]`) stylesheet / script. |
| `hcss( $hack, $name )` / `hjs( $hack, $name )` | Include a hack stylesheet / script. |
| `set( $name )` | Add a ready bundle: `select2`, `tablesorter`, `summernote`, `bootstrap`, `adminlte`. |
| `script( $code )` | Queue an inline JS snippet for the footer. |
| `meta( $name, $value )` / `metaline( $raw )` | Add a `<meta>` tag. |

```php
$core->hack->proxy->css( 'proxy' );   // /hack/proxy/css/proxy.css  (preferred for hacks)
$core->site->set( 'tablesorter' );
$core->site->header();
```

For a hack's own assets, the `$core->hack->(name)->css()/js()` helpers
(see [Main class](hack-main.md)) are equivalent to `hcss()/hjs()` and read better.
