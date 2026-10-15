---
name: altercpa-hacks
description: Complete guide to developing hacks (extensions) for the AlterCPA platform - Lite, Pro and Core. Use whenever writing, designing, reviewing or debugging AlterCPA hack code - the startup file and main class, the $core god-object, modules and libraries, language files, templates and styling, the Settings page, API applications, cron tasks, database schemas, advertising and delivery connectors, and the auto-update service. Also carries the mandatory AlterCPA code style. Triggers on any mention of an AlterCPA hack, the $core object, the hack/<name>/ layout, hack templates or forms, hack API endpoints, or a request to extend or customize AlterCPA.
---

# AlterCPA — Hack Development

**Hacks** are extensions for AlterCPA. They add features and customize the platform: new
control-panel pages, modules, libraries, API applications, database tables, cron tasks,
advertising and delivery connectors. A hack lives in its own `hack/<name>/` folder, is
loaded by a startup file plus a main class, and is installed, configured and activated
from **Control → Hacks and modules**.

The platform code itself is not readable — you extend it through the documented
interfaces alone. Everything you need is in `reference/`. Read the page for the piece you
are building **before** writing it; these are exact APIs, not summaries.

---

## The three rules that override everything

### 1. Code style is mandatory

> All hack code MUST follow **[Code Style](reference/hack-style.md)** — tabs, brace and
> spacing conventions, naming, the boxed file header, the comment style. New code must be
> indistinguishable from the platform's own. Read it first, every time.

### 2. Settings are one array, never a schema

> Every setting of a hack is a **key of one associative array**: `$core->hack-><name>->conf`
> (stored serialized in the single `hack_config` column), or one `$core->config( '<name>' )`
> block per scope, written back whole with `reconf()`.
>
> **Never** give a setting a database column, a settings table (`cpa_x<hack>_config`), or a
> config block of its own. A Settings page with twelve inputs is twelve keys of one array,
> not twelve columns. Hack tables are for **data** — the rows the hack produces or manages.
>
> *Per scope* is the one reason for more than one block: per-user or per-team settings are
> one block per signed scope id, `'<name>' . $uga` — positive a user, negative a team. Still
> one array per block.
>
> Two traps live here. The Settings **Save** button rebuilds `hack_config` from `param.json`
> alone and deletes any key your code added but did not declare. And `cpa_config.name` is
> `varchar(20)`, so a long or id-suffixed key silently collapses into a shared row.
>
> Read **[Configuration page](reference/hack-config.md)** and
> **[The Core → Configuration](reference/hack-core.md#configuration)** before designing any
> settings storage.

### 3. The hack name is an identifier

> Lowercase Latin letters and digits, first character a letter — no capitals, hyphens or
> underscores. The name is concatenated into PHP class and function names (`hack_<name>`,
> `hack_<name>_<lib>`) and into table names (`cpa_x<name>_<table>`), so anything else
> produces invalid identifiers. It also has to fit: config keys live in a
> `varchar(20)` column.

---

## Targets: Lite, Pro, Core

**Lite** and **Pro** are the two products a hack is written against; **Core** is the shared
base. Pro is the superset — it adds offers, orders, postbacks, deliveries and the helpers
that go with them. A hack meant for both must not rely on Pro-only fields, helpers or
events; each reference page marks what is Pro-only.

---

## How a hack is built

Start here, in order:

1. **[Introducing hacks](reference/hack-intro.md)** — what a hack is, its folder layout,
   installing and exporting one.
2. **[Startup file](reference/hack-start.md)** — `start.php`: the handler manifest
   (`init`, `api`, `cron-*`, `tpl`, `lib`, `delivery`, `adv`).
3. **[Main class](reference/hack-main.md)** — `main.php`: configuration and resource loading.

## The `$core` god-object

Almost everything runs through the global `$core`, which is also the first argument of every
procedural module function.

| Reference | What it covers |
|-----------|----------------|
| **[The Core](reference/hack-core.md)** | `$core` itself — events, configuration, cache, sessions, locks, request flow |
| **[User and request state](reference/hack-user.md)** | `$core->user` (identity, rights) and `$core->app` (routing, CSRF) |
| **[Text processing](reference/hack-text.md)** | `$core->text` — sanitizers, validators, formatters |
| **[Site elements](reference/hack-site.md)** | `$core->site` — header, footer, menus, flash messages, assets |
| **[Building forms](reference/hack-form.md)** | `$core->site->form()` — a whole form from a typed field list |
| **[Media library](reference/hack-media.md)** | `$core->media` — uploads, storage, image resizing |
| **[E-mail sending](reference/hack-email.md)** | `$core->email` — sending and mail configuration |
| **[Global functions](reference/hack-func.md)** | Always-loaded helpers — IP, JSON, dates, CURL, … |

## Building the hack out

| Reference | What it covers |
|-----------|----------------|
| **[Modules](reference/hack-mod.md)** | Controllers — the pages and actions |
| **[Libraries](reference/hack-lib.md)** | Models — the data-access layer |
| **[Language files](reference/hack-lang.md)** | Multilingual text |
| **[Templates and styling](reference/hack-tpl.md)** | The template engine — blocks, directives, macros, caching |
| **[Template patterns](reference/hack-tpl-patterns.md)** | The UI cookbook — toolbars, boxes, tables, modals, forms |
| **[Configuration page](reference/hack-config.md)** | `param.json` and the Settings screen |
| **[Creating API](reference/hack-api.md)** | API applications, the response envelope, authorization |
| **[Task schedule](reference/hack-cron.md)** | `cron-*` slots and periodic tasks |
| **[Working with database](reference/hack-db.md)** | `$core->db`, SQL safety, `schema.php`, migrations |
| **[Advertising networks](reference/hack-adv.md)** | Traffic-expense connectors |
| **[Delivery services](reference/hack-delivery.md)** | Logistics connectors (Pro only) |
| **[Automatic updates](reference/hack-update.md)** | `update.json` and version distribution |

---

## Reading order by task

| You are | Read |
|---------|------|
| New to hacks | `hack-intro` → `hack-start` → `hack-main`, then skim `hack-core` |
| Adding a control-panel page | `hack-mod` + `hack-tpl` + `hack-tpl-patterns` (§18 is the screen to copy) |
| Adding a Settings screen | `hack-config`, and `hack-core` §Configuration for the storage rules |
| Building a form | `hack-form` — the typed field list, before hand-writing markup |
| Building an integration or API | `hack-api`, plus `hack-func` for the helpers |
| Touching data | `hack-db` for schema and safety, `hack-lib` for the access layer |
| Shipping an update | `hack-update`, and bump `update.json`'s `db` if the schema changed |
| **Always** | `hack-style` |

---

## Things that bite

- **A page must be indistinguishable from a native one.** Reuse the markup in
  `hack-tpl-patterns` class for class; do not invent your own components. There is no tab
  component — hubs are grids of link cards.
- **SQL is string-interpolated.** There are no prepared statements. Escape at the boundary
  by the convention in `hack-db` §SQL safety, and cast ids with `(int)`.
- **`schema.php` is authoritative and destructive.** A column present in the live table but
  absent from `schema.php` is dropped on the next check. Never hand-edit a live hack table
  while the hack is active.
- **A filter callback must `return` its data.** Forgetting the `return` blanks the value for
  everyone downstream.
- **A cross-site POST arrives empty.** The CSRF guard discards the whole POST array, not just
  `action`. Drive state changes through `action=` and check the result.
- **Register lazily-loaded subsystems in `init`.** `$core->media` and friends are constructed
  on first access; a handler registered later never fires.
