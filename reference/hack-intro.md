# Introducing Hacks

**Hacks** represent extension modules designed for AlterCPA, accessible through the "Control - Hacks and modules" section. This area enables users to view available hacks from AlterCPA developers, create your own hack and change any of its settings.

## Installing and Configuring a Hack

Downloaded hacks appear in the "Hacks and modules" section and require installation via the "Install" button, which prepares the database. After installation, users can adjust basic settings and activate the hack using the "Settings" button. Some hacks offer additional customization options once activated.

> **Important:** Activate third-party hacks on test sites rather than live systems, as corrupted hacks could potentially damage networks.

## Creating a New Hack

Users may develop custom hacks or upload third-party versions. New hacks require a folder name containing small Latin characters and numbers with the first character being a letter — no capitals, hyphens, or underscores permitted.

The system automatically generates the starter files for a new hack — `start.php`, `main.php` and `update.json`. Third-party hack installation requires creating an empty folder matching the archive's name, then using the "Import" button to select and upload the hack archive.

> The "no capitals / hyphens / underscores, must start with a letter" rule is a hard requirement in practice: the hack name becomes part of PHP class and function names (`hack_<name>`, `hack_<name>_<lib>`), so anything outside `[a-z0-9]` would produce invalid identifiers.

Users can share created hacks by clicking "Export" and distributing the downloaded file, ensuring recipients know the hack's name.

## Hack Structure

Hacks reside in the `hack` folder and may include:

| File / Directory | Description |
|-----------------|-------------|
| `start.php` | (required) Initialization function |
| `main.php` | Super class for loading resources |
| `lib/` | Libraries (models) |
| `mod/` | Modules (controllers) |
| `lang/` | Language files |
| `tpl/` | Templates (views) |
| `img/`, `css/`, `js/` | Styling files |
| `delivery/` | Delivery service connectors |
| `adv/` | Advertising network connectors |
| `param.json` | Configuration parameters |
| `schema.php` | Database schema |
| `update.json` | Auto-update configuration |

## Deleting Hacks

The interface lacks a direct deletion option. Disable hacks instead, or manually remove the hack folder from the `hack` directory.
