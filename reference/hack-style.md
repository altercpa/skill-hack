# Code Style

Style conventions used by Anton 'AlterVision' Reznichenko across AlterCPA (Core, Lite, Pro).
**Follow these exactly when writing or editing any hack code.** Goal: new hack code must be
indistinguishable from the existing platform code.

## File layout

- Every PHP file starts with the boxed header comment:
  ```php
  <?php

  /*******************************************************************************

   *
   *  AlterCPA Core
   *  Created by AlterVision - www.altercpa.pro
   *  Copyright (c) 2013-2026 Anton Reznichenko
   *

   *
   *  File: 			hack / proxy / main.php
   *  Description:	Short description of the file
   *  Author:			Anton 'AlterVision' Reznichenko - altervision13@gmail.com
   *

  *******************************************************************************/
  ```
  `File:` path uses ` / ` separators. Hacks may have their own start year (e.g. `2023-2026`).
- Files often end with the comment `// end. =)`. Never add a closing `?>`.
- One class per file for libraries; procedural function sets for modules and helpers.
- No `declare(strict_types=1)`. No namespaces. No type hints, no return types,
  no property types — code must stay readable and 7.2-compatible in style.

## Indentation and whitespace

- **Tabs** for indentation, always. (Space-indented blocks occasionally found in
  legacy code are mistakes, not style — never imitate them.)
- Tab-alignment for grouped declarations and array literals:
  ```php
  public	$get		= [];
  public	$post		= [];
  private	$alive		= true;
  ```
  ```php
  $sess = [
  	'meta'	=> ...,
  	'data'	=> ...,
  ];
  ```
- Spaces **inside** call parentheses: `somefunction( $param1, $param2 )`.
  No space between function name and `(`. Empty parens stay empty: `time()`.
  Occasionally a space appears before the paren in declarations
  (`public function cache ( $name )`) — acceptable, not required.
- Control structures: `if ( $expression )` with inner spaces when the condition is an
  expression; when the condition is a single function call the outer parens hug it:
  `if (isset( $x ))`, `if (file_exists( $f ))`, `if (!empty( $a ))`.
- Negation with spaces: `if ( ! $tz )`. Also `!$x` inside compact compound conditions.
- Casts with a space: `(int) $id`, `(string) $v`.
- Concatenation: spaces around `.` in long expressions (`'config.' . $name`), compact
  in short glue chains (`$mt.'='.$msg`). Interpolation preferred where possible:
  `"SELECT * FROM ".DB_USER." WHERE user_mail = '$email'"`, `"{$user}@gmail.com"`.

## Control flow idioms

- Single-statement `if` without braces on one line: `if ( ! $id ) return false;`
- `else` / `elseif` with a single statement goes on the same line as the closing brace, without braces:
  ```php
  if (isset( $menu['tracker'] )) {
  	$menu['tracker'][] = 'myhack';
  } else $menu['tools'][] = 'myhack';
  ```
- Compound one-liners are common: `foreach ( $uls as $ui ) if ( $ui = (int) $ui ) $list[] = $ui;`
- `elseif` (one word), never `else if`.
- Guard clauses / early returns preferred over deep nesting.
- `switch`: case body starts on the next line at the same indent as `case`, or short
  one-liners `case 1: $x = 'a'; break;`. `default:` last, often without `break`.
- Assignment inside condition is idiomatic: `if ( $result = $this->query( $sql ) ) {`.
- Correlated division guards: a dependent metric is fetched only when its
  denominator is truthy, then the *numerator* guards the division:
  `$at = $mt ? $this->appt() : 0;` … `$xt = $at ? ( $at / $mt * 100 ) : 0;`
  Valid, but preserve the correlation when editing.

## Ternaries and boolean style

- Prefer `empty()` with swapped branches over `!empty()`:
  `$foo = empty( $bar ) ? 'meme' : 'bubu';` — NOT `!empty( $bar ) ? 'bubu' : 'meme'`.
- Explicit boolean ternary for validators: `return filter_var( ... ) ? true : false;`
- `??` used heavily for input/array access: `$this->get['s'] ?? ''`, chained:
  `$this->post[$name] ?? $this->get[$name] ?? null`.
- `?:` shorthand for fallbacks: `$tz ?: 'Europe/Moscow'`.

## Naming

- Functions: all-lowercase, usually without underscores, short and fused:
  `remoteip`, `guesslang`, `startsess`, `ip2hql`, `xxenc`, `c32u`, `randtoken`.
  Converter naming pattern `x2y`: `ip2bin`, `bin2ip`, `form2date`, `log2dec`.
- Module entry functions are prefixed by location: `base_login_page`, `config_skin_*`,
  `hack_<name>_init`, `hack_<name>_menu`.
- Variables: short lowercase abbreviations, often 2–4 letters: `$sd`, `$tmr`, `$ld`,
  `$lv`, `$cn`, `$dt`, `$sess`. Loop pairs like `$li => $lv`, `$mk => $mv`.
- Classes: historical mix — `Core`, `MySQL`, `Template`, `User`, `eMailSender`,
  `SiteElements`, `MediaLib` — but lazy-loaded libs in `lib/` have **lowercase class
  names equal to the file name** (the lazy loader instantiates the class named after the file);
  hack overrides are `hack_<name>` classes.
- Constants: UPPERCASE; DB tables as `DB_*` constants built from `SQL_PREF . 'name'`,
  defined in start files with a trailing comment:
  `define( 'DB_XPROXY_ADDR', SQL_PREF . 'xproxy_addr' );	// Proxy addresses`

## Comments

- English only. A short `//` comment line above each logical block (roughly every
  3–10 lines). Inline trailing comments for constants and array fields.
- **One line, a few words, what the block does:** `// Get the last click ID`,
  `// Rebuild the tail`, `// Save the statistics`. Never two comment lines in a row outside
  a section divider, never a paragraph. A comment does not explain why the code is written this way, what it
  rejected or which case it guards — the code says what it does, the reasoning lives
  outside the source.
- Temporarily disabled code lines are prefixed with `#` (not `//`):
  `#	$email = $core->text->email( $data['email'] );` — keep them, they document
  intent (e.g. a deliberately relaxed validation).
- Section dividers inside files/classes:
  ```php
  //
  // Cache management
  //
  ```
- A function header is the same one `//` line — no PHPDoc blocks, no `@param`, no notes
  on the parameters, the names and the code carry them:
  ```php
  // Print numbered value
  function numinf ( $l, $n ) {
  ```

## Language feature preferences

- Short array syntax `[]` everywhere.
- Optional params default to `= false` for "not provided" (`$func = false`,
  `$field = false`); `= null` only when null is semantically checked (`$value = null`).
- `sprintf`/`printf`-style templating is preferred over concatenation for URLs and
  formats; format string tables stored in arrays (`$this->urls`).
- `@` error suppression is accepted for cleanup/IO calls: `@unlink`, `@fclose`,
  `@mysqli_free_result`, `@file_get_contents` — when failure is expected and handled
  by the surrounding logic.
- `die()` for fatal unrecoverable errors (`die( 'Database connection error!' )`).
- Generators (`yield`) used for DB iterators; `static` local caches in helper
  functions (`static $nw = [];`).
- Polyfills and compatibility helpers live at the bottom of the platform's global-function file
  under `// Back compability` — never per-file. Example: `sanitize_string()` replaces
  the removed `FILTER_SANITIZE_STRING`.
- **Request values into string functions: always `tostr()`** (arrays → `''`, everything
  else → `(string)`). Preferred over
  `is_string( ... ?? false )` guards and over bare `(string)` casts (which warn on
  arrays and yield the literal `"Array"`). Combine with `?? ''` when the key may be
  absent: `stripslashes( tostr( $core->post['code'] ?? '' ) )`. When parameter
  *presence* drives the logic, keep the `isset()` guard and wrap only the value.
  When a value is used several times, normalize once into a short var:
  `$ss = tostr( $core->get['s'] ?? '' );`
- Target compatibility: **PHP 7.2 → 8.4.** Any standalone file deployed outside the
  platform (no `$core`, run on remote boxes) keeps the strictest 7.2 floor: no arrow
  functions, no spread with string keys, no typed properties; `??`, `[]` and
  generators are fine.

## Architectural conventions (write code that fits them)

- `$core` (class `Core`) is the god-object; it is always the first parameter of
  procedural module functions: `function base_login_page( $core )`.
- `$core->req( $name )` reads a request param POST-first, then GET, returns
  **null** when absent (never trust the type of what it returns).
- Libraries are lazy-loaded on first access as `$core-><name>`; the loaded class name is the lowercase
  file name, with an optional hack override class `hack_<name>`.
- Modules run via `$core->mod( 'dir-file', 'entry_function' )`.
- Event system: `$core->handle( 'event', 'callback' )` to subscribe,
  `$core->process( 'event', $info )` to fire, `$core->filter( 'event', $data )` to
  transform, `$core->cando( 'event' )` to test.
- Hacks (plugins) live in `hack/<name>/` with `start.php` returning
  `[ 'init' => 'hack_<name>_init' ]`; tables prefixed `x<name>_`.
- All request input (`$core->get/post/cookie/files/server`) is pre-escaped with
  `addslashes` at the Core boundary; SQL is
  built by direct interpolation of those values. Values must be cast `(int)` or run
  through `$core->text->*` sanitizers when used in numeric/identifier context.
  Binary SQL literals use the `"X'<hex>"` prefix convention understood by
  the `$core->db` builders (`add`, `replace`, `addupd`, `edit`, array-form `where`). **Values read back from the DB are not escaped** —
  re-interpolating them into SQL needs `addslashes()`/`text->ass()` (second-order
  injection risk).
- DB layer return shapes (class `MySQL`): `field()` → string / **null** (no rows) /
  **false** (error); `row()`/`one()` → assoc array / `[]` / `[]`;
  `col()/icol()/data()` → array / `[]`; `it()/ic()/ii()` → generator. Check `row()`
  with `empty()`/`isset( $r['pk'] )` before field access; mysqli returns every value
  as string or null, so aggregates (`SUM`/`MAX`/`AVG` over empty sets → NULL) need care.
- JSON helpers: `xxenc` (encode), `xxdec` (decode), `xxend` (encode + addslashes for
  SQL), `xxload`/`xxsave` (files). Use them instead of raw `json_*` where applicable.
- Templates via `$core->tpl->vars/block/output`, skin via `$core->skin`,
  language strings via `$core->lang['key']`.
- Redirects: `$core->go( $url )`, `$core->msgo( $msg, $url )`; JSON responses:
  `$core->json( $data )`; both call `$core->stop()` and never return.

// end. =)
