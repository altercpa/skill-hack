# Working with Database

All database access goes through `$core->db`, an instance of the `MySQL` class. It is a
thin, hand-rolled wrapper over `mysqli` — there is no ORM, no query builder, and no
prepared statements. SQL is built by direct string interpolation, and safety comes from
the platform's boundary-escaping convention
(see [SQL safety](#sql-safety)).

## Connection

The connection is **lazy**. You never open it yourself — the first query opens it
automatically. If that fails, the whole request dies, printing `Database connection error!`.

The connection charset defaults to `utf8mb4`. Define `SQL_LOG` to collect every
query (with timing) into `$core->db->log[]`; `$core->db->queries` always counts the
number of queries run.

## Table Naming and Constants

Format: `cpa_x<hack>_<table>` — all lowercase Latin letters. The `cpa_` part is the
table prefix (`SQL_PREF`); the `x<hack>_` marks it as belonging to your hack.

Example for a `proxy` hack address table: `cpa_xproxy_addr`.

> **Tables are for data, not for settings.** A hack's settings — everything its Settings
> page edits — live as **one array** in the `hack_config` column the platform already
> gives it, or in one `$core->config()` block per scope (install-wide, or per user / per
> team as the tracker does it). Never create a `cpa_x<hack>_config` table, a column per
> setting, or a `name`/`value` settings table of your own. See
> [The Core → Configuration](hack-core.md#configuration) and
> [Configuration page](hack-config.md). Give a table only to rows the hack **produces or
> manages** — addresses, leads, jobs, log entries.

Declare every table as a `DB_*` constant in `start.php`, built from `SQL_PREF`, with
tab-aligned definitions and a trailing comment where useful:

```php
define( 'DB_XPROXY_ADDR',	SQL_PREF . 'xproxy_addr' );	// Proxy addresses
```

Use the constant everywhere — never hard-code the table name:

```php
$core->db->row( "SELECT * FROM ".DB_XPROXY_ADDR." WHERE proxy_id = '$id' LIMIT 1" );
```

> The `cpa_` prefix is mandatory in `schema.php` array keys even on installs that use
> a different `SQL_PREF` — the schema checker substitutes the real prefix for the
> leading `cpa_` itself (see [Schema](#the-schema-system)).

## Querying the Database

### Query functions

Run a SQL string and get the result in the shape you want. Every accessor returns a
**falsy** value on both an empty result and a query error, so a single falsy check
covers both — but the *type* of that falsy value differs by method, which matters when
you compare with `===` or want to tell "no rows" from "error" apart.

| Function | Returns on success | No rows | Query error |
|----------|--------------------|---------|-------------|
| `query( $sql )` | mysqli result / `true` | — | `false` |
| `field( $sql )` | scalar (first column of first row) | `null` | `false` |
| `row( $sql )` | associative array of one row | `[]` | `[]` |
| `col( $sql )` | flat array of the first column | `[]` | `[]` |
| `icol( $sql )` | array keyed `col0 => col1` | `[]` | `[]` |
| `data( $sql )` | array of associative rows | `[]` | `[]` |
| `it( $sql )` | generator of associative rows | yields nothing | yields nothing |
| `ic( $sql )` | generator of first-column values | yields nothing | yields nothing |
| `ii( $sql )` | generator yielding `col0 => col1` | yields nothing | yields nothing |

> **Only `query()` and `field()` ever return `false`.** `row()`, `col()`, `icol()` and
> `data()` return an **empty array `[]`** when the query finds no rows *and* when it
> fails — never `null` or `false`. `field()` is the one that returns `null` for an empty
> result and `false` on error. The manual cycle's `one()` also returns `[]` at
> end-of-rows.

Because mysqli returns every value as a string (or `null`), cast results when you
need numbers, and guard `row()`/`field()` before use — `field()` returns `null` for an
empty result, and aggregates over an empty set (`SUM`/`MAX`/`AVG`) come back as `NULL`:

```php
$cnt = (int) $core->db->field( "SELECT COUNT(*) FROM ".DB_XPROXY_ADDR." WHERE proxy_status = 1" );

$p = $core->db->row( "SELECT * FROM ".DB_XPROXY_ADDR." WHERE proxy_check = 1 LIMIT 1" );
if (empty( $p )) return false;
```

`use empty()` is the safe universal guard for `row()`/`field()` — it covers `null`,
`[]` and `false` alike.

### Iterators for large sets

`it()`, `ic()` and `ii()` are generators (`yield`) — they stream rows one at a time
and free the result at the end, so they never load the whole set into memory. Use them
for large scans; use `data()`/`col()`/`icol()` for small-to-medium sets you want as a
plain array.

```php
// Stream a large table without buffering it all
foreach ( $core->db->it( "SELECT * FROM ".DB_XPROXY_ADDR." WHERE proxy_check = 1" ) as $d ) {
	// ... process $d ...
}
```

`ic()` / `ii()` are the streaming equivalents of `col()` / `icol()`.

### Manual query cycle

When you need full control over a result handle, use the `start → one → stop` cycle.
`start()` runs the query and returns the raw handle, `one()` fetches the next row,
`stop()` frees it:

```php
$q = $core->db->start( "SELECT * FROM ".DB_XPROXY_ADDR );
while ( $row = $core->db->one( $q ) ) {
	// ... process $row ...
}
$core->db->stop( $q );
```

## CRUD Wrappers

These wrappers build the SQL for you from table + data/condition arrays. They cover
the common cases; drop to raw `query()` for anything more complex (JOINs, GROUP BY,
expressions in SET, etc.).

### Reading

```php
// One row by condition: SELECT * ... WHERE ... LIMIT 1
$core->db->get( DB_XPROXY_ADDR, [ 'proxy_id' => $id ] );

// All rows by condition: SELECT * ... WHERE ...
$core->db->all( DB_XPROXY_ADDR, [ 'proxy_status' => 1 ] );

// All rows, no condition
$core->db->all( DB_XPROXY_ADDR );
```

### Writing

```php
// INSERT
$core->db->add( DB_XPROXY_ADDR, [ 'proxy_name' => 'Meow', 'proxy_ip' => '127.0.0.1' ] );

// ID of the row just inserted
$id = $core->db->lastid();

// REPLACE INTO (delete-then-insert by primary/unique key)
$core->db->replace( $table, $data );

// INSERT ... ON DUPLICATE KEY UPDATE (insert, or update in place — preserves the row/ID)
$core->db->addupd( $table, $data );

// UPDATE ... SET ... WHERE ...
$core->db->edit( DB_XPROXY_ADDR, [ 'proxy_status' => 1 ], [ 'proxy_id' => $id ] );

// DELETE FROM ... WHERE ...
$core->db->del( DB_XPROXY_ADDR, [ 'proxy_id' => $id ] );
```

### Conditions: array vs. string

Every wrapper that takes a `$where` accepts either form:

```php
// Array — keys ANDed together: `proxy_id` = '$id' AND `proxy_check` = '1'
$core->db->edit( DB_XPROXY_ADDR, [ 'proxy_status' => 1 ], [ 'proxy_id' => $id, 'proxy_check' => 1 ] );

// String — passed through verbatim (use for ranges, OR, IN, expressions)
$core->db->edit( DB_XPROXY_ADDR, [ 'proxy_status' => 1 ], "proxy_fail < '$max' OR proxy_id = '$id'" );
```

The array form only produces `column = 'value'` equality joined by `AND`. For anything
else (`<`, `IN`, `OR`, `LIKE`, …) write the condition as a string — and remember it is
**not** escaped for you, so the values you interpolate must already be safe.

### Binary values: the `X'…'` convention

`add`, `replace`, `addupd`, `edit` and array-form `where` all understand a binary
literal escape: if a value **starts with** `X'`, the builder emits it as a raw
`X'<hex>'` SQL literal instead of a quoted string. Pass `"X'" . bin2hex( $raw )` — the
prefix `X'`, then hex digits, no trailing quote (the builder adds it):

```php
// Store a packed IPv4/IPv6 as a binary column
$core->db->add( DB_XPROXY_ADDR, [
	'proxy_bin'	=> "X'" . bin2hex( inet_pton( $ip ) ),
] );
```

### Helpers

| Function | Purpose |
|----------|---------|
| `lastid()` | `mysqli_insert_id()` — ID of the last inserted row |
| `affected()` | `mysqli_affected_rows()` — rows touched by the last write |
| `escape( $s )` | `mysqli_real_escape_string()` — escape one value |
| `error()` | last MySQL error string |

> **Empty-condition guard.** `edit()` and `del()` **bail out and return `false`** when
> their `$where` is empty — there is no accidental full-table `UPDATE`/`DELETE`.
> `add()`/`replace()` require an array of data; `addupd()` additionally returns `false`
> on empty data. Always check the return value of a write you expect to touch rows.

## SQL safety

There are no prepared statements. The platform relies on a strict boundary
convention, and you must follow it:

- **Request input is pre-escaped.** Everything in `$core->get` / `post` / `cookie` /
  `files` / `server` is run through `addslashes()` at the Core boundary. You can
  interpolate those values straight into SQL string literals (`'$value'`), but cast to
  `(int)` or run through `$core->text->*` sanitizers anywhere a value lands in numeric
  or identifier context (not inside quotes).
- **Data read back from the DB is *not* escaped.** Re-interpolating a value you just
  `SELECT`ed into another query is a second-order injection risk. Re-escape it first
  with `addslashes()` or `$core->text->ass()`:

  ```php
  $name = $core->db->field( "SELECT proxy_name FROM ".DB_XPROXY_ADDR." WHERE proxy_id = '$id'" );
  $core->db->query( "UPDATE ... SET proxy_copy = '".$core->text->ass( $name )."' WHERE ..." );
  ```

- **Internally built strings** (anything not from a request and not already escaped)
  need `addslashes()` before going into a quoted literal.

The CRUD wrappers (`add`/`edit`/`where`/…) do **not** escape values — they quote them
as-is. The data you hand them must already be safe by the rules above.

## The Schema System

A hack keeps its own tables in sync automatically through `schema.php` at the hack
root. This file is the declarative model of every table the hack owns.

> **Important format note.** `schema.php` is **not** a list of `CREATE TABLE`
> statements. It is a PHP file that **`return`s an array** describing the tables. The
> platform diffs that array against the live database and emits the `ALTER`/`CREATE`
> statements itself. A hand-written `$schema['x'] = "CREATE TABLE …"` file is wrong and
> will not be applied — always generate it (see below) or follow the array format
> exactly.

### File format

```php
<?php

/*  ...standard boxed header...  */

return [
	'cpa_xproxy_addr' => [
		'main' => [
			'myisam'	=> [ 'engine' => 'MyISAM' ],
			'aria'		=> [ 'engine' => 'Aria', 'options' => 'page_checksum=0 transactional=0' ],
		],
		'field' => [
			'proxy_id' => [
				'type'		=> 'int(10) unsigned',
				'default'	=> NULL,
				'extra'		=> 'auto_increment',
				'outdated'	=> false,
			],
			'proxy_status' => [
				'type'		=> 'tinyint(3) unsigned',
				'default'	=> '0',
				'extra'		=> '',
				'outdated'	=> false,
			],
			// ... more fields ...
		],
		'index' => [
			'PRIMARY'		=> [ 'type' => 1, 'field' => [ 'proxy_id' ] ],
			'proxy_status'	=> [ 'type' => 0, 'field' => [ 'proxy_status' ] ],
		],
	],
	// ... more tables ...
];
```

**Table key** — the full table name including the `cpa_` prefix. On installs with a
non-`cpa_` prefix, the checker swaps the leading four characters for the real
`SQL_PREF`, so the key must always be written with `cpa_`.

**`main`** — engine variants. The platform picks `aria` when the server supports it,
otherwise `myisam`. Aria tables normally carry
`options => 'page_checksum=0 transactional=0'`; fixed-row tables add
`row_format=FIXED`.

> **Charset caveat.** The `mysqli` *connection* runs `utf8mb4` in both products. But newly
> **created** tables differ: AlterCPA **Lite** emits `DEFAULT CHARSET=utf8mb4`, while AlterCPA
> **Pro** currently emits `DEFAULT CHARSET=utf8`. Don't assume `utf8mb4` storage on Pro.

> **`text`-type columns ignore their schema `default`.** The generator forces
> `DEFAULT ''` on any `text` column regardless of what `default` you set, so schema
> examples carry `'default' => ''` for `text` fields.

**`field`** — every column, in order. Columns are always created `NOT NULL`; the
`default` value is rendered automatically (`NULL` → no explicit default, `'0x…'` →
binary `X'…'` default, otherwise a quoted literal). `extra` holds things like
`auto_increment`. `outdated => true` marks a retired column — it is skipped when
creating a fresh table (used to phase columns out without breaking old installs).

**`index`** — keys by name. `type` is `0` = regular `KEY`, `1` = `UNIQUE` / `PRIMARY`,
`2` = `FULLTEXT`. `field` is the ordered list of columns in the key.

### How the model is applied (declarative + destructive)

The platform's schema check compares the model to the live table structure and, table by
table:

- **creates** any table that doesn't exist;
- **adjusts** engine/options if they drifted;
- **adds / changes / drops** columns so the live table matches `field` exactly;
- **adds / changes / drops** indexes so they match `index` exactly.

This is the key consequence: the schema is **authoritative**. Any column or index
present in the live table but **absent from `schema.php` is DROPPED**, and any manual
change to a hack table is reverted on the next check. This is the real reason for the
rule below.

> **Never modify a hack's tables by hand while the hack is active.** Your changes will
> be erased the next time the schema is checked. Deactivate the hack first, make your
> changes, regenerate `schema.php`, then reactivate.

### Generating `schema.php`

You don't write the array by hand — you build the tables in MySQL and let the platform
serialize them, through the control-panel generator:

1. Connect via phpMyAdmin / MySQL Workbench.
2. **Deactivate the hack.**
3. Create the tables with the desired engine, columns and indexes, following the
   `cpa_x<hack>_<table>` naming.
4. Open the hack's **Files → Schema generator** in the control panel.
5. The table list is pre-filled automatically (every table whose name starts with
   `cpa_x<hack>`); review it.
6. Pick an action and submit:
   - **Update schema file and increase version ID** — writes `schema.php` *and* bumps
     `db` in `update.json` (use this for a change you want pushed to existing installs).
   - **Only update schema file** — writes `schema.php` without touching the version.
   - **Download schema without updating file and version** — returns `schema.php` as a
     download, changing nothing on disk.
7. Reactivate the hack.

Under the hood the generator reads the live table structure, keeps the requested tables,
and writes the array with `xxvar()` (the platform's array pretty-printer, see
[Global functions](hack-func.md)).

Two rules that no generator enforces for you: always write columns
`NOT NULL DEFAULT …` (the rebuild makes every column `NOT NULL` anyway), and bump
`update.json`'s `"db"` yourself when you want the change to reach installs that already
run the hack.

### Versioning and auto-apply

The applied schema version is tracked per hack so changes roll out automatically:

- `update.json` carries the target version: `{ "db": 14, ... }`.
- The platform stores the version last applied to each install in
  `DB_HACK.hack_db`.
- **On install/activation** the schema is applied immediately and `hack_db` is set to
  the file's `db`.
- **On platform upgrade**, every active hack whose `update.json` `db` is **greater
  than** the stored `hack_db` has the schema check re-run and `hack_db` bumped.

So to ship a schema change to existing installs: change the tables, regenerate
`schema.php`, and make sure `update.json`'s `db` is **incremented** (the control-panel
generator's "increase version ID" option does this for you). To retire a column without breaking older
installs, mark it `outdated => true` rather than deleting it outright.
