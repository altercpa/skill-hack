# AlterCPA Hacks — a skill for Claude

**Everything Claude needs to write correct hacks for AlterCPA: the platform's interfaces, conventions and code style, in one skill.**

[![License: MIT](https://img.shields.io/github/license/altercpa/skill-hack)](LICENSE)
[![Latest release](https://img.shields.io/github/v/release/altercpa/skill-hack)](https://github.com/altercpa/skill-hack/releases/latest)

**English** · [Русский](README.ru.md)

![Purr the cat in a workshop snaps a new module into a half-assembled machine — cover illustration for the AlterCPA hacks skill](.github/assets/cover.jpg)

## What is it

A **hack** is an extension for [AlterCPA](https://www.altercpa.pro) — Lite, Pro or Core. It adds
control-panel pages, modules, libraries, API applications, database tables, cron tasks, and
advertising or delivery connectors. The platform's own code is not readable, so you extend it through
documented interfaces alone.

This is a [Claude](https://claude.com) **skill** that carries those interfaces — the hack manual, page
by page. With it Claude builds a hack the way the platform expects instead of inventing an API.

## Three rules it enforces

1. **Code style is mandatory** — tabs, braces, naming, the file header: new code is indistinguishable
   from the platform's own.
2. **Settings are one array, never a schema** — a Settings page with twelve inputs is twelve keys of
   one array, not twelve columns.
3. **The hack name is an identifier** — lowercase Latin letters and digits only, because it becomes
   class, function and table names.

## What it covers

| Area | Pages |
|---|---|
| **Basics** | what a hack is, the startup file, the main class and its configuration |
| **The platform object** | events, config, cache, sessions, users and routing, text helpers, site assembly and forms, media, e-mail, global functions |
| **Building** | controllers, libraries, language files, templates and layout patterns, the Settings page, API apps, cron tasks, database |
| **Connectors** | advertising networks, delivery services, update distribution |
| **Style** | the code-style guide |

## Install

### Claude Code

Download [`altercpa-hacks-skill.zip`](https://github.com/altercpa/skill-hack/releases/latest/download/altercpa-hacks-skill.zip)
and unpack it into a skills folder:

```sh
unzip altercpa-hacks-skill.zip -d ~/.claude/skills/        # every project
unzip altercpa-hacks-skill.zip -d .claude/skills/          # this project only
```

On Windows unpack it into `%USERPROFILE%\.claude\skills\`.

Prefer git? Clone straight into the skills folder, and `git pull` is the whole update:

```sh
git clone https://github.com/altercpa/skill-hack.git ~/.claude/skills/altercpa-hacks
```

### Claude.ai and Claude Desktop

In Claude's settings add a custom skill and upload the same zip.

## Try it

Nothing to call by hand — Claude loads the skill on its own whenever a request is about AlterCPA
hacks. For example:

- *Create a new AlterCPA hack that adds a page to the control panel with a settings form.*
- *Add a cron task and an API application to my hack.*
- *Review my hack's code against the platform's code style.*

## Update

Releases are tagged `vYYYY.MM.DD`. Watch the repository (*Watch → Custom → Releases*) to be
notified, then download the new zip — or run `git pull` if you cloned.

## What is inside

| Path | What |
|---|---|
| [`SKILL.md`](SKILL.md) | the skill's entry point: the rules and the index of pages |
| [`reference/`](reference) | the manual, one page per topic |

The skill is based on the hack manual published at
<https://www.altercpa.pro/en/help/section-hack>.

## License

[MIT](LICENSE) — use it, copy it, change it, ship it.

The mascot artwork in [`.github/assets/`](.github/assets) is © AlterCPA and is not covered by the MIT license.
