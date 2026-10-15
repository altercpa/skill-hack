# Delivery Services

Delivery service connectors enable logistics integration: package sending, tracking, label printing, and CRM form extensions.

> **AlterCPA Pro only.** The delivery subsystem — the `deliverybase` class, the bundled
> connectors and the `delivery` dispatcher — ships **only in AlterCPA Pro**. AlterCPA Lite has
> none of it, so delivery hacks do not apply to Lite.

## Creating a Connector

Connector files live in the `delivery/` folder. File naming: lowercase letters and numbers, starting with a letter — no capitals, hyphens, or underscores.

The class name combines "delivery", the hack name, and the connector name with underscores (e.g., `delivery_dhl`), and must extend `deliverybase`.

## Implementable Methods and Properties

| Method / Property | Description |
|-------------------|-------------|
| `track()` | Package tracking |
| `calc()` | Delivery cost and time calculation |
| `send()` | Send parcel to service |
| `$cansend` | Sending capability flag |
| `$canauto` | Automatic sending support |
| `bulkprint()` | Batch label printing |
| `$canprint` | Label-printing capability flag |
| `options()` | Action button definitions |
| `action()` | Button action handler |
| `block()` | Custom form content |
| `ajax()` | Inline form data handling |
| `_courier( $id )` | Courier name retrieval |
| `_tariff( $id )` | Tariff name retrieval |

The three capability flags are `$cansend`, `$canauto` and `$canprint`. The base class
also provides reusable helpers you call rather than override: `calcs()` (build a price
result), `esc()`, `curl()` (the standard HTTP helper), `autosend()`, `topay()`,
`description()` and `tracksort()`.

Tracking is the recommended minimum implementation.

## Registering in start.php

```php
return [
	'delivery' => [ 'cdek', 'dhl', 'ponyexpress' ],
];
```

## Tracking

The `track()` function accepts a tracking code and order data array. Return structure:

```php
return [
	'next'		=> time() + 43000,	// Next check timestamp
	'status'	=> $status,
	'time'		=> $tm,
	'stage'		=> $stage,
];
```

### Status Values

| Status | Meaning |
|--------|---------|
| `wait` | Awaiting shipment |
| `transfer` | In transit |
| `problem` | Delivery issue |
| `delivered` | Destination reached |
| `paid` | Payment completed |
| `return` | Return or disposal |
| `comment` | Status note |

### Stage Array Fields

Each progress entry can include: `status`, `time`, `country` (ISO code), `zip`, `city`, `comment`, `md5` (duplicate check hash).

## Cost Calculator

```php
public function calc( $o ) {
	$price = $this->_price( $o );
	$term = $this->_term( $o );
	return $this->calcs( $price, $o, $term['min'], $term['max'] );
}
```

Only shipping price is mandatory; delivery timeframes are optional.

## Parcel Sending

Enable automatic sending:

```php
public	$canauto	= true;
public	$cansend	= true;
```

The `send()` function accepts order data and returns `true`/`false`. Must update the order with tracking code:

```php
$core->lead->edit( $o['order_id'], [ 'track' => $code, 'trackon' => 1 ] );
```

## Label Printing

```php
public $canprint = true;

public function bulkprint( $ids ) {
	$files = [];
	foreach ( $ids as $i ) $files[] = $this->_print( $i );
	return $files;
}
```

## Action Buttons

`options()` defines menu items with: `name`, `short`, `icon` (Font Awesome), `confirm`, `url`.

Link construction:
```php
$core->u( [ 'order', $o['order_id'] ], 'action=dlaction&oa=send' )
```

`action( $action, $o )` (singular) processes button clicks and handles redirects with status messages.

## Custom Forms and AJAX

`block()` renders content below product lists using the template engine:

```php
$core->hack->dhl->tpl( 'innerform', 'shpt-dhl' );
$core->tpl->vars( 'innerform', [
	'u_check'	=> $core->u( [ 'order', $o['order_id'] ], 'action=dlajax&oa=check' ),
	'couriers'	=> json_encode( $this->courier, JSON_PRETTY_PRINT ),
] );
```

`ajax()` handles inline requests, returns JSON-serializable array:

```php
public function ajax( $action, $o ) {
	switch ( $action ) {
		case 'cities':
		return $this->cities( $this->core->post['area'] );
	}
}
```

### Metadata Storage

```php
$tm = xxdec( $o['track_meta'] );
$core->lead->edit( $o['order_id'], [ 'tmeta' => $tm ] );
```

## Configuration

```php
public function form() {
	return [
		[ 'type' => 'text', 'name' => 'generic_url', 'value' => $this->config['url'] ],
	];
}

public function save( $data ) {
	return [ 'url' => $this->esc( $data['generic_url'] ) ];
}
```

Field types: `text`, `email`, `textarea`, `number`, `select`, `mselect`, `radio`, `checkbox`.
