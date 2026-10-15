# Building Forms with `$core->site->form()`

`$core->site->form()` renders a **complete, ready-to-submit `<form>`** from a typed
array of field definitions. It is the workhorse for the simple, horizontal
"label on the left, control on the right" forms the platform uses everywhere — edit and
add screens, per-section settings, small confirmations, download/action buttons. You
describe *what* fields exist; the builder emits the markup, lays it out on the Bootstrap
grid, and auto-connects the assets each field needs (select2, WYSIWYG, CodeMirror, file
upload).

It is by far the most common way to build a form in AlterCPA, in Lite and Pro alike. This
guide catalogs the patterns.

## What `form()` is — and is not

- It is a **pure UI renderer** over an array. It has **no connection to the database** and
  no knowledge of any table. The field `name`s are just the POST keys *you* choose; you read
  them back from `$core->post` and map them to storage **by hand**. It is used constantly for
  CRUD, but the binding between a field and a column is entirely your code — there is no ORM,
  no schema binding, no auto-save. The same form can write to one table, many tables, a config
  blob, a file, or nothing at all.
- It prints the whole `<form>…</form>` block immediately (it calls `$core->tpl->output()`
  internally). You do **not** queue a body template yourself — `form()` *is* the body.
- Use it for **simple forms**. When you need multiple columns, a table inside the form,
  field-by-field live save, or a bespoke layout, hand-lay a grid template instead — see
  [Multi-column forms](hack-tpl-patterns.md#10-multi-column-forms). Both share the same class
  vocabulary, so you can start with `form()` and graduate later without a visual break.

> The `form()` implementation and its base template are **identical in AlterCPA Lite and
> Pro**. Examples below follow the Lite behaviour;
> Pro is cited where it shows a variant Lite does not.

---

## The signature

```php
$core->site->form( $name, $action, $title, $fields, $button = false );
```

| Arg | Meaning |
|-----|---------|
| `$name` | Internal identifier for this form. Becomes the `<form id>`/`name`, and prefixes every field id as `form_<name>_<field>`. Pick a short slug (`edit`, `news`, `gang`, `set`); it must be **unique among forms on the same page**. It is *not* the URL. |
| `$action` | The POST target, built with `$core->u( … )`. Almost always encodes an `action=<verb>` that the surrounding module dispatches on (see the page pattern below). |
| `$title` | The box header title (a `lang` string). Pass `false` to omit the header — do that when `pt()` already set the page `<h1>`, or the form is embedded. A nullable lang key is often written `$core->lang['x'] ?? false`. |
| `$fields` | Array of field-definition arrays (the body of the form). Detailed below. |
| `$button` | Submit button(s). `false` → a single **Save** button (caption `$core->lang['save']`); a string → a single submit with that caption; an array → several buttons (very rare). |

---

## The page pattern: one function, both directions

The idiom is a single module function that **both processes the submission and renders the
form**. The router switches on `$core->app->action`; a matching case reads `$core->post`,
does whatever persistence you want, and redirects; when no action matches, the form is drawn.
Add and edit share the function, split by an id.

```php
function hack_widget_edit( $core, $id = false ) {

	// --- Handle the submission -------------------------------------------
	switch ( $core->app->action ) {

		case 'edit':
		// Map POST -> whatever storage you like. This mapping is entirely manual;
		// form() knows nothing about DB_WIDGET or these column names.
		$data = [
			'widget_name'   => $core->text->line( $core->post['name'] ),
			'widget_url'    => $core->text->link( $core->post['url'] ),
			'widget_count'  => (int) $core->post['count'],
			'widget_status' => $core->post['status'] ? 1 : 0,
			'widget_descr'  => $core->text->mcode( $core->post['descr'] ),
		];
		if ( $id )	$ok = $core->db->edit( DB_WIDGET, $data, [ 'widget_id' => $id ] );
		else		$ok = $core->db->add( DB_WIDGET, $data );
		$core->go( $core->u( 'widgets', 'message=' . ( $ok ? 'ok' : 'error' ) ) );

	}

	// --- Flash messages from the redirect --------------------------------
	switch ( $core->app->message ) {
		case 'ok':		$core->site->info( 'ok', 'done_basic' ); break;
		case 'error':	$core->site->info( 'error', 'error_basic' ); break;
	}

	// --- Build the form --------------------------------------------------
	if ( $id ) {
		$title = $core->lang['widget_edit'];
		$url   = $core->u( "widgets/$id", 'action=edit' );
		$d     = $core->db->row( "SELECT * FROM ".DB_WIDGET." WHERE widget_id = '$id' LIMIT 1" );
	} else {
		$title = $core->lang['widget_add'];
		$url   = $core->u( 'widgets', 'action=edit' );
		$d     = [ 'widget_status' => 1 ];   // sensible defaults for a new row
	}

	$field = [
		[ 'type' => 'text',     'name' => 'name',   'head' => $core->lang['name'],   'value' => $d['widget_name']  ?? '', 'req' => true ],
		[ 'type' => 'text',     'name' => 'url',    'head' => $core->lang['url'],    'value' => $d['widget_url']   ?? '' ],
		[ 'type' => 'number',   'name' => 'count',  'head' => $core->lang['count'],  'value' => $d['widget_count'] ?? 0, 'min' => 0 ],
		[ 'type' => 'checkbox', 'name' => 'status', 'head' => $core->lang['active'], 'checked' => $d['widget_status'] ?? false ],
		[ 'type' => 'textarea', 'name' => 'descr',  'head' => $core->lang['descr'],  'value' => $d['widget_descr'] ?? '', 'rows' => 4 ],
	];

	// --- Render ----------------------------------------------------------
	$core->site->bc( $core->lang['widgets'], $core->u( 'widgets' ) );
	$core->site->pt( $title );
	$core->site->header();
	$core->site->form( 'edit', $url, false, $field );   // title=false: pt() set the heading
	$core->site->footer();
	$core->stop();

}
```

What to carry away:

- **`$action` is a verb.** The form posts back to the same page with `action=edit`; the
  `switch` at the top catches it. Adding and editing reuse one verb, split by `$id`. Cases
  usually have **no `break`** — each ends in `$core->go()` (which redirects and exits), so
  fall-through never happens.
- **The POST→storage map is yours.** Read every value through the right `$core->text`
  sanitizer (`line`, `link`, `mcode`, `email`, casts, `? 1 : 0`) — never store `$core->post`
  raw. See [Text processing](hack-text.md).
- **Redirect, then flash** (Post/Redirect/Get). On success *and* failure call `$core->go()`
  with a `message=…` query; the reload turns it into a `$core->site->info()` banner, so a
  refresh never re-submits.
- **Prefill with `?? ''`.** In add mode `$d` is a small defaults array, so every field falls
  back cleanly.
- **Standard chrome.** `bc()` / `pt()` / `header()` … `footer()` / `stop()` wrap the form;
  `form()` is the only body call. **Every real call site is wrapped this way** — `form()` is
  not used as a bare AJAX partial. See [Site elements](hack-site.md).

### Render and handle can be two functions

In hacks the two halves are often **separate functions wired to separate routing hooks** —
one builds `$field` + calls `form()`, another reads `$core->post` and redirects. Typically
`hack_<name>_form_admin` (render) and `hack_<name>_save_admin`
(handle) are connected in `start.php` via `$core->handle('footer', …)` and
`$core->handle('actions', …)`, both keyed on `$core->app->div` + `action`. See
[Modules](hack-mod.md).

---

## The field array

`$fields` is an ordered list. Each entry is an associative array whose `type` selects the
control; the remaining keys configure it. Entries render top-to-bottom, so array order *is*
visual order.

```php
[ 'type' => 'text', 'name' => 'name', 'head' => $core->lang['name'], 'value' => $d['name'], 'req' => true ]
```

Two entries are handled specially **before** rendering:

- A **falsy entry** (`false`, `null`, `[]`) is skipped — handy for conditional fields:
  `$upd ? [ 'type' => 'head', … ] : null`.
- **`'skip' => true`** on any entry drops it. Use it to hide a field by condition while keeping
  the array readable: `'skip' => empty( $d['image'] )`.

### Field types

Grouped by how often real code uses them. Every type below is a `case` in the builder;
anything else you pass is silently ignored.

**Layout / static (no input):**

| `type` | Renders | Keys |
|--------|---------|------|
| `head` | `<h4>` section heading — segments a long form | `value` = heading text |
| `line` | `<p>` help/intro paragraph | `value` = text (often `$core->text->lines(...)`) |
| `hidden` | `<input type=hidden>` | `name`, `value` |

**Text family:**

| `type` | Renders | Type-specific keys |
|--------|---------|--------------------|
| `text` | Single-line text input | `value`, `length` (→ `maxlength`), `req`, `ro`, `ph` |
| `email` | `<input type=email>` | `value`, `req`, `ro`, `ph` |
| `pass` | `<input type=password>` | `value`, `length`, `ph` |
| `number` | `<input type=number>` | `value`, `min`, `max`, `step`, `req`, `ro`, `ph` |
| `textarea` | Multi-line textarea | `value`, `rows`, `ro` |

**Choice / boolean:**

| `type` | Renders | Type-specific keys |
|--------|---------|--------------------|
| `checkbox` / `vcheckbox` | One checkbox + bold label (`vcheckbox` is an exact alias) | `checked` (truthy = pre-checked), `value` (submitted value) |
| `select` | Dropdown | `options` (`value=>label`), `value` (selected), `s2` (searchable) |
| `mselect` | Multi-select (select2) | `options` (`value=>label`), `value` (array or CSV), `raw` |
| `radio` | Stacked radios | `options` (`value=>label`), `value` |
| `radioline` | Inline radios | `options` (**list of `['name'=>…,'value'=>…]`**), `value`, `descr` (help block under the radios) |

**Dates:**

| `type` | Renders | Type-specific keys |
|--------|---------|--------------------|
| `date` | `<input type=date>` (native browser picker) | `value` (`Y-m-d`), `ph` |
| `datetime` | `<input type=datetime-local>` | `value` (`Y-m-d\TH:i`), `ph` |

**Files & rich editors:**

| `type` | Renders | Type-specific keys |
|--------|---------|--------------------|
| `file` | File upload (auto-sets `multipart/form-data`) | `name`, `head`, `descr` |
| `mces` | Simple WYSIWYG (Summernote) | `value` |
| `mcea` | Advanced WYSIWYG (Summernote) | `value` |
| `code` | CodeMirror editor (auto-sizes to content — no `rows`) | `value`, `lang` / `mime` (editor mode) |
| `button` | An **action link** styled as a button (`<a>`, GET — not a submit) | `value` (href URL), `text` (caption), `confirm` (JS prompt). Rarely used. |

### Keys common to most input fields

| Key | Applies to | Effect |
|-----|------------|--------|
| `name` | all inputs | The `name="…"` attribute — the POST key you read back. |
| `head` | all | Label to the left of the control. |
| `value` | most | Current value (pre-fill). |
| `descr` | most | Grey `help-block` note under the control. |
| `req` | text/email/number/checkbox | Adds the HTML5 `required` attribute (browser-side validation — no visual asterisk). |
| `ro` | text/email/number/textarea | `readonly`. |
| `ph` | text inputs, date | `placeholder`. |
| `length` | text/pass | `maxlength`. |
| `skip` | any | Omit this entry. |

> **Per-type exceptions.** `pass` honours `ph` but **not** `req`/`ro`. `radioline` honours
> `descr` (a help block under the inline radios), like `radio`. The legacy `color` type is
> **not** wired in the builder — ignore it. `rows` is irrelevant for `code`: CodeMirror
> auto-sizes.

---

## Choices in depth: `select`, `radio`, `mselect`

**Simple convention (the usual one).** Pass `options` as a `value => label` map and `value` as
the current selection. Comparison is loose, so integer keys match string POST values:

```php
[ 'type' => 'select', 'name' => 'status', 'head' => $core->lang['status'],
  'options' => $core->lang['widget_status'],   // e.g. [ 0 => 'Off', 1 => 'On' ]
  'value'   => $d['widget_status'] ?? 0 ],
```

`options` most often comes straight from a `lang` array or a helper (`$core->set->names(...)`,
`$core->currency->codes`, `$core->lang['country']`). A common touch is a leading
**em-dash "none" option**: `[ '' => '&mdash;' ] + $list` or `[ 0 => '&mdash;' ] + $list`.

Add **`'s2' => true`** to turn a long `select` into a searchable select2 box:

```php
[ 'type' => 'select', 's2' => 1, 'name' => 'flow', 'head' => $core->lang['flow'],
  'options' => $flows, 'value' => $d['flow'] ?? 0 ],
```

**`mselect`** is the multi-value variant (always select2, auto-loaded). Its `value` accepts an
array **or** a comma-separated string (DB columns are usually stored as CSV, so a string is the
common case), and it submits as `name[]`:

```php
[ 'type' => 'mselect', 'name' => 'country', 'head' => $core->lang['geo'],
  'options' => $core->lang['country'],       // value => label
  'value'   => $offer['offer_country'] ],      // "RU,UA,KZ" or [ 'RU', 'UA' ]
```

Add **`'raw' => 1`** for free-tag entry — it adds a text box and a toggle so the user can type
raw comma-separated values not in `options` (used heavily in Pro's offer geo/comp pickers).

**Two asymmetries to know:**

- `radioline` does **not** accept a `value=>label` map. Its `options` must already be a list of
  `['name'=>…, 'value'=>…]` dicts. `select`/`radio`/`mselect` convert the map for you;
  `radioline` does not.
- `select`, `radio` and `mselect` also accept a **pre-built option list** via `value` when
  `options` is omitted — each element being `['name'=>…, 'value'=>…, 'select'=>…]`. Rarely
  needed, but that is the fallback path.

---

## Checkboxes and checkbox groups

A single checkbox: `checked` (any truthy value) pre-checks it; read it back with a truthiness
test.

```php
// build:  'checked' => $d['widget_status'] ?? false
// save:   'widget_status' => $core->post['status'] ? 1 : 0
```

A **group** that submits into one array: give every box the same `name="key[]"` and a distinct
`value`. A flag matrix built entirely in a loop:

```php
$field = [[ 'type' => 'line', 'value' => $core->lang['myhack_edit'] ]];
foreach ( $group as $g => $title ) {
	if ( empty( $core->lang["myhack_$g"] ) ) continue;        // skip an empty section
	$field[] = [ 'type' => 'head', 'value' => $title ];       // section divider
	foreach ( $core->lang["myhack_$g"] as $k => $v ) {
		$active = in_array( $k, $cc );
		$field[] = [ 'type' => 'checkbox', 'name' => 'basic[]', // shared name[]
		             'head' => $v, 'value' => $k, 'checked' => $active ];
	}
}
// read back:  foreach ( (array) $core->post['basic'] as $k ) ...
```

For an **indexed** group (one value per id), use `name="key[$id]"` — e.g. a per-variant price
grid, `'name' => "prcs[$vi]"`, read back as `$core->post['prcs'][$vi]`.

---

## Hidden fields, read-only display fields

Two idioms recur across Pro:

- **Referer capture** — carry the page the user came from through the POST so the handler can
  return there:
  `[ 'type' => 'hidden', 'name' => 'r', 'value' => $core->server['HTTP_REFERER'] ]`.
- **Read-only display** — a `text` field with `'ro' => 1` used purely to *show* a value the
  user copies (a postback URL, an API token, a generated link). It is not really input; it just
  reuses the form row layout:
  `[ 'type' => 'text', 'ro' => 1, 'head' => $core->lang['postback'], 'value' => $url ]`.

---

## File uploads

Add a `file` field and the builder switches the whole `<form>` to
`enctype="multipart/form-data"` automatically. Read the upload from `$core->files`, not
`$core->post`:

```php
[ 'type' => 'file', 'name' => 'image', 'head' => $core->lang['image'], 'descr' => 'JPEG 400x250px' ],
// handler:
if ( ! empty( $core->files['image']['name'] ) ) {
	$ext = $core->media->ext( $core->files['image']['name'] );
	$core->media->upload( 'news', $core->files['image'], "$id.$ext" );
}
```

A common companion is a "remove image" checkbox that only appears when an image exists —
`'skip' => empty( $d['image'] )`. See [Media library](hack-media.md).

---

## Rich editors

- **`mces` / `mcea`** — Summernote WYSIWYG, simple and advanced toolbars. Just `name`/`value`.
  Save with `$core->text->mcode()`.
- **`code`** — a CodeMirror editor that **auto-sizes to its content**. Set the language mode
  with `lang` (and optionally `mime`); the builder loads a CodeMirror mode file per comma-
  separated language and the editor uses `mime` (falling back to `lang`) as its mode:

  ```php
  [ 'type' => 'code', 'name' => 'add_pre', 'head' => $core->lang['php_before'],
    'lang' => 'clike,php', 'mime' => 'text/x-php', 'value' => $d['add_pre'] ],
  ```

  Real forms put **many** code fields on one screen — Pro's integration editor has about a
  dozen. A bare `[ 'type' => 'code', 'name' => 'code' ]` (no `lang`) falls back to a generic
  mode. Save with `$core->text->code()` / `codeline()`.

> Note: full-page code editors (a single large source file) are usually **not** built with
> `form()` — they hand-roll a CodeMirror page from a dedicated template. Use the `code` field
> for code that is *one input among several* in an ordinary form.

---

## Buttons (the submit row)

The 5th argument controls the submit row in the box footer:

```php
$core->site->form( 'edit', $url, $title, $field );                        // one "Save" button (default)
$core->site->form( 'edit', $url, $title, $field, $core->lang['create'] );  // one button, custom caption
```

A custom caption is common for non-"save" actions — `download`, `import`, `create`, `join`, `do`.

The **array form** (multiple buttons) is supported, though no screen of Lite or Pro uses it. If
you need it, each entry takes
`type` (`submit`/`button`, default `button`), `name`, `value` (caption), `cls`:

```php
$core->site->form( 'edit', $url, $title, $field, [
	[ 'type' => 'submit', 'name' => 'act', 'value' => $core->lang['save'] ],
	[ 'type' => 'submit', 'name' => 'act', 'value' => $core->lang['delete'] ],
] );
// (In the base Lite template the button's `cls` is ignored — the footer button is styled btn-main.)
```

Do not confuse this with the **`button` field type** (a GET action *link* inside the form body,
with `value` = href, `text` = caption, `confirm` = JS prompt). It exists but is virtually never
used — a "form that is really just a button" is normally a normal form with a single `line`
field and a custom submit caption — here, one that triggers a file download:

```php
$core->site->form( 'export', $core->u( $core->app->pp, 'action=download' ),
	$core->lang['myhack_export_h'],
	[[ 'type' => 'line', 'value' => $core->lang['myhack_export_t'] ]],
	$core->lang['download'] );
```

---

## Assets are automatic

You never connect editor/picker assets for a `form()` — the builder detects the field types it
rendered and pulls in exactly what is needed, before the header:

| Field present | Auto-connected |
|---------------|----------------|
| `file` | `enctype="multipart/form-data"` on the `<form>` |
| `select` with `s2`, or `mselect` | select2 (`$this->set('select2')`) |
| `mces` / `mcea` | Summernote (simple / advanced toolbar) |
| `code` | CodeMirror + a language-mode file per `lang` token |

(`date`/`datetime` render as native HTML5 inputs — no JS picker is wired in the base skin.)

---

## Dynamic and conditional field building

Because `$fields` is a plain array, build it however the data dictates. The techniques, in
increasing order of power:

- **`skip`** — drop one entry by condition: `'skip' => empty( $d['image'] )`.
- **Falsy entry** — `$isAdmin ? [ 'type' => 'checkbox', … ] : false` (the builder drops `false`/`null`).
- **`continue`** — decide whether to *append* at all inside a loop.
- **Append in a loop** — `$field[] = [ … ]` per row of data (checkbox groups, price grids).
- **Whole-form branches** — `if ( $vars ) { …per-variant fields… } else { …flat fields… }`.
- **Merge extra fields** — `$field = array_merge( $field, $worker->form() )` to let a sub-component
  contribute its own fields (ad-network workers, user meta).
- **Filter hook** — let other code mutate the array before render:
  `if ( $core->cando('users_one_form') ) $field = $core->filter( 'users_one_form', $field, $u );`.

```php
$field = [ [ 'type' => 'line', 'value' => $core->lang['intro'] ] ];
$field[] = [ 'type' => 'head', 'value' => $core->lang['basics'] ];
$field[] = [ 'type' => 'text', 'name' => 'name', 'head' => $core->lang['name'], 'value' => $d['name'] ?? '', 'req' => true ];
foreach ( $channels as $k => $label )                       // one checkbox per channel
	$field[] = [ 'type' => 'checkbox', 'name' => 'ch[]', 'head' => $label, 'value' => $k, 'checked' => in_array( $k, $on ) ];
$field[] = [ 'type' => 'checkbox', 'name' => 'noimage', 'head' => $core->lang['del_image'], 'skip' => empty( $d['image'] ) ];
```

`head` and `line` entries are your section structure — `head` is a bold `<h4>` divider, `line`
an explanatory paragraph. Use them to group a long form into readable blocks.

---

## Multiple forms on one page

Nothing stops you calling `form()` several times between one `header()` / `footer()`. Pro's
utility and offer-site pages render five or more independent forms in a row, each guarded by a
data condition; the skin presents each as its own card/modal:

```php
$core->site->header();
if ( $domains ) $core->site->form( 'a', $urlA, $core->lang['a_h'], $fieldA );
if ( $urls )    $core->site->form( 'b', $urlB, $core->lang['b_h'], $fieldB );
                $core->site->form( 'c', $urlC, $core->lang['c_h'], $fieldC );
$core->site->footer();
$core->stop();
```

Each form needs a **distinct `$name`** (it becomes the `<form id>` and the field-id prefix), and
each posts to its own `action` verb handled by the same module.

---

## Reading the POST back (the CRUD side)

`form()` gives you inputs; turning them into stored data is ordinary handler code. The recurring
shape:

- **Read + sanitize** through `$core->text` — `line` (plain), `link` (URL), `email`, `mcode`
  (WYSIWYG), `code`/`codeline` (source), `decimal` (money), plus `(int)` casts and
  `$core->post['x'] ? 1 : 0` for checkboxes. Files come from `$core->files`.
- **Validate before saving where it matters** — e.g. check an IP is public, an e-mail is unique,
  a required field is non-empty; on failure `$core->go( …'message=error' )` *before* writing.
- **Persist however you like** — `$core->db->add/edit/del`, a library method, a config blob,
  a file. There is no prescribed target; the form does not care.
- **Redirect with a `message=`**, then map it to a flash on reload (`$core->site->info`).

For nested resources, build the action URL with the array form of `$core->u`:
`$core->u( [ 'offer', $id, 'vars', $subid ], 'action=edit' )`.

---

## Generated (schema-driven) forms

The Settings subsystem builds `form()` field arrays **from a config schema** rather than by hand
— it maps each option's declared widget type to a field entry (`decimal`→`number`+`step`,
`info`→read-only `text`, etc.) and calls `form()` once
with the whole list. Note that config schemas use a **second, separate type vocabulary** for
*storage/sanitisation* (`bool`, `int`, `decimal`, `link`, `email`, `code`, `ints`, `codes`, …)
applied when reading the POST back — do not confuse it with the form field types here. If your
hack adds a Settings page, you feed that schema rather than call `form()` directly; see
[Configuration page](hack-config.md).

(`$core->site->config()` is a different method — it renders the settings *index* grid of cards,
not an input form.)

---

## Rendered markup, for reference

The generated `<form>` is a Bootstrap **horizontal** form (`form form-horizontal`), label in a
`col-sm-2` column, control in `col-sm-10`. Facts worth knowing:

- The `<form>` carries `id`/`name` = your `$name`; each control's id is `form_<name>_<field>`.
- The box header (`<h3>` with `$title`) only appears when `$title` is truthy.
- `descr` renders as `<p class="help-block">` under the control; `req` becomes
  `required="required"`; `ro` becomes `readonly`.
- The submit row sits in a right-aligned `box-footer`.

If you need anything the generator does not express — columns, a table inside the form, inline
live-save, a custom footer — drop to a hand-written template
([Multi-column forms](hack-tpl-patterns.md#10-multi-column-forms)); the class vocabulary is the same.

---

## Gotchas

- **No DB binding.** `form()` renders inputs; it never reads or writes storage. The map from
  `name` keys to columns is entirely your handler code.
- **`$name` must be unique per page** and is not the URL — it is the form id / field-id prefix.
- **`title => false`** removes the box header; do that when `pt()` already set the heading.
- **`code` ignores `rows`** — CodeMirror auto-sizes. Don't bother setting it.
- **`date`/`datetime` are native inputs** in the base skin — no JS datepicker is attached; feed
  `Y-m-d` / `Y-m-d\TH:i` values.
- **`radioline` options** are a list of `['name'=>,'value'=>]`, *not* a `value=>label` map.
- **`color` is legacy** — present in the template, not wired in the builder. Do not use it.
- **`pass`** ignores `req`/`ro` (only `ph`/`length` apply).

---

## See also

- [Site elements](hack-site.md) — `$core->site` header/footer, flash `info()`, page flow.
- [Template Patterns cookbook](hack-tpl-patterns.md) — hand-laid grid forms, toolbars, modals.
- [Text processing](hack-text.md) — sanitizers for reading `$core->post` back safely.
- [Media library](hack-media.md) — handling `file` uploads.
- [Configuration page](hack-config.md) — schema-driven Settings forms.
- [Modules](hack-mod.md) — where the router / `action` dispatch and hack routing hooks live.
