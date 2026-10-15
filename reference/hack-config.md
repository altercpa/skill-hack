# Configuration page

The configuration page is the hack's own control panel. It is opened by the **Settings** button next to the hack in *Control → Hacks*, and for many hacks it is the entire user interface — it both configures the hack and, often, fully implements its functionality (lists, forms, actions).

It is handled by a single function, `hack_<name>_config`, placed in `mod/config.php`. The function is optional: when it is absent, the platform shows an auto-generated form built from `param.json` (or a raw key/value editor). Create the config page when you need a richer UI than that fallback.

## How it is invoked

The **Settings** button links to `hacks/<name>`. The platform handles that page itself and, near the end, when the hack is active, calls `$core->hack-><name>->mod( 'config', 'hack_<name>_config' )`.

So:

- The page is wired up automatically — there is **nothing to register** in `start.php`. Just create `mod/config.php` with the correctly named function.
- The function is called **only when the hack is active** (installed *and* enabled). For an inactive hack the Settings button is disabled and the generic param form is shown instead.
- `mod( 'config', 'hack_<name>_config' )` includes `mod/config.php` and calls `hack_<name>_config( $core )`.
- Your function **must take over the page** by stopping execution — via `$core->go()`, `$core->json()`, or `$core->site->footer()` followed by `$core->stop()`. If it returns without stopping, execution falls through to the platform's generic param form below it.

### Reserved actions

The platform router processes its own actions **before** calling your function, each ending in a redirect. If `$core->app->action` matches one of these, your function never runs for that request:

`upd`, `param`, `export`, `install`, `toggle`, `schema`

Never reuse these names for your own actions. Namespace every action with a short prefix — `myhack-conf`, `myhack-make`, `myhack-kill` — as the existing hacks do.

### ⚠ `param` destroys every key you did not declare

The reserved `param` action is the **Save** button of the platform's generic settings form, and it does not merge. It rebuilds the configuration array from the `param.json` field list alone — one entry per declared field, typed by that field's `val` — and writes the result over `hack_config` whole. Anything the array held that `param.json` does not name is gone the moment anyone presses Save, including a save that changed nothing. (A hack with no `param.json` gets the raw key/value editor instead — also a wholesale rewrite, from whatever rows the form happened to render.)

That is fatal for **runtime-generated state** — a derived secret, a random seed, a token minted on first use, a cached remote id. Written with `conf( $field, $value )` from your own code, it lives in that same array, undeclared, and the next Save deletes it. If the value cannot be regenerated identically, whatever depended on it breaks.

So:

- **Settings a human edits** → `param.json` fields, or your own config page writing `conf()`, and every one of them declared.
- **State the code generates and reads back** → a `$core->config()` block. That is `DB_CONFIG`, a different table entirely, which the `param` action never touches. See [The Core → Configuration](hack-core.md#configuration).

A hack with its own `mod/config.php` is not exempt: `param` is handled by the router *before* your function is reached, so the Settings button on the hacks list can still fire it.

## Function skeleton

```php
<?php

/*******************************************************************************

 *
 *  AlterCPA Core
 *  Created by AlterVision - www.altercpa.pro
 *  Copyright (c) 2013-2026 Anton Reznichenko
 *

 *
 *  File: 			hack / myhack / mod / config.php
 *  Description:	MyHack configuration
 *  Author:			Anton 'AlterVision' Reznichenko - altervision13@gmail.com
 *

*******************************************************************************/

// MyHack configuration
function hack_myhack_config( $core ) {

	// 1. Load current configuration and reference data
	$conf = $core->hack->myhack->conf;
	$offer = $core->cpa->get( 'offersa' );

	// 2. Process incoming actions (each case redirects or stops)
	switch ( $core->app->action ) {

		case 'save-conf':
		$core->hack->myhack->conf( 'active', $core->post['active'] ? 1 : 0 );
		$core->go($core->u( $core->app->pp, 'message=ok' ));

	}

	// 3. Render the page
	$core->site->pt( $core->lang['myhack_h'] );
	$core->site->header();
	$core->hack->myhack->tpl( 'body', 'config' );
	$core->tpl->vars( 'body', [
		'u_save'	=> $core->u( $core->app->pp, 'action=save-conf' ),
		'active'	=> $conf['active'],
		'save'		=> $core->lang['save'],
	]);
	$core->tpl->output( 'body' );
	$core->site->footer();
	$core->stop();

}
```

The three numbered stages — **load → act → render** — are the shape of every config page in the platform.

## Routing and the self URL

While the config page is open, the routing object describes `hacks/<name>`:

| Property | Value on the config page |
|----------|--------------------------|
| `$core->app->div` | `hacks` |
| `$core->app->item` | the hack name |
| `$core->app->pp` | `hacks/<name>` — the full self path |
| `$core->app->action` | the current action (GET or POST `action`) |
| `$core->app->message` | the `message` GET param after a redirect |

Build every link and form target back to the page itself with `$core->u( $core->app->pp, ... )`. The second argument may be a query string or an array:

```php
$core->u( $core->app->pp, 'action=myhack-make' );
$core->u( $core->app->pp, [ 'action' => 'myhack-form', 'uid' => $i ] );
```

Actions are accepted only when the request referer matches the host (built-in CSRF protection), so always reach actions from a form or link rendered on the page.

## Reading and writing configuration

Each active hack carries its stored settings in the `conf` property of its main object — **one array holding every setting of the hack**, serialized into the single `hack_config` column of the hacks table (see [Main class](hack-main.md)).

> **All of the hack's settings live in that one array.** There is no second store and no schema for settings: do not add a `cpa_x<hack>_config` table, do not add a database column per setting, do not spread one scope's settings over several `$core->config()` records (a block *per scope* — per user, per team — is the one legitimate multiplication; see [The Core → Per-user and per-team settings](hack-core.md#configuration)). `conf( $field, $value )` looks like a per-field setter, but it only assigns one key of the array and rewrites the whole `hack_config` value — the array is the unit of storage. The hack's own tables are for its **data** — the rows it produces or manages ([Database](hack-db.md)); its settings are one array. This is the mistake to watch for: a settings page with twelve inputs is twelve keys of `conf`, not twelve columns.

```php
// Read — conf is a plain array
$active = $core->hack->myhack->conf['active'] ?? 0;
$push   = $core->hack->myhack->conf['push'] ?? [];

// Write a single field — persists to the DB and refreshes the cache
$core->hack->myhack->conf( 'active', 1 );
$core->hack->myhack->conf( 'push', $push );   // values may be arrays
```

Nested structure belongs inside the same array: a list of notification rules is `conf['rule']`, an array of arrays, rewritten whole on every change — see the [complete example](#complete-example) below.

`conf( $name, $value )` saves immediately and updates the in-memory copy; there is no separate "save all" call. Inside a custom main class you may also expose a typed helper (a `main()` method, say) that maps the whole form to several `conf()` calls.

**Sanitize every input** before storing it (request values are pre-escaped for SQL but still need typing/validation):

```php
$b = [
	'comp'		=> (int) $core->post['comp'],
	'cron'		=> $core->text->link( $core->post['cron'] ),
	'limit'		=> max( 0, (int) $core->post['limit'] ),
	'offer'		=> $core->text->ints( $core->text->intline( $core->post['offer'] ) ),
	'geo'		=> $core->text->codes( $core->text->codeline( $core->post['geo'] ) ),
	'host'		=> $core->text->line( $core->post['host'] ),
];
```

### Input sanitizers

`$core->text->*` is the standard toolbox for cleaning form input before storage and for escaping stored values on output. The ones you reach for on a config page:

| Method | Use |
|--------|-----|
| `link( $s )` | Slug — lowercase letters, digits, hyphen (action names, codes, cron keys) |
| `line( $s )` | Single-line text, tags stripped (hosts, URLs, names) |
| `lines( $s )` | Multi-line text, normalized newlines (textareas) |
| `intline( $s )` / `ints( $s )` | Newline/comma list of integers → cleaned string / array of ints |
| `codeline( $s )` / `codes( $s )` | Same for slug codes (countries, statuses) → string / array |
| `float( $n )` / `decimal( $n, $round )` | Numbers; `decimal` rounds to N places |
| `email( $s )` / `url( $s )` / `domain( $s )` | Validate and normalize those formats |
| `metasafe( $s )` | **Output** — escape a stored value for an HTML attribute (`value="{x}"`) |
| `code( $s )` | **Output** — keep code/HTML intact (template/source editors) |

Casts stay inline: `(int)`, `(string)`, and `max()`/`min()` clamps for ranges. For request values fed into string functions use `tostr( $core->post['x'] ?? '' )` (see [Code Style](hack-style.md)). Pair list cleaners — `$core->text->ints( $core->text->intline( $in ) )` — to go from raw textarea to a clean array of ints.

## Rendering with templates

The page is drawn with the hack's own templates from `tpl/` (see [Templates and styling](hack-tpl.md)):

```php
$core->site->pt( $core->lang['myhack_h'] );	// page title
$core->site->set( 'select2' );				// asset set: searchable selects
$core->site->js( 'jquery.tablesorter' );	// extra system script (before header)
$core->site->header();

$core->hack->myhack->tpl( 'body', 'myhack' );	// load tpl/myhack.tpl into part 'body'
$core->tpl->vars( 'body', [ /* scalar vars + labels */ ] );

// Repeating rows
if ( $box ) foreach ( $box as $i => $b ) {
	$core->tpl->block( 'body', 'item', [ 'id' => $i, /* ... */ ] );
} else $core->tpl->block( 'body', 'noitem' );

$core->tpl->output( 'body' );
$core->site->footer();
$core->stop();
```

Populate select lists from CPA reference data and language tables. `$core->cpa->get( $key, $id )` is the cached gateway to platform entities; most keys return `id => name` maps ready for a `<select>`:

| `cpa->get()` key | Returns |
|------------------|---------|
| `offersa` | All offers, `id => name` |
| `offers` | Active **top-level** offers (`offer_active=1 AND comp_id=0`), `id => name` |
| `compa` | All companies / sources, `id => name` |
| `comps` | Basic-type companies only, `id => name` |
| `ccs` | Call centers, `id => name` |
| `coffersa` / `coffers` | Offers of company `$id` (all / active) — needs `$id` |
| `sites` | **Full** landing-site rows for offer `$id` — needs `$id`. For an `id => url` map use `siteurls` instead |
| `flows` | Traffic flows of user `$id` — **needs a user `$id`** |
| `advs` | Advertising networks of user `$id` — **needs a user `$id`** |
| `shpts` | Shipment methods of company `$id` — needs `$id` |
| `gangs` | Teams (gangs), `id => name` |
| `offer` / `oinfo` / `ofp` | One offer: full row / short row / parameters array (needs `$id`) |
| `comp` | One company row (needs `$id`) |

> **AlterCPA Lite caveat.** `$core->cpa` exists in Lite too, but its `get()` only
> implements **`gangs`** (plus `filter`/`filters`) — every other key above returns
> **`false`** on Lite. The offer/company/flow/site reference data is **AlterCPA Pro
> only**. A Lite hack must source its select options elsewhere (its own tables or
> language lists). Treat this whole reference-data table as Pro-specific.

Language tables provide the static lists: `$core->lang['country']` (`code => name`), `$core->lang['status']`, and any list you define in your own `lang/` files (see [Language files](hack-lang.md)).

```php
foreach ( $offer as $v => $n )
	$core->tpl->block( 'body', 'offer', [ 'v' => $v, 'n' => $n, 's' => in_array( $v, $sel ) ] );
```

## Actions, saving and feedback

Handle every action in the opening `switch ( $core->app->action )`. The typical case validates input, calls `conf()`, then redirects back to the page so a refresh cannot re-submit the form:

```php
switch ( $core->app->action ) {

	// Save / add an item
	case 'myhack-make':
	$uid = (int) ( $core->get['uid'] ?? -1 );
	$b   = [ /* sanitized fields */ ];
	if (isset( $box[$uid] )) {
		$box[$uid] = $b;
	} else $box[] = $b;
	$core->hack->myhack->conf( 'box', $box );
	$core->go($core->u( $core->app->pp, 'message=ok' ));

	// Delete an item
	case 'myhack-kill':
	$uid = (int) ( $core->get['uid'] ?? -1 );
	if (isset( $box[$uid] )) {
		unset( $box[$uid] );
		$core->hack->myhack->conf( 'box', $box );
		$core->go($core->u( $core->app->pp, 'message=ok' ));
	} else $core->go($core->u( $core->app->pp, 'message=error' ));

}
```

Note the fall-through `switch` with no `break`: each case ends in `$core->go()`, which redirects and stops. After the redirect, read `$core->app->message` to show a banner with `$core->site->info( $type, $text )` — type is `ok`, `error`, `info` or `text`; a `$text` that matches a language key is auto-translated:

```php
switch ( $core->app->message ) {
	case 'ok':		$core->site->info( 'ok', 'done_basic' ); break;
	case 'error':	$core->site->info( 'error', 'error_basic' ); break;
}
```

## AJAX fragments and modal forms

For per-item editing, render the edit form as a **fragment** — no header or footer — and load it into a modal with JavaScript. The same function serves the fragment under a dedicated action:

```php
// Show the edit form (inside the opening switch)
case 'myhack-form':
$uid = (int) ( $core->get['uid'] ?? -1 );
$b   = $box[$uid] ?? $def;
$core->hack->myhack->tpl( 'form', 'myhack-form' );
$core->tpl->vars( 'form', [ 'u_save' => $core->u( $core->app->pp, [ 'action' => 'myhack-make', 'uid' => isset($box[$uid]) ? $uid : null ] ), /* ... */ ]);
foreach ( $offer as $ai => $av ) $core->tpl->block( 'form', 'offer', [ 'v' => $ai, 'n' => $av, 's' => in_array($ai,$off) ]);
$core->tpl->output( 'form' );
$core->stop();   // no header/footer — just the form HTML
```

The list page requests it over AJAX and drops the HTML into a modal:

```javascript
$.ajax({ url: $("#editbtn"+uid).data("url"), type: "GET", dataType: "text",
	success: function( data ) { $("#theformcontent").html( data ); }
});
```

For silent auto-save, post the form with an extra `&ajax=1` flag and answer with JSON instead of a redirect:

```php
case 'myhack-conf':
$core->hack->myhack->main( $core->post );
if ( $core->get['ajax'] ) $core->json([ 'status' => 'ok' ]);
$core->go($core->u( $core->app->pp, 'message=ok' ));
```

`$core->json( $data )` prints the JSON and stops.

## Page helper reference

| Call | Purpose |
|------|---------|
| `$core->site->pt( $title )` | Set the page title |
| `$core->site->set( $name )` | Add an asset set — before the header. Useful sets: `select2` (searchable multi-selects, auto-inits `.select2`), `tablesorter` (sortable tables, auto-inits `.tablesorter`), `summernote` (rich text), `adminlte`, `bootstrap` |
| `$core->site->js( $name )` / `css( $name )` | Add a system script / style — before the header |
| `$core->hack-><name>->js( $name )` / `css( $name )` | Add a script / style from the hack |
| `$core->site->header()` / `footer()` | Render page chrome |
| `$core->site->info( $type, $text )` | Queue an alert banner |
| `$core->hack-><name>->tpl( $part, $file )` | Load a hack template into a part |
| `$core->tpl->vars / block / output` | Fill and render the template |
| `$core->u( $path, $param )` | Build a URL (string or array params) |
| `$core->go( $url )` | Redirect and stop |
| `$core->msgo( $msg, $url )` | Redirect, setting a `message` param, and stop |
| `$core->json( $data )` | Output JSON and stop |
| `$core->stop()` | End the request (after `footer()`) |

## Complete example

A minimal but real config page for a hack named `notifier`: one global toggle, plus a list of notification rules each editable in a modal. Three files.

**`mod/config.php`**

```php
<?php

/*******************************************************************************

 *
 *  AlterCPA Core
 *  Created by AlterVision - www.altercpa.pro
 *  Copyright (c) 2013-2026 Anton Reznichenko
 *

 *
 *  File: 			hack / notifier / mod / config.php
 *  Description:	Notifier configuration
 *  Author:			Anton 'AlterVision' Reznichenko - altervision13@gmail.com
 *

*******************************************************************************/

// Notifier configuration
function hack_notifier_config( $core ) {

	// Load configuration and reference data
	$offer = $core->cpa->get( 'offersa' );
	$rule  = $core->hack->notifier->conf['rule'] ?? [];
	$def   = [ 'email' => '', 'offer' => [] ];

	// Process actions
	switch ( $core->app->action ) {

		// Toggle the whole hack on/off
		case 'note-active':
		$core->hack->notifier->conf( 'active', $core->post['active'] ? 1 : 0 );
		$core->go($core->u( $core->app->pp, 'message=ok' ));

		// Save or add a rule
		case 'note-make':
		$uid = (int) ( $core->get['uid'] ?? -1 );
		$off = $core->text->intline( $core->post['offer'] );
		$b = [
			'email'	=> $core->text->email( $core->post['email'] ),
			'offer'	=> $off ? $core->text->ints( $off ) : [],
		];
		if ( ! $b['email'] ) $core->go($core->u( $core->app->pp, 'message=error' ));
		if (isset( $rule[$uid] )) {
			$rule[$uid] = $b;
		} else $rule[] = $b;
		$core->hack->notifier->conf( 'rule', $rule );
		$core->go($core->u( $core->app->pp, 'message=ok' ));

		// Delete a rule
		case 'note-kill':
		$uid = (int) ( $core->get['uid'] ?? -1 );
		if (isset( $rule[$uid] )) {
			unset( $rule[$uid] );
			$core->hack->notifier->conf( 'rule', $rule );
		}
		$core->go($core->u( $core->app->pp, 'message=ok' ));

		// Serve the edit form fragment for the modal
		case 'note-form':
		$uid = (int) ( $core->get['uid'] ?? -1 );
		$b   = $rule[$uid] ?? $def;
		$core->hack->notifier->tpl( 'form', 'notifier-form' );
		$core->tpl->vars( 'form', [
			'u_save'	=> $core->u( $core->app->pp, [ 'action' => 'note-make', 'uid' => isset($rule[$uid]) ? $uid : null ] ),
			'title'		=> $core->lang['notifier_edit'],
			'email'		=> $core->lang['notifier_email'],
			'offer'		=> $core->lang['offers'],
			'save'		=> $core->lang['save'],
			'emails'	=> $core->text->metasafe( $b['email'] ),
		]);
		foreach ( $offer as $v => $n ) $core->tpl->block( 'form', 'offer', [ 'v' => $v, 'n' => $n, 's' => in_array( $v, $b['offer'] ) ] );
		$core->tpl->output( 'form' );
		$core->stop();

	}

	// Show the message banner after a redirect
	switch ( $core->app->message ) {
		case 'ok':		$core->site->info( 'ok', 'done_basic' ); break;
		case 'error':	$core->site->info( 'error', 'error_basic' ); break;
	}

	// Render the page
	$core->site->pt( $core->lang['notifier_h'] );
	$core->site->set( 'select2' );
	$core->site->header();

	$core->hack->notifier->tpl( 'body', 'notifier' );
	$core->tpl->vars( 'body', [
		'text'		=> $core->text->lines( $core->lang['notifier_t'] ),
		'u_active'	=> $core->u( $core->app->pp, 'action=note-active' ),
		'u_add'		=> $core->u( $core->app->pp, 'action=note-form' ),
		'active'	=> $core->hack->notifier->conf['active'],
		'enabled'	=> $core->lang['notifier_on'],
		'email'		=> $core->lang['notifier_email'],
		'offer'		=> $core->lang['offers'],
		'action'	=> $core->lang['action'],
		'edit'		=> $core->lang['edit'],
		'del'		=> $core->lang['del'],
		'confirm'	=> $core->lang['confirm'],
		'noitems'	=> $core->lang['noitems'],
	]);

	if ( $rule ) foreach ( $rule as $i => $b ) {
		$oc = count( $b['offer'] ?? [] );
		$core->tpl->block( 'body', 'item', [
			'id'	=> $i,
			'email'	=> $core->text->metasafe( $b['email'] ),
			'offer'	=> $oc == 1 ? $offer[$b['offer'][0]] : numinf( $core->lang['notifier_offers'], $oc ),
			'edit'	=> $core->u( $core->app->pp, [ 'action' => 'note-form', 'uid' => $i ] ),
			'del'	=> $core->u( $core->app->pp, [ 'action' => 'note-kill', 'uid' => $i ] ),
		]);
	} else $core->tpl->block( 'body', 'noitem' );

	$core->tpl->output( 'body' );
	$core->site->footer();
	$core->stop();

}
```

**`tpl/notifier.tpl`** — the list page, with a checkbox that auto-submits and a modal loaded over AJAX:

```html
<div class="entry">{text}</div>

<form action="{u_active}" method="post" id="noteactive">
<div class="checkbox form-group"><label>
	<input type="checkbox" name="active" value="1" class="editable" <!-- IFSET active -->checked<!-- ENDIF --> /> <b>{enabled}</b>
</label></div>
</form>

<div class="{tpl:form-group}"><a class="{tpl:btn-main} {tpl:btn-block}" href="#" data-url="{u_add}" id="editbtn-1" onclick="return editform(-1);">+ {email}</a></div>

<div class="{tpl:box-main}"><div class="{tpl:box-body} {tpl:no-padding} {tpl:table-responsive}">
<table class="{tpl:table-compact} {tpl:table-border}">
<thead><tr><th>{email}</th><th>{offer}</th><th width="3%">{action}</th></tr></thead>
<tbody>
<!-- BEGIN item -->
<tr>
	<td>{item.email}</td>
	<td align="center">{item.offer}</td>
	<td class="{tpl:no-padding} compact-td">
		<a href="#" class="{tpl:btn-info} {tpl:btn-sm}" id="editbtn{item.id}" data-url="{item.edit}" onclick="return editform('{item.id}');"><i class="fa fa-fw fa-edit"></i></a>
		<a href="{item.del}" class="{tpl:btn-danger} {tpl:btn-sm}" onclick="return confirm('{confirm}');"><i class="fa fa-fw fa-trash"></i></a>
	</td>
</tr>
<!-- END item -->
<!-- BEGIN noitem -->
<tr><td class="noitems" colspan="3">{noitems}</td></tr>
<!-- END noitem -->
</tbody>
</table>
</div></div>

<div class="modal fade" id="noteform" style="display: none;"><div class="modal-dialog"><div class="modal-content" id="noteformbody">
<div class="modal-body"><div class="progress active"><div class="progress-bar progress-bar-primary progress-bar-striped" style="width: 100%"></div></div></div>
</div></div></div>

<script type="text/javascript">
	$("#noteactive .editable").change( () => $("#noteactive").submit() );
	const dummy = $("#noteformbody").html();
	function editform( uid = 0 ) {
		$('#noteform').modal('show');
		$("#noteformbody").html( dummy );
		$.ajax({ url: $("#editbtn"+uid).data("url"), type: "GET", dataType: "text",
			success: function( data ) { $("#noteformbody").html( data ); $("#noteoffers").select2({ width: "100%", dropdownParent: $('#noteform') }); },
			error: function() { $('#noteform').modal('hide'); }
		});
		return false;
	}
</script>
```

**`tpl/notifier-form.tpl`** — the fragment rendered into the modal:

```html
<form action="{u_save}" method="post">
	<div class="modal-header">
		<button type="button" class="close" data-dismiss="modal"><span aria-hidden="true">×</span></button>
		<h4 class="modal-title">{title}</h4>
	</div>
	<div class="modal-body">
		<div class="form-group">
			<label for="noteemail">{email} <span class="text-danger">*</span></label>
			<input type="email" name="email" id="noteemail" class="{tpl:form-control}" value="{emails}" required />
		</div>
		<div class="form-group">
			<label for="noteoffers">{offer}</label>
			<select name="offer[]" id="noteoffers" class="{tpl:form-control}" multiple>
<!-- BEGIN offer -->
				<option value="{offer.v}" <!-- IFSET offer.s -->selected<!-- ENDIF -->>{offer.n}</option>
<!-- END offer -->
			</select>
		</div>
	</div>
	<div class="modal-footer"><button type="submit" class="{tpl:btn-main} {tpl:btn-block}">{save}</button></div>
</form>
```

The matching language keys (`notifier_h`, `notifier_t`, `notifier_edit`, `notifier_email`, `notifier_on`, `notifier_offers`, …) go in the hack's `lang/en.php` and `lang/ru.php`; shared labels (`save`, `edit`, `del`, `offers`, `action`, `confirm`, `noitems`, `done_basic`, `error_basic`) already exist in the platform language files. `numinf()` is a platform helper that picks the correct numeric plural form (see [Language files](hack-lang.md)).

## Checklist

- `mod/config.php` defines `hack_<name>_config( $core )` — no registration needed.
- The hack is installed and enabled (otherwise the generic param form is shown).
- The function always ends by stopping: `go()`, `json()`, or `footer()` + `stop()`.
- Action names are namespaced and avoid the reserved set (`upd`, `param`, `export`, `install`, `toggle`, `schema`).
- All `$core->post` / `$core->get` input is typed and sanitized before `conf()`.
- Every setting is a key of the single `conf` array — no settings table, no column per setting.
- Every key of `conf` is declared in `param.json`; runtime-generated state lives in a `$core->config()` block instead, or the `param` save destroys it.
- Forms and links target `$core->u( $core->app->pp, ... )`; saves redirect back with a `message`.
