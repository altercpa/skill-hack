# Libraries

Libraries are the data access layer — they connect modules to databases or external services. Within MVC, they function as "models."

## Creating a Library

Library files live in the `lib/` folder. File naming: lowercase Latin letters and numbers, starting with a letter — no capitals, hyphens, or underscores.

The class name combines "hack", the hack name, and library name with underscores:

```php
class hack_proxy_items {
	private $core;
	private $hack;

	public function __construct( $core, $hack = false ) {
		$this->core = $core;
		$this->hack = $hack;
		// Other initialisation code
	}
}
```

The loader instantiates the class as `new hack_proxy_items( $core, $this )` — the **first**
argument is the Core god-object (required for system operations), and the **second** is
the hack's own main object (`$core->hack->proxy`). Accept the second parameter when the
library needs to reach back into its hack's configuration or sibling libraries; ignore it
otherwise.

## Connecting and Using Libraries

Libraries load and initialize automatically on first access:

```php
$core->hack->(name)->(library)
```

Example — accessing the `items` library from the `proxy` hack:

```php
$core->hack->proxy->items->add( $item );
$core->hack->proxy->items->mode = 10;
```

## Overriding System Libraries

Extending system libraries is possible but discouraged. The file name must match the original, and the class must inherit from the source:

```php
class hack_address extends address {
	public function check( $geo, $i, $a, $c, $s, $h ) {
		if ( $geo == 'kz' ) return $this->core->hack->cisaddr->kz->check( $i, $a, $c, $s, $h );
		if ( $geo == 'kg' ) return $this->core->hack->cisaddr->kg->check( $i, $a, $c, $s, $h );
		return parent::check( $geo, $i, $a, $c, $s, $h );
	}
}
```

Register overridden libraries in `start.php` using the `lib` key:

```php
return [
	'lib' => [ 'address' ],
];
```
