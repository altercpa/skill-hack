# Advertising Networks

Advertising network connectors load traffic expense data to calculate ROI statistics. The connector implements a `fetch` function that returns an array with spends for the period.

## Returned Data Fields

| Field | Description |
|-------|-------------|
| `cpc` | Price per click |
| `cost` | Total price for clicks |
| `curr` | ISO currency code (the key shipped connectors emit; `currency` is also accepted) |
| `from` / `to` | Period dates as UNIX timestamps |
| `flow` | Flow ID (matching key in **Lite**) |
| `utms`, `utmc`, `utmn`, `utmt`, `utmm` | UTM parameters |
| `extu`, `exts` | Agency identifier fields (**AlterCPA Pro only**) |

The column the spend is matched against (`field()`) differs between products: Pro keys on
`utmn/utmt/utmm/utmc/utms/extu/exts/utmd/utmu/subid`, while Lite keys on
`utmn/utmt/utmm/utmc/utms/utmd/subid/flow/way/utmu`. So `extu`/`exts` matching is Pro-only
and `flow`/`way` is Lite-only — verify against your target product's connector base.

## Creating a Connector

Connector files live in the `adv/` folder. File naming: lowercase letters and numbers, starting with a letter.

The connector class combines "adv" with the connector name (e.g., `adv_facebook`) and inherits from `advbase`.

Required elements:
- `$auths` — authorization data array
- `fetch()` — data loading function

Language file keys: `advrt_<name>` for display name, `advrt_<name>_d` for description.

> **Where `advbase` lives.** The base class `advbase` exists in **both** products, but in
> different libraries: AlterCPA **Pro** ships it alongside class `adv`, AlterCPA **Lite**
> alongside class `spend`.
> The ad-network subsystem — the bundled connectors and the `fetch()` pipeline — is fully
> usable in Lite; only the host class differs. The hack `adv`
> start.php key is consumed identically in both.

## Authorization Parameters

`$auths` can include: `url`, `user`, `login`, `pass`, `api`, `key`.

## Class Methods

| Method | Description |
|--------|-------------|
| `$this->config[$name]` | Configuration array |
| `$this->from()` | Period start timestamp |
| `$this->to()` | Period end timestamp |
| `$this->field()` | Campaign ID field |
| `$this->curl()` | HTTP request |
| `$this->json()` | JSON HTTP request |

## Additional Configuration (form/save)

Implement `form()` and `save()` for extra settings fields.

Supported field types: `text`, `email`, `textarea`, `number`, `select`, `mselect`, `radio`, `checkbox`.

The `save()` function processes and returns configuration data.

## Registering in start.php

```php
return [
	'adv' => [ 'facebook', 'google' ],
];
```
