# Modules

Modules implement the main functionality — page processing, scheduler tasks, and API functions. Within MVC, they are most similar to "controllers."

## Creating a Module

Module files live in the `mod/` folder. File naming: lowercase Latin letters, numbers, and hyphens only — no capitals or special characters.

Recommended naming convention: `hack_<hackname>_<funcname>` (e.g., `hack_proxy_list`).

### Loading Modules

```php
$core->hack->(name)->mod( $module );
```

Example for the `listing` module in the `proxy` hack:

```php
$core->hack->proxy->mod( 'listing' );
```

Call a function immediately after loading (function must accept `$core`):

```php
$core->hack->proxy->mod( 'listing', 'hack_proxy_list' );
```

## Use Cases

- **Page handlers** — implement routing for sections and pages
- **Scheduler tasks** — files named `cron-name.php`
- **API applications** — files named `api-name.php`

## Routing

### URL Properties

| Property | Description |
|----------|-------------|
| `$core->app->div` | Current section |
| `$core->app->item` | Selected element |
| `$core->app->id` | Integer from item |
| `$core->app->subdiv` | Subsection |
| `$core->app->subitem` | Nested element |
| `$core->app->subid` | Integer from subitem |
| `$core->app->action` | Action GET parameter |
| `$core->app->message` | Success/error message |

### Routing Example

```php
function hack_proxy_route( $core ) {
	switch ( $core->app->div ) {
		case 'proxy-list':	$core->hack->proxy->mod( 'listing', 'hack_proxy_list' );
		case 'proxy-check':	$core->hack->proxy->mod( 'listing', 'hack_proxy_check' );
		case 'proxy-stats':	$core->hack->proxy->mod( 'stats', 'hack_proxy_stats' );
	}
}
```

Register the routing function during hack initialization:

```php
function hack_proxy_init( $core ) {
	$core->handle( 'modules', 'hack_proxy_route' );
}
```
