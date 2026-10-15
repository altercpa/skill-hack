# Language Files

Language files let you add custom text in system languages and modify existing text. **Using text directly in hacks without language files is forbidden.**

## Creating Language Files

Files are named by language code: `ru.php` for Russian, `en.php` for English. Each file returns an associative array where keys are identifiers and values are translated text. Keys should begin with the hack name.

```php
<?php
return [
	'proxy_config_h' => 'Proxy configuration',
	'proxy_config_t' => 'In this section, you can set up the configuration.',
];
```

## Using in Code

Access strings via the `lang` array on `$core`, which merges all language files:

```php
$core->lang['proxy_config_h']
```

## Cache

The language cache clears automatically on updates and hack activation. During development, clear it manually:

**Management → Settings → Diagnostics → Caching → Language files → Purge**

## Adding a New Language

New language files can be partial translations — missing phrases are pulled from the base language (`ru` or `en`) automatically.

Register a new language via the `languages` handler:

```php
function hack_pl_languages( $core, $lang ) {
	$lang['pl'] = [ 'name' => 'Polski', 'base' => 'en', 'guess' => 'pl' ];
	return $lang;
}
```

The array value accepts:

| Field | Description |
|-------|-------------|
| `name` | User-visible language name |
| `base` | Foundation language (`ru` or `en`) — missing keys fall back to it |
| `guess` | String or array; matched as a case-insensitive substring of the `Accept-Language` header |
| `locale` | Optional system locale passed to `setlocale()` (e.g. `pl_PL.UTF-8`) |

Register in the initialization function:

```php
$core->handle( 'languages', 'hack_pl_languages' );
```
