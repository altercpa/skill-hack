# Main Class

The `main.php` file contains a class responsible for configuration parameters, loading modules and libraries, connecting templates, styles and scripts.

## Syntax

The class is named `hack_**name**` and inherits from `thehack`:

```php
class hack_proxy extends thehack {
	public function init() {
		// Constructor code (no parameters)
	}

	public function stop() {
		// Destructor code (no parameters)
	}
}
```

Do not override the built-in constructor/destructor — use `init()` and `stop()` instead. If only built-in functions are needed, `main.php` can be omitted.

## Built-in Methods

All methods and fields are accessed via `$core->hack->hackname->funcname()`.

### Configuration

| Method | Description |
|--------|-------------|
| `$core->hack->(name)->conf[$field]` | Read-only configuration array |
| `$core->hack->(name)->conf()` | Returns entire configuration array |
| `$core->hack->(name)->conf( $field )` | Returns specific field value |
| `$core->hack->(name)->conf( $field, $val )` | Sets field to value |

The whole set of the hack's settings is **one array**, stored serialized in the single
`hack_config` column of the hacks table. `conf( $field, $val )` assigns one key of that
array and rewrites the column whole — it is a convenience, not a separate per-setting
store. Never give a setting a database column or a table of its own; see
[Configuration page](hack-config.md).

> **`conf()` is for declared settings, not for runtime state.** Saving the hack's Settings
> form rebuilds `hack_config` from the `param.json` fields alone and writes it whole, so any
> key your code added with `conf()` and `param.json` does not declare is deleted on the next
> Save. A seed, a derived secret, a cached remote id belongs in a `$core->config()` block —
> a different table, untouched by that save. See
> [Configuration page → `param` destroys every key you did not declare](hack-config.md) and
> [The Core → Configuration](hack-core.md#configuration).

### Loading Modules and Libraries

| Method | Description |
|--------|-------------|
| `$core->hack->(name)->(library)` | Loads and initializes library (once) |
| `$core->hack->(name)->mod( $name )` | Loads module by name (`include_once mod/<name>.php`) |
| `$core->hack->(name)->mod( $name, $func )` | Loads module and immediately calls `$func( $core )` |
| `$core->hack->(name)->mod( $name, true )` | `include` (not once) and **return** the file's return value |

> `thehack` defines a `__call` that returns `false` for any method it doesn't have, so a
> call to a non-existent helper fails silently rather than raising an error. Don't rely
> on this — it only means typos won't fatal; they just return `false`.

Example library access:

```php
$core->hack->proxy->checker->run( 123 );
```

### Templates and Assets

| Method | Description |
|--------|-------------|
| `$core->hack->(name)->tpl( $part, $name )` | Loads template section (no .tpl extension) |
| `$core->hack->(name)->css( $name )` | Includes CSS file (no .css extension) |
| `$core->hack->(name)->js( $name )` | Includes JS file (no .js extension) |
