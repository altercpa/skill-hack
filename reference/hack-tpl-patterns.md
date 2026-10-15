# Template Patterns — the AlterCPA UI cookbook

This is the practical companion to [Templates and Styling](hack-tpl.md). That page explains the
*engine* (blocks, directives, macros, caching). This one is a **catalog of the concrete layout
patterns** the platform reuses on every page — the ones AlterCPA Pro's and AlterCPA Lite's own
screens are built from. Styling and behaviour throughout are `core.css` + `core.js`.

> **Why this matters.** A hack page must be indistinguishable from a native AlterCPA page. That is
> achieved by *reusing the exact structures below*, class for class — not by inventing markup. Take
> the snippet closest to your screen as your starting point, then swap the data. The single most
> useful thing a hack author can do is copy
> [§18 — the standard hack CRUD screen](#18-the-standard-hack-crud-screen-copy-this-first).

**How to read this.** Recurring conventions get a full section with a snippet. Rarer one-offs are
gathered into compact "recipe" bullets. Where a bullet names a screen, it names the *kind* of screen
the pattern belongs to — the markup in the snippet is the whole of what you need to reproduce it.

---

## Contents

**Foundations** — [0. Mental model](#0-the-mental-model) · [1. Class macros (what's safe)](#1-class-macros--the-only-classes-you-should-write)
**Page anatomy** — [2. Toolbars](#2-toolbars-search-add-import-filters) · [3. Boxes](#3-boxes-panels) · [4. Widgets & dashboards](#4-widgets--dashboard-tiles)
**Tables** — [5. The list table](#5-the-list-table) · [6. Action cells & buttons](#6-action-cells-and-buttons) · [7. Grouped & configurable headers](#7-grouped-multi-level-and-configurable-headers) · [8. Key/value & special tables](#8-keyvalue-and-special-tables) · [9. Empty states & grouping](#9-empty-states-and-row-grouping)
**Forms** — [10. Multi-column forms](#10-multi-column-forms) · [11. Live-save & validation](#11-live-save-and-validation) · [12. Controls & pickers](#12-controls-and-pickers) · [13. Widgets & asset sets](#13-enhanced-widgets-and-asset-sets)
**Modals** — [14. Detail & menu modals](#14-detail-and-menu-modals) · [15. Modal forms & AJAX partials](#15-modal-forms-and-ajax-partials) · [16. Log/debug viewer modal](#16-the-logdebug-viewer-modal)
**Beyond the chrome** — [17. Standalone & export templates](#17-standalone-raw-and-export-templates) · [18. The standard hack CRUD screen](#18-the-standard-hack-crud-screen-copy-this-first)
**Reference** — [19. Status, glyphs & flags](#19-status-glyphs-and-flags) · [20. CSS helpers](#20-reference-altercpa-specific-css-classes) · [21. Naming & JS hooks](#21-naming-conventions-and-js-hooks) · [22. Checklist](#22-checklist-before-you-ship) · [23. Pattern index](#23-pattern-index)

---

## 0. The mental model

Almost every management page is assembled from four ingredients:

1. **A toolbar** on top — a collapsible search form and/or add form (§2).
2. **One or more boxes** (panels) wrapping the content (§3).
3. **A compact table** inside the box body, with a mirrored header/footer (§5–§9).
4. **Modals** popping from table rows or toolbar buttons (§14–§16), plus **pagination** at the bottom.

Editors (flow, box/case, split path) are the other archetype: a **multi-column form** (§10), often
served **inside a modal** and loaded over AJAX (§15), with fields that **save live** as you change
them (§11). Dashboards and hubs are built from **widget tiles** (§4).

Everything is expressed through `{tpl:…}` class macros (§1), never literal Bootstrap classes. The
only literal classes you write are the AlterCPA helpers (§20) and grid/utility classes that have no
macro (`row`, `col-*`, `checkbox`, `input-group`, `pull-left`, `text-right`, `hidden-xs`, …).

---

## 1. Class macros — the only classes you should write

Write `{tpl:macro}` instead of hard-coded CSS classes. This is the house convention used throughout
the platform: a macro resolves at **compile time** to the fixed `core.css` class name, and writing
`{tpl:btn-main}` instead of a literal `btn btn-primary` is part of what makes a hack page
indistinguishable from a native one (see [hack-tpl.md](hack-tpl.md#class-macros)). All the styling and
behaviour is `core.css` + `core.js`.

**Only use macros that exist.** A `{tpl:x}` whose name isn't in the built-in macro set resolves to an
**empty string** — so never invent macro names. Stick to the ones below and the ones you see in system
templates.

**You only need a handful.** By real usage frequency across the platform, the macros that matter are:

| Tier | Macros (by frequency) |
|------|-----------------------|
| Everywhere | `form-control` `btn-sm` `form-group` `no-padding` `btn-block` `btn-main` `input-sm` `box-body` `box-main` `table-compact` |
| Common | `box-header` `box-title` `table-border` `table-responsive` `btn-info` `btn-success` `btn-danger` `btn-default` `text-*` `alert-*` `form-inline` |
| Occasional | `box-tools` `box-footer` `box-widget` `table-main` `bg` `bg-active` `btn-xs` `btn-lg` `input-group`(literal) |

Everything else in the macro set is used in single digits — reach for it only when a system template does.

**Pair box/table macros with their parts.** `core.css` expects a box to contain its header/body/footer
and a table to sit inside the box-body trio — always pair `{tpl:box-*}` with its matching
`{tpl:box-body}` / `{tpl:box-header}` / `{tpl:box-footer}`, and put `{tpl:table-*}` inside
`{tpl:box-body} {tpl:no-padding} {tpl:table-responsive}` (§3) rather than hard-coding the structure.
The only literal classes you ever write are the AlterCPA helpers in §20 and grid/utility classes that
have no macro (`row`, `col-*`, `checkbox`, `input-group`, `pull-left`, `text-right`, `hidden-xs`, …).

---

## 2. Toolbars: search, add, import, filters

### 2.1 The collapsible search / add toolbar (the signature pattern)

Above a list sits a search form and, optionally, an add form. On desktop the fields are inline on one
line; on phones/tablets they collapse behind a full-width button.

```html
<form class="{tpl:form-inline} {tpl:form-group}" action="{u_search}" method="get">
<a class="{tpl:btn-default} {tpl:btn-xs} {tpl:btn-block} visible-xs visible-sm" role="button" data-toggle="collapse" href="#xfsearch">{search}</a>
<div id="xfsearch" class="{tpl:form-inline}-block collapse">
	<input type="text" class="{tpl:form-control} {tpl:input-sm}" name="s" value="{s}" placeholder="{search}" />
	<button type="submit" class="{tpl:btn-main} {tpl:btn-sm}">{find}</button>
</div>
</form>
```

Four moving parts: the `visible-xs visible-sm` toggle button; the `{tpl:form-inline}-block collapse`
container (the macro expands to give the literal `form-inline-block` class); `{tpl:input-sm}` on every
control; and CSS that forces `.form-inline-block.collapse` to `display:flex` on `≥992px` — so the
toolbar is **always open on desktop** and the toggle button is hidden. Conventions:

- Container ids: `xfsearch` (search), `xfadd` (add), `xfilter` / `xfsearch` (stat filters). Two
  toolbars on one page ⇒ **different ids**.
- Search = `method="get"` (query survives in URL + pagination); add = `method="post"`.
- Preserve context with hidden inputs inside the block (`wm` impersonated user, `g` gang, `sort`, …).
- Two toolbars sharing a line: give the left one `pull-left margin-hr`.

### 2.2 Composite multi-toolbar stack

Filter/admin pages stack **three** independent collapsible forms — search, add, and an
**import/export/clean** bar — each with its own id (`#xfsearch` / `#xfadd` / `#xfimport`). The
import bar is the standard "bulk data" affordance:

```html
<div id="xfimport" class="{tpl:form-inline}-block collapse">
	<input type="file" name="import" required="required" />
	<button type="submit">{import}</button>
	<a href="{u_export}">{export}</a>
	<a href="{u_clean}" onclick="return confirm('{confirm}');">{clean}</a>
</div>
```

### 2.3 Richer filter toolbars

The maximal filter bar adds: `select2` dropdowns,
`type="date"` from/to, narrow selects (`short`/`shorter`/`tiny`, §20), an Excel export link, a settings
cog, and an **extended-filter modal** trigger. Idioms:

- **Quick-range pills** carry `active` when current: `<a class="{tpl:btn-info} {tpl:btn-sm}<!-- IFSET is_week --> active<!-- ENDIF -->" href="{u_week}">{day7}</a>`.
- **Boolean toggle as a pill** — the same `active`/collapsing-label idiom toggling a yes/no filter
  (target-clicks-only), not a date preset.
- **Excel export**: `<a class="{tpl:btn-default} {tpl:btn-sm}" title="Microsoft Excel XML" href="{u_csv}"><i class="fa fa-file-text"></i> Excel</a>`.
- **Extended-filter modal** (`#extrasearch`): overflow filters live in a modal whose `modal-footer`
  submit button submits the **same** outer `<form>` (§14). Each field is gated either by an
  admin-config flag (`filteruser`, `filteroffer`, …) or by data presence.
- **Empty-value companion checkbox** — a free-text filter paired with a "search for blank" checkbox:
  ```html
  <label class="inline-checkbox empty-value"><input type="checkbox" name="emptyus" /> {empty}</label>
  <input class="{tpl:form-control}" type="text" name="us" value="{param_us}" />
  ```
  Recurs across every stats/analytics extended-filter modal (Pro and Lite).

### 2.4 Faceted filter chips

Currently-applied filters render as removable `btn-warning`/`btn-danger` chips with a trailing
`fa-times`, each linking to a URL that clears just that facet, plus a hidden input that re-persists it
:

```html
<!-- IFSET ip -->
<input type="hidden" name="ip" value="{ip}" />
<a href="{u_ip}" class="{tpl:btn-warning} {tpl:btn-sm}">{ip} <i class="fa fa-times"></i></a>
<!-- ENDIF -->
```

### 2.5 Bulk-action toolbar (check-all)

A master `.checkall` checkbox toggles per-row `.checks` checkboxes; a `form-inline-block` **below** the
table holds several `<button name="act" value="…">` sharing one form. Often paired with live
subtotal aggregation (§11).

---

## 3. Boxes (panels)

Never place a bare table on the page — wrap it. Box macro families: `box-main` / `box-primary` /
`box-success` / `box-danger` / `box-warning` / `box-info` / `box-widget`, with parts `box-header` /
`box-body` / `box-footer` / `box-title` / `box-tools`.

**Table box** — the standard body class trio drops padding and lets the table scroll:

```html
<div class="{tpl:box-main}"><div class="{tpl:box-body} {tpl:no-padding} {tpl:table-responsive}">
	<table class="{tpl:table-compact} {tpl:table-border}"> … </table>
</div></div>
```

- **Several tables ⇒ several titled boxes** (one per kind of item on the page): each is its
  own `box-main` with a `box-header`/`box-title`.
- **Box tools** — right-aligned header actions, usually `hidden-xs`, with the same buttons re-rendered
  as a `btn-group-justified visible-xs` block in the body for phones.
- **Colour the box by state** — `<div class="<!-- IFNOT offer.my -->{tpl:box-success}<!-- ELSE -->{tpl:box-primary}<!-- ENDIF -->">`.
- **Settings hub = icon-tile grid**, not tabs: a `row` of `col-sm-6 col-md-4 col-lg-3`
  `{tpl:box-widget} widget-user-2` link-cards, each pointing to a self-contained sub-page.
  There is **no tab/sub-nav component** in the platform — hubs are grids of links.
- **Whole-section empty state**: wrap an entire module box in `<!-- IFSET hasX --> …box… <!-- ELSE --><div class="{tpl:alert-info}">{noitems}</div><!-- ENDIF -->`.

---

## 4. Widgets & dashboard tiles

Dashboards, hubs, money pages and call-queues are built from **tiles**, not tables. The KPI tile is
`{tpl:box-widget} widget-user-2` wrapping a `config-block`:

```html
<div class="{tpl:box-widget} widget-user-2"><a href="{widget.url}">
	<div class="config-block {tpl:bg}">
		<span class="config-block-icon {tpl:bg-active}"><i class="fa fa-{widget.icon}"></i></span>
		<span class="config-block-name">{widget.value}</span>
		<span class="config-block-line"><i class="fa fa-caret-up"></i> {widget.trend}</span>
	</div>
	<div class="config-block-descr {tpl:bg-active}">{widget.name}</div>
</a></div>
```

Grid: `col-sm-6 col-md-4 col-lg-3` (or `col-md-6 col-lg-4`) inside a `BEGIN widget` loop. Variants:

- **Trend caret** — `fa-caret-up` (success) / `fa-caret-down` (danger) / `fa-sort` (muted) chosen by
  `IFSET/ELSEIF/ELSE` on up/down/same.
- **Balance tile** — big current-balance figure + an "on hold" sub-line.
- **These tiles appear outside dashboards** — queue counters, settings hub cards.
- **User/manager profile card** — `widget-user-2` + `widget-user-header {tpl:bg}` with a circular
  avatar (or a `.support-user {tpl:bg-active} disabled` fallback icon), `widget-user-username` /
  `widget-user-desc`, a `nav nav-stacked` contact list.
- **Announcement slots** — the dashboard exposes eight CMS-configurable page zones
  (header/maintop/center/bottom/sidetop/…), each rendering either a dismissible alert
  (`<div class="{x.ac}"><a class="close" href="{x.hide}">×</a>{x.text}`) or a full titled box.
- **News-as-nav-pills** — a compact list widget: `<ul class="nav nav-pills nav-stacked">` rows with a
  pulled-right date and a coloured category badge.
- **Adaptive column width** — `col-md-<!-- IFSET hasrec -->6<!-- ELSE -->12<!-- ENDIF -->` lets one of
  two optional columns take the full row when its sibling is absent.

Charts that accompany tiles are covered in §17.4.

---

## 5. The list table

The workhorse. The minimal form is a search toolbar, a table, and a pagination nav.

```html
<div class="{tpl:box-main}"><div class="{tpl:box-body} {tpl:table-responsive} {tpl:no-padding}">
<table class="{tpl:table-compact} {tpl:table-border}">
<thead><tr>
	<th width="1%">#</th><th width="84%">{name}</th><th width="20%">{info}</th><th width="5%">{action}</th>
</tr></thead>
<tfoot><tr>
	<th width="1%">#</th><th width="84%">{name}</th><th width="20%">{info}</th><th width="5%">{action}</th>
</tr></tfoot>
<tbody>
<!-- BEGIN item -->
	<tr>
		<td align="right">{item.id}</td>
		<td><a href="{item.url}">{item.name}</a> {item.more}</td>
		<td align="center" nowrap="nowrap">{item.info}</td>
		<td align="center" nowrap="nowrap" class="{tpl:no-padding}"><a href="{item.edit}" class="{tpl:btn-info} {tpl:btn-sm}">{edit}</a><a href="{item.del}" class="{tpl:btn-danger} {tpl:btn-sm}" onclick="return confirm('{confirm}');">{del}</a></td>
	</tr>
<!-- END item -->
<!-- BEGIN noitem -->
	<tr><td class="noitems" colspan="4">{noitems}</td></tr>
<!-- END noitem -->
</tbody>
</table>
</div></div>
<nav class="text-center">{pages}</nav>
```

Rules that make it look native:

- **Mirror the header in the footer.** `<thead>` and `<tfoot>` carry the *same* row (copy-paste, edit
  both). Every list/stats table does this.
- **Widths on `<th>`** as `width="N%"`; `width="1%"` shrinks a column to its content (id / icon / single button).
- **Alignment by attribute**: `align="right"` for ids/numbers, `align="center"` for status/actions,
  `nowrap="nowrap"` on numbers/dates/button strips. House style — do not swap for utility classes.
- **Empty state is explicit**: a `noitem` block with `<td class="noitems" colspan="N">`, added by the
  controller when the list is empty. Get `colspan` right — compute it in PHP (or with an inline
  conditional, `colspan="<!-- IFSET admin -->9<!-- ELSE -->8<!-- ENDIF -->"`) when
  columns are conditional.
- **`{tpl:table-main}` vs `{tpl:table-compact}`** — `table-compact`+`table-border` is the dense admin
  table; `{tpl:table-main} {tpl:no-margin}` is its **front-end sibling** for public/catalog tables,
  where secondary columns get `hidden-xs` for progressive disclosure.
- **Sortable** (client side): add `tablesorter` + an `id`, and connect the `tablesorter` asset set.
  Sorting is bound to `.sub` header cells; there are **no** server-side sort links in the grids. To
  sort by a hidden key while displaying a label, prefix a `<span class="hidden">{sortkey}</span>`
.
- **Pagination nav** has three interchangeable spellings: `<nav class="lister">{pages}</nav>`
  (list/grid), `<nav class="text-center">{pages}</nav>`, and `<nav class="pages">{pages}</nav>` (common
  in hacks). Pick one; they differ only in margin.

---

## 6. Action cells and buttons

Buttons **inside tables** are always `{tpl:btn-sm}`, packed into `class="{tpl:no-padding} compact-td"`
(zero font-size cell that butts inline buttons together):

```html
<td class="{tpl:no-padding} compact-td">
	<a class="{tpl:btn-main} {tpl:btn-sm}" href="{item.url}" target="_blank" title="{open}"><i class="fa fa-fw fa-external-link"></i></a>
	<a class="{tpl:btn-info} {tpl:btn-sm}" href="{item.edit}" title="{edit}"><i class="fa fa-fw fa-cogs"></i></a>
	<a class="{tpl:btn-danger} {tpl:btn-sm}" href="{item.del}" onclick="return confirm('{confirm}')" title="{del}"><i class="fa fa-fw fa-trash"></i></a>
</td>
```

- Icon-only buttons: Font Awesome `fa-fw` + a `title`. **Colour = intent**: `btn-main`/`btn-primary`
  neutral, `btn-info` edit/settings, `btn-success` add/enable, `btn-warning` caution, `btn-danger`
  delete/disable, `btn-default` secondary.
- **Confirm destructive actions** — inline `onclick="return confirm('{confirm}')"`, or the stronger
  typed-word `safedel()` (a `prompt()` that requires typing `delete`), defined at the top of
  the page, next to the table it guards:
  ```html
  <script>function safedel(url){ if(prompt('{confirm}')=='delete'){ location.href=url; return true; } return false; }</script>
  <a href="#" class="{tpl:btn-danger} {tpl:btn-sm}" onclick="safedel('{item.del}');">{del}</a>
  ```
- **Disabled action** — drop the `href` and add a bare `disabled` attribute on the `<a>` when the
  action isn't available: `class="{tpl:btn-info} {tpl:btn-sm}<!-- IFNOT item.x --> disabled<!-- ENDIF -->"`
  Works for a whole `compact-td` cluster.
- **Action-overflow modal** — when a row has many actions, keep the primary few inline and put a
  trailing `<i class="fa fa-ellipsis-h">` button that opens a per-row modal of secondary actions split
  by `<hr/>` into tiers.
- **Inline copy-link input** — a read-only `.flowlink` input holding a URL/code + a copy button wired
  to `copycode()`; the input auto-selects on focus:
  ```html
  <td class="{tpl:no-padding}"><input type="text" id="flowlink{id}" class="{tpl:form-control} {tpl:input-sm} flowlink" value="{url}" /></td>
  …
  <a class="{tpl:btn-main} {tpl:btn-sm}" href="#" onclick="return copycode('flowlink{id}');"><i class="fa fa-fw fa-copy"></i></a>
  <script>
  $(".flowlink").focus(function(){ this.select(); this.setSelectionRange(0,9999); }).mouseup(e=>e.preventDefault());
  function copycode(uid){ var t=document.getElementById(uid); t.select(); t.setSelectionRange(0,99999); document.execCommand("copy"); return false; }
  </script>
  ```
  A modern no-library variant reads a `data-*` attribute and calls `navigator.clipboard.writeText`
.
- **Multi-action submit** — one form, several submit buttons distinguished by `name="action"
  value="accept|delete|export"` (or a `name="download" value="1"` second submit for a save-and-download
  branch).
- **Add-row form inside `<tbody>`** — instead of a toolbar-level add form, a literal `<form>` wraps a
  single trailing `<tr>` (a file/name input + submit) appended after the `item`/`noitem` rows, so the
  add control lives *in* the table and lines up with its columns — an upload row, a
  create-by-name row, a file manager:
  ```html
  <form action="{u_upload}" method="post" enctype="multipart/form-data"><tr>
  	<td colspan="3"><input type="file" name="upload[]" multiple /></td>
  	<td class="{tpl:no-padding} compact-td"><button type="submit" class="{tpl:btn-main} {tpl:btn-sm}">{upload}</button></td>
  </tr></form>
  ```

---

## 7. Grouped, multi-level, and configurable headers

### 7.1 Two-level grouped header

Columns that fall into groups get a top row of `colspan` group cells and a second row of `<td class="sub">`
column labels:

```html
<thead>
<tr>
	<th colspan="4">{descr}</th><th colspan="3">{process}</th><th colspan="2">{cancel}</th>
</tr>
<tr>
	<td class="sub" width="1%"><a href="{srt0}" class="black">#</a></td>
	<td class="sub"><a href="{srt1}" class="black">{who}</a></td>
	…
</tr>
</thead>
```

- `.sub` = compact centered 11px header cell. Sort links wrap in `class="black"` to keep the label the
  normal foreground colour instead of link-blue.
- **Dynamic group widths** — `<th colspan="{wfinance}">{gfinance}</th>` where the colspan/gate is a
  template variable, so a group vanishes entirely when its columns are all hidden. Analytics grids have
  nine such independent optional groups.
- **The full head/foot sandwich** — the main statistics grid's `<tfoot>` emits three rows: totals data, then a
  mirrored `.sub` legend, then a mirrored group-header row — a true mirror image of the head. Simpler
  grids emit only the bare totals row.

### 7.2 Column-visibility (the `z*` / `x*` flag system)

Wide stats/analytics grids let the user pick columns. A settings modal drives it:

```html
<!-- #statcols modal: BEGIN statcol loop of checkboxes, POSTs to {u_cols} -->
<!-- every column, in thead/tbody/tfoot, is wrapped identically: -->
<!-- IFSET zclicks --><td align="center">{stat.clicks}</td><!-- ENDIF -->
<!-- group colspans use the x* / w* twin family: --> <th colspan="{xspace}">{spaces}</th>
```

`z*` gates a single column, `x*`/`w*` a group colspan. The **same flag name gates the cell in thead,
tbody, and both tfoot rows** — keep them in lockstep. Per-goal metric blocks repeat via a nested
`<!-- BEGIN goal -->…<!-- END goal -->` loop that re-emits the whole metric-flag set once per goal
(export mirrors of the same grid use `c*` flags instead, §17.3).

### 7.3 Feature-matrix and per-currency columns

- **Boolean matrix** — N narrow icon-only columns (one per phase/capability), each cell a coloured
  `fa-check` or empty, giving an at-a-glance grid.
- **Data-driven currency columns** — header *and* body both loop `<!-- BEGIN cur -->` to build parallel
  `<th>`/`<td>` per active currency, with a dynamic `colspan="{colspan}"` where the group spans
  them.
- **Data-driven schema** — a price table builds one `<th>` per geo via `BEGIN geo`,
  or falls back to fixed columns via `BEGIN nogeo`: the column set itself is per-record.

### 7.4 Reorderable grouping dimensions

A `jQuery.sortable()` list of `<li data-id>` pills (`#utmlist`) whose drag order is serialized into a
hidden input and resubmitted, letting the user reorder which dimensions group the rows. The same
widget reorders priority rules.

---

## 8. Key/value and special tables

- **System-info table** — `{tpl:table-compact} {tpl:table-border}` used as a label/value grid: a fixed
  `width="10%"` `<strong>` label cell + a free-text value cell, no thead/tfoot/actions.
- **Fact-list table** — a bare `<table class="table">` of `<td class="text-info"><i class="fa fa-fw fa-barcode"></i> <b>ID</b>: {id}</td>` rows for an entity's meta sidebar.
- **Modal key/value table** — `{tpl:table-compact} {tpl:no-margin}` (no box) with `<td class="fat text-right">`
  labels; add a `BEGIN head` sub-loop for arbitrary HTTP-header name/value rows, and a `toolong` class
  on values that may overflow.
- **Definition-list in a cell** — a `<dl class="dl-horizontal no-margin">` of field→old/new pairs
  inside one table cell, to express a multi-field diff/changelog.

---

## 9. Empty states and row grouping

- **Per-row empty state**: the `noitem` block (§5). **Whole-module empty state**: an `{tpl:alert-info}`
  in place of the whole box (§3).
- **In-body group separators**: a full-width `<td class="head" colspan="N">` row between groups
  (`.head` = bold). Variants: `<tr class="active fat">`, or a KPI body cell that becomes a head via
  `<td <!-- IFSET row.head --> class="head"-->…>`.
- **Master-detail grouping**: `<!-- BEGIN user --> …group-header <tr class="info">… <!-- BEGIN item -->…<!-- END item --> <!-- END user -->`, with running subtotals reset per group.
- **Two-row-per-record packing**: two adjacent `<tr>`s sharing `data-oid`/`ohover{id}`, JS co-highlights
  them on hover to fit more fields per record.

---

## 10. Multi-column forms

Non-trivial editors are hand-laid multi-column forms on the Bootstrap grid — `row` → `col-*`
`{tpl:form-group}` cells:

```html
<div class="row">
	<div class="{tpl:form-group} col-sm-6">
		<label for="thename">{name}</label>
		<input type="text" name="name" value="{name}" class="{tpl:form-control} editables" required="required" />
	</div>
	<div class="{tpl:form-group} col-sm-6">
		<label for="thecomp">{default}</label>
		<select class="{tpl:form-control} editables" id="thecomp" name="comp"> … </select>
	</div>
</div>
```

- Responsive mixes per field (`col-md-4 col-sm-6`, `col-sm-6 col-md-4 col-lg-3`); a field owning its
  line uses `col-sm-12`. Column class can even be computed per field from data.
- **Field descriptions** in `<p class="help-block">`.
- **Section dividers** — `<div class="hh"><span>{title}</span></div>` (a hairline with a centered
  label). A plain `<hr/>` is the lighter break inside modals.
- **Input + button**: Bootstrap `input-group` + `input-group-btn`.
- **The `form()` generator** — a flat, horizontal label-left/field-right form built from a typed field
  list (text/email/number/pass/textarea/select/mselect/radio/checkbox/date/file/color/code/mces/mcea/button
  + head/line separators). Use it for simple settings forms; hand-lay a grid form when you need columns,
  tables-in-forms, live save, or a bespoke layout. The hack Settings page uses the same vocabulary
  ([hack-config.md](hack-config.md)).

---

## 11. Live-save and validation

Editors save **field-by-field over AJAX**. Mark live controls `editables`; a `change` handler
serializes the form, POSTs it, and paints the field wrapper green/red:

```html
<input type="text" name="name" value="{name}" class="{tpl:form-control} editables" required="required" />
<script>
$("#theform").submit( () => false );              // block the real submit; saving is per-field
$(".editables").change(function(){
	$.ajax({ url:"{u_save}", type:"POST", dataType:"JSON", data:$("#theform").serialize(),
		success:d=>{ $(this).parent().toggleClass("has-success",!!d.status).toggleClass("has-error",!d.status); },
		error:()=>{ $(this).parent().removeClass("has-success").addClass("has-error"); } });
});
</script>
```

Variants:

- **Single-field autosave**: `autocaps` + `data-url` + a `…holder` wrapper id.
- **Group rollup**: a live-edited cell recomputes client-side subtotals into `tfoot` cells on success
  (a small recompute function per group; another that sums the checked rows).
- **Computed value** (not save-on-blur): a per-row `<form>` whose wallet `<option data-cms data-purse>`
  drives a live "amount to pay" recalculation on every keystroke.
- **Save-on-Enter**: `<input onkeypress="return <handler>(this)">` intercepts keyCode 13, confirms,
  disables itself, fires a bare AJAX POST.
- **Server-rendered validation** — per-field error classes `{er1}`, `{er2}`, … injected into
  `class="{tpl:form-group} has-feedback {erN}"` (auth forms), or an
  **auth "icon-in-input"** with a trailing `<span class="{tpl:form-control}-feedback"><i class="fa fa-lock"></i></span>`.
- **Required-fields-as-data guard** — on submit, JS iterates a template-injected JSON id list
  (`{checkapprove}`) and blocks + highlights (`has-error text-danger`) + `alert()`s if any is empty —
  validation rules delivered as data, not HTML5 `required`.

---

## 12. Controls and pickers

- **Button-group segmented control** — `<div class="btn-group btn-group-justified" data-toggle="buttons">`
  of `<label class="{tpl:btn-main}<!-- IFSET x.s --> active<!-- ENDIF -->"><input type="radio|checkbox" …></label>`
  (source picker, weekday picker).
- **Weekday preset shortcuts** — `wkds0/1/d/e()` helpers flip both the `.active` classes and the hidden
  checkboxes for none/all/weekdays/weekends.
- **Radio picker with live limits** — payment methods as a `<ul>` of labelled radios (not a `<select>`)
  each annotated with max/min/commission; selecting one syncs the amount input's `min`/`max` from
  `data-*` and gates the submit.
- **Source-mode radios gating field groups** — mutually-exclusive radios (empty/dummy/upload/zip/url)
  where each `change` disables all sibling class-scoped groups (`.forsiteready`, `.forsitearchive`, …)
  and enables only its own.
- **Mutually-exclusive action radios + proxy** — action blocks share `name="act"`; any sub-control
  carrying `class="changeaction" data-action="X"` auto-checks radio `#Xaction` on change, so far-apart
  fields drive one radio group. Independent `BEGIN mark` flag checkboxes sit alongside.
- **optgroup hints** — a country `<select>` splits into `<optgroup label="{goodcountry}">`/`{badcountry}`
  when geo-targeting is set.
- **Host/slug split input** — a 3-cell `<table class="table-main no-margin">` visually joins a host
  input, a literal `/`, and a slug input `pattern="[a-z0-9\/\-\_]+"`.
- **Static pre-numbered rows** — a low-tech key/value editor appends ten blank rows with reserved ids
  `param[1001]`…`param[1010]`, so new params need no add-row JS.

---

## 13. Enhanced widgets and asset sets

Connect an asset set with `$core->site->set('…')` **before** `header()`, then use the trigger class.

| Set | Trigger | Gives you |
|-----|---------|-----------|
| `select2` | `class="… select2"` / `select2w` (100% width) | Searchable / multi-select dropdowns; also defines `reset2s(name)`. |
| `tablesorter` | `class="… tablesorter"` + table `id` | Client-side column sorting. |
| `summernote` | `class="… simplemce"` / `advancemce` | WYSIWYG editors. |
| — (core) | `class="… usersearch"` + `data-linkto` | AJAX user autocomplete (`initusersearch`). |
| — (core) | `class="… codemirror" lang="…"` | Code editor. |
| — (core) | `class="… colorpick"` | Colour picker. |

- **select2 in a modal** must pass `dropdownParent: $('#modalId')` (+ `width:"100%"`) or the dropdown
  renders outside the modal. `usersearch` reads `data-linkto="modalId"` to do this automatically.
- **`reset2s` multiselect ⇄ raw text** — a `<i class="fa fa-code">` toggle swaps a multi-select for a
  comma-list textarea. Needs the companion structure: `…_group` wrapper, a hidden `…_dummy` text input
  carrying `data-name` + the raw value (the `mselect raw` field type does this).
- **Drag-reorder** — `jQuery.sortable({handle:'i.dragme'})` over `<li data-id>` items, POSTing the new
  comma-joined id order on drop.
- **CodeMirror editor** — connect the `codemirror` asset set and add `class="codemirror" lang="…"` to a
  `<textarea>` (the `code` field type is the inline form of it). The full-page editor variant
  wraps the textarea in a `fullcode` box with a Save button in *both* box-body and
  box-footer, plus a global Ctrl/Cmd+S keydown that POSTs to `formAction+"&z=ajax"` for a no-redirect save.
- **Third-party embeds** seen in the wild: reCAPTCHA v2, Commento comments, `<audio>` notification
  elements, embedded payment-gateway forms — grafted in with only `{tpl:btn-*}` styling.

---

## 14. Detail and menu modals

Data-heavy tables push detail into a modal opened from a compact button. **Per-row modals re-iterate
the block after the table** — a block can be re-opened any number of times, so you iterate `item` once
for rows and again for modals with no extra controller code:

```html
<!-- row action cell --> <a class="{tpl:btn-primary} {tpl:btn-sm}" href="#flow{flow.id}" data-toggle="modal" data-target="#flow{flow.id}"><i class="fa fa-fw fa-bar-chart"></i></a>
…
<!-- after the table --> <!-- BEGIN flow -->
<div class="modal fade" id="flow{flow.id}" style="display: none;"><div class="modal-dialog modal-sm"><div class="modal-content">
	<div class="modal-header">
		<button type="button" class="close" data-dismiss="modal" aria-label="Close"><span aria-hidden="true">×</span></button>
		<h4 class="modal-title">{stats}</h4>
	</div>
	<div class="modal-body"> … </div>
</div></div></div>
<!-- END flow -->
```

Reproduce that skeleton exactly. Dialog size: `modal-sm` (menu/short), default (medium), `modal-lg`
(full editor). Shapes:

- **Menu modal** — a stack of `btn-block` links (the "more actions" popup).
- **Copy modal** — a `<form>` around a single name input + a `modal-footer` submit (duplicate entity).
- **Multiple modals per row** — a row can spawn several differently-purposed modals: a stats modal
  *and* an "enter as" impersonation modal, say.
- **Settings-modal-as-own-form** — the column/grouping config modals (`#statcols`, `#statgroup`) are
  each their own `<form>` posting to their own URL, sitting outside the filter form.
  A **goals/events grid-in-modal** editor renders rows of id + active checkbox + code + name saved as
  one form.

---

## 15. Modal forms and AJAX partials

Row editors open as a form inside a modal. Two mechanisms:

**A) Inline modal form** — the whole form is on the page and just shown (the `#create` modal of an
add button, say).

**B) AJAX-loaded partial (the standard for edit).** The page ships an empty modal shell with a spinner;
JS GETs the form HTML and injects it. The partial contains **only** the form content — from
`modal-header` to `modal-footer`, with **no** outer `modal/modal-dialog/modal-content` wrappers, since
it's dropped straight into `.modal-content`.

The refined, widely-copied version snapshots the pristine spinner shell and restores it before every
load so stale state can't leak between opens (this is the shape §18 uses):

```html
<div class="modal fade" id="theboxform" style="display:none;"><div class="modal-dialog modal-lg"><div class="modal-content" id="theformcontent">
<div class="modal-body"><div class="progress active margin"><div class="progress-bar progress-bar-primary progress-bar-striped" style="width:100%"></div></div></div>
</div></div></div>
<script>
const dummyformcontent = $("#theformcontent").html();          // snapshot the spinner
function editform( uid = 0, clone = false ) {
	let action = clone ? $("#editbtn"+uid).data("clone") : $("#editbtn"+uid).data("url");
	$('#theboxform').modal('show');
	$("#theformcontent").html( dummyformcontent );             // reset before each load
	$.ajax({ url: action, type:"GET", dataType:"text",
		success: data => { $("#theformcontent").html(data); /* re-init select2 w/ dropdownParent, etc. */ },
		error: () => $('#theboxform').modal('hide') });
	return false;
}
</script>
```

Related idioms:

- **Repopulate a shared modal from `data-*`** (no server round-trip) — one `#whitesite` modal serves
  both add and edit by rewriting the form `action` and refilling from the trigger's
  `data-url/name/user/offer`.
- **Shared-modal-by-class** — many unrelated links all carry `class="makenicetable"` and open one
  `#nicetable` modal, `.load()`ing their own href into it.
- **Read-only info modal** — same lazy-load, but a read-only fragment (a bare `<table>`, no box)
  instead of a form — a list screen opening a per-row detail fragment.
- **Lazy sub-section** — an empty `<div data-url="…">` inside a bigger modal that `.load()`s a partial
  only when opened.
- **Reload-own-modal** — a select change re-fetches the *same* modal body to render dependent fields
  (a network picker whose fields depend on the network chosen).

---

## 16. The log/debug viewer modal

A recurring, near-identical pattern across every log screen (advertiser/delivery/API/postback/company
call logs): the list row shows only status + a `{show}` button; the modal dumps the raw HTTP exchange
as a `<dl>` of independently-gated `<dt>`/`<dd><pre>` pairs (URL, GET, headers, POST, POST-decoded,
reply, CURL config), often with a one-click "resend" (confirm + GET):

```html
<div class="modal-body"><dl>
	<dt>URL</dt><dd><pre>{item.url}</pre></dd>
	<!-- IFSET item.post --><dt>POST</dt><dd><pre>{item.post}</pre></dd><!-- ENDIF -->
	<dt>{reply}</dt><dd><pre>{item.reply}</pre></dd>
</dl></div>
```

Every log screen in the platform — advertising, delivery, API, postback — uses this one shape. A
metadata companion (`fat text-right` key/value table with IP+flag/browser/OS/UA) is often stacked in
the same modal, or offered as a second modal. HTTP status cells colour by result:
`<span class="<!-- IFSET item.codeok -->text-success fat<!-- ELSE -->text-warning-->">{item.code}</span>`.

---

## 17. Standalone, raw, and export templates

Not every template is an admin page inside the header/footer chrome.

### 17.1 Standalone / raw output pages

Some templates are full, self-contained HTML documents (or bare JS/pixel fragments) served directly to
a browser, **with no `{tpl:}` macros at all**:

- **Vendor-pixel / redirect pages** — full `<!DOCTYPE html>` docs firing VK/FB/GA/Metrika/… snippets
  per `IFSET`, with `BEGIN fb`-style loops for multiple ids and a raw `{pixs}` passthrough for
  arbitrary extra code.
- **Iframe relay** — a full-bleed embed document, plus a pure-JS relay fragment beside it.
- **Error / lock pages** — a ban page is a standalone doc with its own inline `<style>`, so it renders
  even when the configuration is broken; a 404 is a one-line fragment dropped inside the normal
  layout. A hack intercepting authentication may ship a full page too, linking
  `/core/css/style.css` directly the way the login screen does.
- **Setup screens** — the installer screens use **raw Bootstrap classes** and hard-code
  `<link href="/core/css/core.css">`, because they render outside the normal app shell (before the
  page assets are wired up).

**Takeaway for hacks:** if your template renders outside the authenticated shell (a pixel, a redirect,
a pre-login gate), don't reach for `{tpl:box-*}` — write a plain self-contained document like these.

### 17.2 Public / guest pages

The marketing landing uses a **fully custom design system** — `hero`, `offer-grid`, `card`, `lb` … —
and **zero** `{tpl:}` macros. Guest-facing pages don't use the admin box/table look; don't apply the
admin vocabulary to them.

### 17.3 Export templates — a whole category

Every on-screen list/stats page has a same-named export sibling. **These are Excel-2003 SpreadsheetML
fragments** (`<Row>/<Cell>/<Data>`), *not* CSV, despite the `xls-`/"Excel" naming. The `<?xml>` /
`<Workbook>` / `<Worksheet>` / `<Styles>` wrapper is emitted by PHP; the template supplies only the
`<Row>` stream. **The governing rule: an export template is a structural transliteration of its screen
template** — same `BEGIN` loops, same `IFSET` flag names, same variables; `<tr>`→`<Row>`,
`<td>`→`<Cell><Data>`. Links and icons collapse to plain typed cells (no `<a>`, no `<i>`).

```xml
<Row>
	<Cell ss:MergeAcross="{xfinance}" ss:StyleID="Bold"><Data ss:Type="String">{gfinance}</Data></Cell>
</Row>
<Row>
	<Cell ss:StyleID="Bold"><Data ss:Type="String">{colname}</Data></Cell>
	<!-- IFSET zcol --><Cell ss:StyleID="Bold"><Data ss:Type="String">{colname2}</Data></Cell><!-- ENDIF -->
</Row>
<!-- BEGIN row -->
<Row>
	<Cell><Data ss:Type="String">{row.name}</Data></Cell>
	<Cell><Data ss:Type="Number">{row.total}</Data></Cell>
	<!-- IFSET zcol --><Cell><Data ss:Type="Number">{row.col}</Data></Cell><!-- ENDIF -->
</Row>
<!-- END row -->
```

Rules a hack author must know:

- **Cell typing** — `ss:Type="String"` or `"Number"`. Numbers are raw (Excel formats them); anything
  mixed/currency/formerly-a-link is `String`.
- **Group headers merge with `ss:MergeAcross="{x…}"` = colspan − 1.** The export needs its **own**
  pre-decremented `x*` variable family — you cannot reuse the screen's `w*`/colspan var. (Column-gating
  flags are often renamed `c*` in exports, e.g. `cutm` gates 13 columns.)
- **Only one style**, `ss:StyleID="Bold"`, on header/total rows. No colours/borders/formats in-template
  (they live in the upstream `<Styles>`).
- **Same auto-escaping as HTML** — don't double-escape, don't pass raw HTML into an export var.
- **Parallel `{field_csv}` vars** — exports often consume a locale-flat `{ord.price_csv}` family instead
  of the screen's display `{ord.price}`.
- **Footgun** — when a merge span is a nested-`IFSET` literal (not a computed var), keep it hand-synced
  to the number of optional `<Cell>`s before it.

Lite ships the same idea under an `xml-` prefix instead of `xls-`.

### 17.4 Charts

A chart is a `<canvas>` plus a data literal built by re-walking a template loop:

```html
<canvas id="graphic" style="width:100%; height:330px;"></canvas>
<script src="/core/js/chart-bundle.js"></script><script src="/core/js/graph.js"></script>
<script>
var graphtype = dategraph ? 'line' : 'bar';
var graphdata = [ <!-- BEGIN gr -->{ x: new Date('{gr.date}'), y: {gr.total} },<!-- END gr --> ];
</script>
```

Series toggles use `graphcolbox` checkboxes (one per visible column, in a `.graphcol` cell) whose state
persists via `{u_graph}`. Load `chart-bundle.js`/`graph.js` **only when a graph is active**, as the
statistics screens do, to avoid the payload on every view.

---

## 18. The standard hack CRUD screen (copy this first)

Hacks converge on the **same** list-with-modal-editor screen, copied near-verbatim from one another.
This is the canonical shape for a hack's main page — start here and swap the data. It composes: the collapsible search toolbar (§2.1), the compact table with mirrored footer
(§5), compact-td actions (§6), pagination, and the snapshot-shell AJAX modal editor (§15B).

```html
<div class="entry">{text}</div>

<form class="{tpl:form-inline} {tpl:form-group}" action="{u_search}" method="get">
<a class="{tpl:btn-default} {tpl:btn-xs} {tpl:btn-block} visible-xs visible-sm" role="button" data-toggle="collapse" href="#xfsearch">{search}</a>
<div id="xfsearch" class="{tpl:form-inline}-block collapse">
	<input type="text" class="{tpl:form-control} {tpl:input-sm}" name="s" value="{s}" placeholder="{search}" />
	<button type="submit" class="{tpl:btn-main} {tpl:btn-sm}">{find}</button>
	<a href="#" class="{tpl:btn-success} {tpl:btn-sm}" id="editbtn-1" data-url="{u_add}" onclick="return editform(-1);">{add}</a>
</div>
</form>

<div class="{tpl:box-main}"><div class="{tpl:box-body} {tpl:no-padding} {tpl:table-responsive}">
<table id="itemlist" class="{tpl:table-compact} {tpl:table-border}">
	<thead><tr><th>{name}</th> … <th width="4%">{action}</th></tr></thead>
	<tfoot><tr><th>{name}</th> … <th width="4%">{action}</th></tr></tfoot>
	<tbody>
	<!-- BEGIN item -->
		<tr class="<!-- IFNOT item.active -->text-muted <!-- ENDIF -->{item.line}">
			<td><a href="{item.url}">{item.name}</a></td>
			…
			<td class="{tpl:no-padding} compact-td">
				<a href="#" class="{tpl:btn-primary} {tpl:btn-sm}" onclick="return editform('{item.id}', true);" title="{clone}"><i class="fa fa-fw fa-clone"></i></a>
				<a href="#" class="{tpl:btn-info} {tpl:btn-sm}" id="editbtn{item.id}" data-url="{item.edit}" data-clone="{item.clone}" onclick="return editform('{item.id}');" title="{edit}"><i class="fa fa-fw fa-edit"></i></a>
				<a href="{item.del}" class="{tpl:btn-danger} {tpl:btn-sm}" onclick="return confirm('{confirm}');" title="{del}"><i class="fa fa-fw fa-trash"></i></a>
			</td>
		</tr>
	<!-- END item -->
	<!-- BEGIN noitem --><tr><td class="noitems" colspan="9">{noitems}</td></tr><!-- END noitem -->
	</tbody>
</table>
</div></div>
<nav class="pages">{pages}</nav>

<div class="modal fade" id="theboxform" style="display:none;"><div class="modal-dialog modal-lg"><div class="modal-content" id="theformcontent">
<div class="modal-body"><div class="progress active margin"><div class="progress-bar progress-bar-primary progress-bar-striped" style="width:100%"></div></div></div>
</div></div></div>

<script type="text/javascript">
const dummyformcontent = $("#theformcontent").html();
function editform( uid = 0, clone = false ) {
	let action = clone ? $("#editbtn"+uid).data("clone") : $("#editbtn"+uid).data("url");
	$('#theboxform').modal('show');
	$("#theformcontent").html( dummyformcontent );
	$.ajax({ url: action, type:"GET", dataType:"text",
		success: data => { $("#theformcontent").html( data );
			$("#thestatus").select2({ width:"100%", dropdownParent:$('#theboxform') }); },
		error: () => $('#theboxform').modal('hide') });
	return false;
}
</script>
```

Conventions to keep (they're copy-pasted hack-to-hack): one fixed id on the list table (`itemlist`
here); `editform(-1)` for add, `editform(id)` for edit, `editform(id, true)` for clone; `data-url`
(edit) + `data-clone` (duplicate) on the edit button; the `#theboxform`/`#theformcontent` shell +
`dummyformcontent` snapshot. The edit form partial (loaded into `#theformcontent`) is a §15B
partial — starts at `modal-header`, no outer wrappers.

Lighter hack shapes exist too: a **single on/off setting** is one `{tpl:box-main}` status card + a
footer button opening a small confirm modal; a **docs/snippet page** is an `entry` + `<pre>` copy
blocks reusing `copycode()`.

---

## 19. Status, glyphs, and flags

- **Icons** — Font Awesome, `<i class="fa fa-fw fa-…"></i>`. `fa-fw` keeps icon columns aligned; always
  give an icon-only control a `title`.
- **Status glyph convention** (reuse so your columns read like native ones): `fa-cog text-muted` = wait,
  `fa-clock-o text-info` = hold, `fa-check-circle text-success` = accepted, `fa-times-circle text-warning`
  = cancelled, `fa-trash text-danger` = trash, `fa-pause-circle` = paused.
- **Server-driven class + icon** — the richest convention: the controller returns the colour suffix and
  icon name as data, and the template consumes them: `class="text-{item.stcl}"><i class="fa fa-{item.stic}"></i>`
  Prefer this when states are many/dynamic.
- **Status class by numeric id** — `<span class="status offtype{item.tid}">` / `fintype{id}` / `lst{id}`
  for CSS-per-state colouring.
- **Multi-flag cluster** — several *independent* glyphs prefixed before a name, each its own `IFSET`
  (active + blocked + private, say).
- **Mutually-exclusive badge** — a 3–4-branch `IFSET/ELSEIF/ELSE` chain rendering exactly one coloured
  icon+link — company/team/referrer/manager on a user row, or ban/admin/leader/member on a
  staff row.
- **Tri/quad-state via nested `IFSET`** — e.g. cron: `fa-cog fa-spin`(running)/`fa-check-circle`(idle)/`fa-clock-o`(never).
- **Value-driven class** — colour a plain `<b class="{item.avc}">{item.avail}</b>` by a server class,
  no icon (stock counts).
- **Country flags** by cardinality: single geo → `<i class="flag-{code}"></i>` (or `flagicon flag-{code}`,
  `flag flag-icon flag-{code}` — several spellings coexist); many geos → looped
  `<abbr class="label label-success" title="{name}">{code}</abbr>` badges; a 3-way fallback
  (single flag / text list / muted globe) for optional geo.
- **Device / target / online** — `<abbr class="mobicon mi{n}">` or a raw `fa-mobile`/`fa-desktop` swap;
  target-type glyphs `fa-rocket`(flow)/`fa-random`(split)/`fa-low-vision`(site)/`fa-globe`(url); an online dot `fa-circle` + `text-success`.
- **Rich content** goes in `<div class="entry">` (prose padding + heading/list/table styling).

---

## 20. Reference: AlterCPA-specific CSS classes

Not Bootstrap — AlterCPA's own helpers, shipped in `core.css`. Use these literally.

| Class | Meaning |
|-------|---------|
| `entry` | Prose block (`padding:10px 0`, styles headings/lists/tables). |
| `hh` + inner `<span>` | Form section divider: hairline with a centered label. |
| `sub` | Compact secondary header cell in a table (11px, centered). |
| `head` | Bold in-body group-separator row. |
| `noitems` | Centered muted empty-state cell. |
| `compact-td` | `font-size:0` cell so inline buttons butt together. |
| `fat` | `font-weight:bold`. |
| `black` (on `<a>`) | Force normal foreground colour (sortable headers). |
| `label-inline` | Zero-margin normal-weight label (radio/checkbox in tables). |
| `inline-checkbox` / `checkbox-inline` | Inline checkbox layout. |
| `empty-value` | The "search-for-blank" companion-checkbox label in filter modals. |
| `margin-hl` / `margin-hr` / `margin-vt` / `margin-vb` | 7px h / 5px v margins. |
| `short` / `shorter` / `tiny` | Fixed narrow widths (135 / 95 / 65px) for inline selects. |
| `no-arrows` | Hide number-input spinner arrows. |
| `no-margin` / `no-padding` | Spacing resets (`{tpl:no-padding}` is the macro form). |
| `flag` / `flagicon` / `flag-icon` | Country-flag icon spacing (multiple spellings coexist). |
| `mobicon mi{n}` | Device (mobile/desktop) glyph. |
| `graphcol` | Chart-series-toggle column in analytics headers. |
| `lister` / `pages` | Pagination `<nav>` wrappers (margin only). |
| `config-block` / `config-block-icon` / `config-block-name` / `config-block-line` / `config-block-descr` | KPI/stat tile internals (§4). |
| `widget-user-2` / `widget-user-header` / `widget-user-username` / `widget-user-desc` | Profile/KPI card internals. |
| `news-entry` / `news-entry-image` / `news-entry-text` | News card body. |
| `bonus-image` / `bonus-left` / `autotitle` | Bonus-shop product card. |
| `callaudio` | Inline call-recording `<audio>` player. |
| `fullcode` | Full-page CodeMirror editor wrapper. |
| `toolong` | Overflow handling for long values in modal key/value tables. |
| `makenicetable` | Marks a link that opens the shared `#nicetable` preview modal. |
| `expand-child` | Injected row holding AJAX drill-down content. |
| `editables` / `autocaps` / `checkall` / `checks` / `paybtn` | JS hooks (your JS wires these). |
| `usersearch` + `data-linkto` | AJAX user autocomplete (core JS). |

State classes from `core.css`: `has-success` / `has-error` (validation paint), `active`
(selected pill / button-group option), `info` / `active fat` (grouped rows), `overlay` + `fa-spin`
(loading veil), `progress … progress-bar-striped` (spinner shell).

---

## 21. Naming conventions and JS hooks

- **Ids**: search/add/import toolbars `xfsearch`/`xfadd`/`xfimport`/`xfilter`; the AJAX edit-modal shell
  `theboxform` + `theformcontent`; one fixed id on a hack's list table (`itemlist` in §18);
  form-scoped controls prefixed `the…` (`thename`, `thestatus`); edit buttons `editbtn{id}` with
  `data-url`/`data-clone`.
- **Flag families**: `z*` (single column) / `x*`,`w*` (group colspan) / `g*` (group label) in stats;
  `c*` in exports; `filter*` (admin-enabled filter fields); `sh*` (show-column) in offer tables;
  `is*`/`has*`/`can*` (capability gates).
- **Export vars**: parallel `{field_csv}` family; `x*` = colspan−1 for `ss:MergeAcross`.
- **Sign-encoded ids**: mixed entity kinds in one `<select>` via a negated id (`value="-{split.v}"` for
  splits vs positive for flows), mirrored in JS by the same negated id (`uid = -{id}`).
- **JS helpers worth reusing**: `safedel(url)` (typed-word delete) and `copycode(uid)` (clipboard)
  are a few lines each, written out above — paste them into your template; `initusersearch(el)` and
  `reset2s(name)` ship with the platform's `usersearch` and `select2` assets; `editform(uid, clone)` +
  `dummyformcontent` are the hack CRUD modal's shell (§15B, §18). Anything else a platform screen does
  in script — a status-button poll, an AJAX row drill-down, live subtotals, a login-panel switcher —
  is a function inline in that page: write the few lines you need in your own template.
- **Status-driven action button** — an AJAX poll swaps a button's text, `disabled`, and full class
  (`btn-primary`/`success`/`warning`/`default`) by a returned status string; reused for the split↔flow
  "glue" and domain-glue controls.
- **Self-embedding via a `face` toggle** — wrap a page's chrome (toolbar, box, scripts, modals) in
  `<!-- BEGIN face -->…<!-- END face -->` while the bare `<table>` always renders, so the *same file*
  serves both as the full page and as the AJAX fragment a row drill-down's `.load()` fetches. Give such a
  table a dynamic `id="{tid}"` to avoid DOM-id collisions when spliced into a host page.
- **Page-scoped script hook** — the footer renders `<!-- IFSET script --><script>{script}</script><!-- ENDIF -->`,
  the standard "one JS blob per page" injection point. Flash messages loop at the top of the content
  region; breadcrumb+title live in a `<!-- IFNOT hidebc -->` block a page can opt out of.

---

## 22. Checklist before you ship

- [ ] Every class that can be a `{tpl:…}` macro **is** one; only §20 helpers and grid/utility classes
      are literal. `box*` macros are paired with their `box-body`/`box-header` parts (§1).
- [ ] List page: collapsible toolbar (§2) → table box (§3) → mirrored `thead`/`tfoot` (§5) → `noitem`
      with correct `colspan` → `{pages}` nav.
- [ ] Table actions use `{tpl:btn-sm}` in a `{tpl:no-padding} compact-td` cell; icon buttons have
      `fa-fw`+`title`; deletes confirm.
- [ ] Row popups reuse the exact modal skeleton; per-row modals re-iterate the block after the table.
- [ ] Edit modals use the `dummyformcontent` snapshot shell (§18); the loaded partial has **no** outer
      modal wrappers and re-inits its widgets (select2 with `dropdownParent`).
- [ ] Live fields are `editables` and paint `has-success`/`has-error`; the real submit is blocked when
      saving is per-field.
- [ ] Every asset set (`select2`/`tablesorter`/`summernote`/…) is connected with `$core->site->set(…)`
      **before** `header()`.
- [ ] Multi-column forms group fields in `row`/`col-*`, break sections with `.hh`, descriptions in
      `help-block`.
- [ ] If this is a pixel/redirect/pre-login page, it's a plain self-contained document — no `{tpl:box}`.
- [ ] If it exports data, the `xls-*`/`xml-*` template is a transliteration of the screen table
      (§17.3): same loops/flags, `<Row>/<Cell>`, `ss:Type`, `x*` merge spans.
- [ ] You started from §18 or the nearest snippet above, not blank markup.

---

## 23. Pattern index

| You are building | Read |
|------------------|------|
| **The standard hack CRUD screen** | §18 — start here |
| Minimal list + search/add | §2.1, §5 |
| List + per-row modals, copy-link, stats menu | §5, §6, §14 |
| Composite multi-toolbar (search + add + import) | §2.2 |
| Faceted filter chips | §2.4 |
| Bulk-action check-all toolbar | §2.5, §11 |
| Card/box grid list · flat BEM list | §3, §5 |
| Settings hub (icon-tile grid) | §3 |
| Dashboard widgets / KPI tiles / charts | §4, §17.4 |
| Multi-column editor · live save · API modal | §10, §11, §14 |
| List + add modal + AJAX edit modal | §15A, §15B |
| Modal form partial (no wrappers) | §15B |
| Snapshot-shell AJAX edit modal | §15B, §18 |
| Grouped headers · configurable `z*` columns | §7.1, §7.2 |
| AJAX row drill-down | §5, §20 |
| Multi-currency grid · drag-reorder rules | §7.3, §7.4, §13 |
| Key/value and fact-list tables | §8 |
| Withdrawals · payment-method picker | §10, §12 |
| Token palette · feature matrix | §7.3, §12 |
| Log/debug viewer modal | §16 |
| File manager (forms-in-table) | §6, §12 |
| Generic horizontal field form | §10 (and [Building forms](hack-form.md)) |
| Full-page code editor · add-row-in-tbody | §6, §13 |
| Excel/XML export | §17.3 |
| Standalone pixel/redirect/error pages | §17.1 |
| Public/guest landing | §17.2 |
| Status badges, glyphs, flags | §19 |

Every pattern above is written out with its markup in the section named. Reuse the markup
class-for-class — that is what makes a hack page indistinguishable from a native one.
