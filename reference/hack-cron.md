# Task Schedule

## Creating a CRON File

Scheduler files are created in the "Modules" section. The filename must start with `cron-` followed by the task name: `cron-stats.php`.

The main function naming format: `hack_<name>_cron_<task>` — e.g., `hack_proxy_cron_stats`. Accepts one parameter: `$core`.

```php
<?php
function hack_proxy_cron_stats( $core ) {
	// Cron code here
}
```

## Built-in Scheduler Events

| Key | Frequency | Recommended for |
|-----|-----------|----------------|
| `cron-1min` | Every minute | Time-sensitive fast tasks |
| `cron-3min` | Every 3 minutes | Longer non-critical tasks |
| `cron-10min` | Every 10 minutes | Medium-duration tasks |
| `cron-1day` | Daily at midnight | Long operations, stats, DB optimization |

Register in `start.php`:

```php
return [
	'cron-1min' => [ 'visits' ],
	'cron-1day' => [ 'stats', 'emails' ],
];
```

> **Key vs. value — they mean different things.** The **key** `cron-<slot>` selects
> *when* the task runs (which scheduler invocation fires it). The **value** names the
> *task*: each `<task>` maps to the file `mod/cron-<task>.php` and the function
> `hack_<name>_cron_<task>`. So `'cron-1day' => [ 'stats', 'emails' ]` runs, on the daily
> tick, `hack_<name>_cron_stats()` (from `mod/cron-stats.php`) and
> `hack_<name>_cron_emails()`. The value need not match the slot — `'cron-1min' => [ 'visits' ]`
> looks for `mod/cron-visits.php` / `hack_<name>_cron_visits`, *not* `cron-1min`. By
> convention hacks often set them equal (`'cron-1min' => '1min'`). A bare string instead
> of an array is accepted for a single task.

> **Overlap protection.** Before running a slot, `crontab()` writes a PID file under
> `data/work/` and checks whether the previous run of the same slot is still alive
> (via `/proc/<pid>/cmdline`, or `ps`); if so it **exits immediately**. Long tasks
> therefore never overlap themselves — a 1-minute task that takes 5 minutes simply
> skips the intervening ticks. Each run also records its timing and peak RAM to
> `data/work/cron-<slot>.txt`, surfaced on the Settings → Service page.
>
> On every invocation the platform runs its own system tasks and base-module tasks **first**,
> then the registered hack tasks — so your hack cron always runs after the core's
> housekeeping for that slot.

> **Background workers and a root tick.** A worker that a cron task launches in the
> background outlives its tick. After a platform update one cron tick runs as **root**: the
> update process runs it synchronously, and a worker launched from it would be a root process
> that the web user (`wsvr`) could neither stop nor replace. Guard the launch with `isroot()`
> and let the next ordinary tick after the root one start the worker instead.

```php
// $has - the worker is needed and not running yet
if ( $has && ! isroot() ) exec( 'php -f ' . PATH . 'hack/proxy/worker.php >/dev/null 2>/dev/null &' );
```

## Arbitrary Intervals

Use a timer check for custom intervals:

```php
$timer = $core->config( 'hack-proxy-stats' );
if ( $timer > time() ) return true;
$settimer = time() + rand( 1800, 3600 );
$core->reconf( 'hack-proxy-stats', $settimer );
```

Use the hack name with the `hack-` prefix for the config field. Include random variance of several minutes to avoid thundering herd.

> **Count the key: `cpa_config.name` is `varchar(20)`.** `hack-<name>-<what>` reaches the limit
> quickly — `hack-<name>-lastcheck` overruns it for all but the shortest names — and an
> over-length key is not an error you will see: a server without `STRICT_*` in its
> `sql_mode` truncates it silently, so two keys sharing their first 20 characters become one
> row. A strict server errors instead. Never
> build a per-item key by appending an id (`'hack-myhack-seen-' . $id`): the prefix eats the
> budget, the ids collapse onto their surviving digits, and every item then shares one marker.
> Per-item state is a **column on that item's row**. See
> [The Core → Configuration](hack-core.md#configuration).

A timer like this is an **internal state marker**, which is why the block is a bare scalar
rather than an array. The hack's **settings** are never stored this way — they are one
array, see [The Core → Configuration](hack-core.md#configuration).

## Custom Scheduler Events

Create a file in the `tasks/` folder named after your event (e.g., `crawl.php`):

```php
<?php
define( 'PATH', realpath( dirname( __FILE__ ) . '/../' ) . '/' );
require_once PATH . 'core/mods/cron.php';
crontab( 'crawl' );
```

Register with the system crontab as user `wsvr`:

```
*/15 * * * * php -f /var/www/network.domain/tasks/crawl.php > /dev/null 2>&1
```

Initialize functions using the `cron-crawl` key in your configuration.
