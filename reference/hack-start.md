# Startup File

The hack initialization file should return an array containing the hack configuration with launch functions, delivery and ad network handlers, scheduler tasks, and API functions.

## Configuration Keys

| Key | Description |
|-----|-------------|
| `init` | Hack initialization function |
| `api` | List of API application files |
| `api-auth` | API functions authorization (array only) |
| `cron-1min` | Scheduler: every minute (quick tasks) |
| `cron-3min` | Scheduler: every 3 minutes (high duration tasks) |
| `cron-10min` | Scheduler: every 10 minutes (medium duration tasks) |
| `cron-1day` | Scheduler: daily (very long tasks) |
| `tpl` | Overriding system templates |
| `lib` | Overriding system libraries |
| `delivery` | Handlers for delivery services |
| `adv` | Handlers for ad networks |

**Return array example:**

```php
return [
	'init' => [ 'hack_proxy_init' ],
	'api' => [ 'proxy', 'wm' ],
	'api-auth' => [
		[ 'proxy', [ 'check', 'list', 'add', 'del' ] ],
		[ 'wm', 'getproxy' ],
	],
	'cron-3min' => [ 'checker' ],
];
```

> Every value may be a **single string** or an **array** — the loader wraps a bare
> string into a one-element list. Both `'init' => 'hack_proxy_init'` and
> `'init' => [ 'hack_proxy_init' ]` are valid; real hacks use both forms. The keys are
> read generically, so any `cron-<slot>` key works,
> not just the four built-in slots (see [Task schedule](hack-cron.md)).

## Initialization Function

The system executes initialization functions immediately after loading all modules. These functions prepare handlers used in the system — complex actions should be avoided during initialization.

```php
function hack_proxy_init( $core ) {
	$core->handle( 'modules', 'hack_proxy_route' );
	$core->handle( 'menu', 'hack_proxy_menu' );
}
```

## Handlers and Filters

Handlers are added using the `handle` function:

```php
$core->handle( $place, $function );
```

The platform fires each place either as an **action** (`process`) or a **filter**
(`filter`). An action callback receives `( $core )` (or `( $core, $info )`) and its
return value is only used as a boolean; a **filter** callback receives
`( $core, $data )`, threads `$data`, and **must `return` the (possibly modified) data**
— `menu`/`newmenu`/`medialibs` are filters, so a handler that forgets to `return $data`
wipes the menu/library list. See [Events](hack-core.md#events-handlers-and-filters).

| Handler | Description |
|---------|-------------|
| `modules` | Routing functions executed before system routing |
| `actions` | Preparatory actions executed before module handlers |
| `menu` | Prepares navigation menu |
| `newmenu` | Menu post-processing filter |
| `header` | Executed before displaying top block |
| `footer` | Executed before displaying footer |
| `medialibs` | Filters library list for loading files |

## Full Example

```php
<?php

function hack_proxy_init( $core ) {
	$core->handle( 'modules', 'hack_proxy_route' );
}

function hack_proxy_route( $core ) {
	switch ( $core->app->div ) {
		case 'proxy-list':	$core->hack->proxy->mod( 'listing', 'hack_proxy_list' );
		case 'proxy-check':	$core->hack->proxy->mod( 'listing', 'hack_proxy_check' );
		case 'proxy-stats':	$core->hack->proxy->mod( 'stats', 'hack_proxy_stats' );
	}
}

return [
	'init' => [ 'hack_proxy_init' ],
	'api' => [ 'proxy' ],
	'api-auth' => [
		[ 'proxy', [ 'check', 'list', 'add', 'del' ] ],
	],
	'cron-3min' => [ 'checker' ],
	'cron-1day' => [ 'stats' ],
];
```
