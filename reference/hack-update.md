# Automatic Updates

Hacks from the AlterCPA team update automatically. You can implement an update service for your own hacks and deliver fresh versions to all AlterCPA users in under an hour.

## Configuration (update.json)

Edit via the "Update settings" section → "Files" button.

| Field | Description |
|-------|-------------|
| `disable` | Disables automatic updates (useful when you've made local changes) |
| `base` | URL to your update configuration JSON file |
| `db` | Database schema version — increment when changing the schema. Requires a `schema.php`; on update the platform re-runs its schema check when the new `db` is greater than the installed one (see [Schema versioning](hack-db.md#versioning-and-auto-apply)) |
| `platform` | Comma-separated list of products this hack applies to (e.g. `crm`, `crm,lite`). The updater **skips** the hack when `$core->platform` is not in the list |
| `time` | Timestamp of the installed version — the field the updater actually compares against the remote config |

After a successful update the merged remote entry is written back to the local
`update.json`, so `url`/`name`/`time`/`date`/`md5` may appear locally too.

## Running Your Own Update Service

Your server needs:
1. An update configuration JSON file
2. ZIP archives with fresh hack versions

### Update Configuration File Format

An associative array keyed by hack name:

```json
{
  "proxy": {
    "url":      "https://storage.server/hacks/proxy.zip",
    "name":     "proxy",
    "platform": "crm,lite",
    "time":     1668753381,
    "date":     "2022-11-18 09:36:21",
    "db":       1001,
    "md5":      "0f240c960700b925319e5264fa68162d"
  }
}
```

| Field | Description |
|-------|-------------|
| `name` | Hack name (matches the array key) |
| `url` | Link to ZIP archive |
| `platform` | Products the hack applies to; the updater skips installs whose `$core->platform` is not listed |
| `time` | **The version gate.** The updater downloads and reinstalls the hack when the remote `time` is greater than the installed `time` |
| `date` | Human-readable update time (YYYY-MM-DD HH:MM:SS), informational |
| `db` | Database schema version (drives the schema re-check) |
| `md5` | Informational integrity hash — it is **not** used to decide whether to update (the decision is the `time` comparison above) |

> **How a rollout works.** The platform's 3-minute system cron fetches each hack's
> `base` config, compares the remote `time` to the installed one, and (when newer and the
> `platform` matches) downloads the ZIP, replaces the hack folder, and applies the schema.
> Because it runs every few minutes across all installs, a published change reaches every
> active install within roughly an hour. Set `disable: true` in a hack's local
> `update.json` to freeze an install you've modified by hand.
