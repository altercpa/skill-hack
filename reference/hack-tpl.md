# Templates and Styling

Templates control the appearance of displayed pages. Within MVC, they are the "views" — plain HTML files with placeholders and HTML-comment directives. The engine (class `Template`) compiles each template to PHP once and caches the result; the controller only feeds it data.

> **Building an actual page?** This page covers the *engine* (blocks, directives, macros, caching).
> For the concrete layout patterns the platform reuses everywhere — collapsible search toolbars,
> boxes, compact tables with mirrored headers, detail modals, multi-column and modal-loaded
> editors, live-save fields — read the companion **[Template Patterns cookbook](hack-tpl-patterns.md)**.
> A hack page must be indistinguishable from a native one; copy those patterns class-for-class.

## Folder Structure

| Folder | Contents |
|--------|----------|
| `tpl/` | Template structure files |
| `css/` | CSS styling files |
| `js/` | JavaScript files |
| `img/` | Images for CSS files |

A hack ships these under its own folder, e.g. `hack/<name>/tpl/proxy-list.tpl`, `hack/<name>/css/style.css`.

## Including Assets

Reference files by name without extension (`style.css` → `style`, `script.min.js` → `script.min`). Each asset is added to the page only once, even if requested repeatedly.

```php
$core->hack->(name)->js( $name );   // Hack script  → /hack/(name)/js/(name).js
$core->hack->(name)->css( $name );  // Hack style   → /hack/(name)/css/(name).css
$core->site->js( $name );           // System script → /core/js/(name).js
$core->site->css( $name );          // System style  → /core/css/(name).css
$core->site->js( [ $mod, $name ] ); // Module script → /(mod)/js/(name).js
$core->site->set( $name );          // Add a ready-made asset set
```

Available sets for `set()`: `core` (modern default bundle), `adminlte` (classic AdminLTE bundle), `bootstrap`, `select2`, `tablesorter`, `summernote`. Each pulls in the right jQuery/CSS/JS and any init script. Connect sets and assets **before** drawing the header (`$core->site->header()`), because the header template renders the `<head>` block.

## Template Path Resolution

A hack template resolves to `hack/<hack>/tpl/<name>.tpl`. System templates resolve
the application template directory → the base template directory (a hack can ship a matching file to override a system
template — see *Overriding Templates* below). Compiled output is cached in `data/tpls/`.

## Template Engine

The controller loads a template into a named **part** (a handle such as `body`, `header`, `footer`), fills it with data, and queues it for output. Parts are rendered in the order they were queued, at the end of the request.

```php
// Load a hack template into the 'body' part
$core->hack->(name)->tpl( 'body', 'my-template-file' );

// Set root-level scalar variables
$core->tpl->vars( 'body', [ 'foo' => 'bar', 'peace' => 'deathz' ] );

// Add one row to a repeating block (dot notation nests: 'menu.sub.item')
$core->tpl->block( 'body', 'item', [ 'myvar' => 'myvalue' ] );

// Queue the part for output (rendered at request end)
$core->tpl->output( 'body' );
```

### Data API

| Method | Purpose |
|--------|---------|
| `tpl( $part, $name )` | Load a hack template file into a part (via `$core->hack->(name)`). |
| `vars( $part, $array )` | Merge root-level scalar variables. Call repeatedly to add more. |
| `block( $part, $name, $row )` | Append one row to block `$name`. Each call = one rendered iteration. |
| `info( $part, $array )` | Convenience: scalar values become `vars()`, array values become a series of `block()` rows. |
| `output( $part )` | Queue the part. All queued parts render in order when the page finishes. |
| `make( $part )` | Render a part **to a string** immediately instead of queuing (AJAX/partials). |

The standard page flow — `header()` queues `header`, the controller queues `body`, `footer()` queues `footer`, and the core renders the whole queue at shutdown:

```php
$core->site->header();
$core->hack->(name)->tpl( 'body', 'my-template' );
$core->tpl->vars( 'body', [ ... ] );
// ... block() calls ...
$core->tpl->output( 'body' );
$core->site->footer();
$core->stop();
```

### Blocks

A block is a region repeated once per row added with `block()`. **A block with no rows renders nothing** — there is no automatic empty state. The convention is a second block (`noitem`) added explicitly when the list is empty:

```php
if ( $rows ) {
	foreach ( $rows as $r ) $core->tpl->block( 'body', 'item', [
		'name'   => $r['name'],
		'active' => $r['active'] ? true : false,
	] );
} else $core->tpl->block( 'body', 'noitem' );
```

The same block name may appear several times in one template — each `<!-- BEGIN ... -->` re-iterates the rows already stored (e.g. a table that also feeds a set of per-row modals or `<option>`s).

**Nested blocks** use dot notation. Add the parent first, then children attach to the *most recently added* parent row:

```php
$core->tpl->block( 'header', 'menu', [ 'name' => 'Reports' ] );  // parent row
$core->tpl->block( 'header', 'menu.sub' );                       // container under that row
$core->tpl->block( 'header', 'menu.sub.item', [ 'name' => 'Daily', 'link' => '...' ] );
$core->tpl->block( 'header', 'menu.sub.item', [ 'name' => 'Weekly', 'link' => '...' ] );
```

## Template Directives

Templates are plain HTML with placeholders and HTML-comment directives.

| Syntax | Meaning |
|--------|---------|
| `{name}` | Scalar variable from `vars()`. A missing variable renders as an empty string. |
| `{block.field}` | A field of the current block row. |
| `{parent.child.field}` | A field of a nested block. **Only the last two segments matter** — `{menu.sub.item.name}` and `{item.name}` compile identically; the longer form is written for readability. |
| `<!-- BEGIN name -->` … `<!-- END name -->` | Repeated once per row added with `block( $part, 'name', [...] )`. Renders nothing when no rows were added. Nest `BEGIN`/`END` to match nested blocks. |
| `<!-- IFSET name -->` … `<!-- ENDIF -->` | Render the body when the variable/field is **non-empty**. |
| `<!-- IFNOT name -->` … `<!-- ENDIF -->` | Render the body when the variable/field is **empty**. |
| `<!-- ELSE -->` | Else branch inside an IF block. |
| `<!-- ELSEIF name -->` / `<!-- ELIF name -->` | Else-if on a non-empty value. |
| `<!-- ELSEIFNOT name -->` / `<!-- ELNO name -->` | Else-if on an empty value. |

Conditions apply both to top-level variables (`<!-- IFSET text -->`) and to block fields (`<!-- IFSET item.active -->`). Names may contain letters, digits, `-` and `_` — no spaces, no other punctuation.

> **Important:** `IFSET` tests `!empty()`, not "is defined". A value of `0`, `''`, `false`, `'0'` or an empty array all count as **not set**. Pass `true`/`false` flags for booleans, and never rely on `IFSET` to detect a literal zero.

A repeating list with an empty-state fallback is the standard shape:

```html
<!-- BEGIN item -->
<tr><td>{item.name}</td><td><!-- IFSET item.active -->on<!-- ELSE -->off<!-- ENDIF --></td></tr>
<!-- END item -->
<!-- BEGIN noitem -->
<tr><td colspan="2" class="noitems">{noitems}</td></tr>
<!-- END noitem -->
```

```php
if ( $rows ) foreach ( $rows as $r ) {
	$core->tpl->block( 'body', 'item', [ 'name' => $r['name'], 'active' => $r['active'] ? true : false ] );
} else $core->tpl->block( 'body', 'noitem' );
```

Conditional chains over a block field (the platform's own hacks list shows each hack's state this way):

```html
<!-- BEGIN item -->
<!-- IFNOT item.setup -->
	<span class="{tpl:text-info}"><i class="fa fa-fw fa-cubes"></i> {sset}</span>
<!-- ELSEIF item.active -->
	<b class="{tpl:text-success}"><i class="fa fa-fw fa-check-circle"></i> {son}</b>
<!-- ELSE -->
	<span class="{tpl:text-muted}"><i class="fa fa-fw fa-cubes"></i> {soff}</span>
<!-- ENDIF -->
<!-- END item -->
```

## Class Macros

Write `{tpl:macro}` instead of hard-coded CSS classes. This is the house convention used everywhere in
the platform: a macro is resolved **at compile time** (plain text substitution before the placeholder
pass) to the fixed `core.css` class name, and writing `{tpl:btn-main}` instead of a literal
`btn btn-primary` is part of what keeps a hack indistinguishable from native code. A macro whose name
isn't in the set below resolves to an empty string, so never invent macro names.

The built-in set:

| Macro | Typical use |
|-------|-------------|
| `{tpl:bg}` `{tpl:bg-active}` | Accent / background classes |
| `{tpl:primary}` `{tpl:default}` `{tpl:success}` `{tpl:danger}` `{tpl:warning}` `{tpl:info}` | Color-name roots |
| `{tpl:box-main}` `{tpl:box-primary}` `{tpl:box-default}` `{tpl:box-success}` `{tpl:box-danger}` `{tpl:box-warning}` `{tpl:box-info}` `{tpl:box-widget}` | Panel containers |
| `{tpl:box-header}` `{tpl:box-body}` `{tpl:box-footer}` `{tpl:box-title}` `{tpl:box-tools}` | Panel parts |
| `{tpl:btn-main}` `{tpl:btn-primary}` `{tpl:btn-default}` `{tpl:btn-success}` `{tpl:btn-danger}` `{tpl:btn-warning}` `{tpl:btn-info}` | Buttons |
| `{tpl:btn-block}` `{tpl:btn-sm}` `{tpl:btn-xs}` `{tpl:btn-lg}` | Button modifiers |
| `{tpl:form-control}` `{tpl:form-group}` `{tpl:form-inline}` `{tpl:input-sm}` | Form controls |
| `{tpl:table}` `{tpl:table-main}` `{tpl:table-compact}` `{tpl:table-border}` `{tpl:table-responsive}` | Tables |
| `{tpl:text-muted}` `{tpl:text-primary}` `{tpl:text-success}` `{tpl:text-danger}` `{tpl:text-warning}` `{tpl:text-info}` | Text colors |
| `{tpl:no-padding}` `{tpl:no-margin}` | Spacing helpers |
| `{tpl:alert-main}` `{tpl:alert-active}` `{tpl:alert-success}` `{tpl:alert-warning}` `{tpl:alert-danger}` `{tpl:alert-info}` | Alert / callout boxes |

Always prefer a macro over the literal class. The layout-pattern catalog in
[Template Patterns](hack-tpl-patterns.md#1-class-macros--the-only-classes-you-should-write) lists which
of these you actually reach for in practice.

## Real-World Template Structure

Almost every page is built from "boxes" (panels) and compact tables, expressed entirely through macros. A panel:

```html
<div class="{tpl:box-main}">
	<div class="{tpl:box-header}"><h3 class="{tpl:box-title}">{title}</h3></div>
	<div class="{tpl:box-body}">{text}</div>
</div>
```

A list page — search/add toolbar, a striped bordered table, an empty-state row, and pagination — is the most common layout in the platform:

```html
<div class="{tpl:box-main}"><div class="{tpl:box-body} {tpl:table-responsive} {tpl:no-padding}">
<table class="{tpl:table-compact} {tpl:table-border}">
<tbody>
<!-- BEGIN item -->
	<tr>
		<td align="right">{item.id}</td>
		<td><a href="{item.url}">{item.name}</a></td>
		<td class="{tpl:no-padding}">
			<a href="{item.edit}" class="{tpl:btn-info} {tpl:btn-sm}">{edit}</a>
			<a href="{item.del}" class="{tpl:btn-danger} {tpl:btn-sm}" onclick="return confirm('{confirm}');">{del}</a>
		</td>
	</tr>
<!-- END item -->
<!-- BEGIN noitem -->
	<tr><td class="noitems" colspan="3">{noitems}</td></tr>
<!-- END noitem -->
</tbody>
</table>
</div></div>
<nav class="text-center">{pages}</nav>
```

Templates are not limited to HTML — the same engine drives CSV and other text formats. A CSV export is just a template with one block per row:

```
id;email;pass;
<!-- BEGIN item -->
{item.id};"{item.email}";"{item.pass}";
<!-- END item -->
```

## Page Helper Functions

| Function | Description |
|----------|-------------|
| `$core->site->header()` | Display site header with menus and navigation |
| `$core->site->footer()` | Display site footer (follow with `$core->stop()`) |
| `$core->site->bc( $name, $url )` | Add breadcrumb item |
| `$core->site->pt( $title )` | Set page title |
| `$core->u( $url, $param )` | Generate link from string or array with optional GET params |

## Full Example

```php
$core->site->bc( $core->lang['proxy_h'], $core->u( 'proxy-list' ) );
$core->site->header();

$core->hack->proxy->tpl( 'body', 'proxy-list' );

$core->tpl->vars( 'body', [
	'name'		=> $core->lang['name'],
	'status'	=> $core->lang['status'],
	'noitems'	=> $core->lang['noitems'],
] );

if ( $proxy ) {
	foreach ( $proxy as $p ) {
		$core->tpl->block( 'body', 'item', [
			'name'		=> $p['proxy_name'],
			'status'	=> $core->lang['proxy_status'][$p['proxy_status']],
		] );
	}
} else $core->tpl->block( 'body', 'noitems' );

$core->tpl->output( 'body' );
$core->site->footer();
$core->stop();
```

## Caching and Recompilation

Each template is compiled to PHP in `data/tpls/<hash>.php`. A template is recompiled automatically whenever the source `.tpl` file is newer than its cached copy — editing a template during development takes effect on the next request. If a compiled copy ever goes stale, clear the template cache:

**Management → Settings → Diagnostics → Caching → Page templates → Purge**

## Overriding Templates

**System templates.** A hack can replace built-in templates by shipping files with matching names and listing them in `start.php` under the `tpl` key:

```php
return [
	'tpl' => [ 'analytics-main', 'analytics-lead' ],
];
```

The hack's own file (`hack/<hack>/tpl/<name>.tpl`) then takes precedence over the built-in
the platform's own base and application template directories.
