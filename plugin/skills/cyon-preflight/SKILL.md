---
name: cyon-preflight
description: Audit an existing codebase against what cyon hosting can run, before deploying it or quoting a migration. Use when asked whether a project will run on cyon, why a deployment failed, or to check a repository before moving it to cyon hosting. Reports blockers that cannot be fixed on the platform and what replaces each when moving a project here, provisioning steps that need a person in my.cyon, and adjustments the code needs. Also the test oracle for anything the cyon archetype skills generate.
cyon-facts-version: 2026-09-23
---

# cyon preflight

Audits code that already exists. The question is not "is this good code" but "will this serve
on cyon, and what has to happen first".

Read `cyon-platform` for the fact base. This skill is how you apply it to a repository.

## How to report

Three buckets, in this order, because they need different people:

| Bucket | Meaning | Who acts |
| --- | --- | --- |
| **Blocker** | Cannot run on cyon at all | The customer decides: change the design, Managed Server, external service, or another host |
| **Provisioning** | Runs, once someone enables something in my.cyon | A person, once, before deployment |
| **Adjustment** | Runs, once the code changes | Whoever deploys |

Never soften a blocker into an adjustment. If an application needs a Node server, no amount of
configuration fixes that, and saying so early is the whole value of this audit.

## Blockers, with what to look for

| Look for | Why it blocks |
| --- | --- |
| `next.config.*` without `output: 'export'`, `nuxt.config.*` without `ssr: false`, `@sveltejs/adapter-node` or `adapter-auto`, `remix.config.*` | Server-side rendering needs a Node HTTP runtime, which does not exist |
| `server.js`, `app.listen(`, `"start": "node ..."` as the serve command | Same |
| `predis/predis`, `ext-redis`, `REDIS_HOST`, `CACHE_DRIVER=redis`, `SESSION_DRIVER=redis`, `memcached` | No Redis, no Memcached, no external object cache |
| `Dockerfile`, `docker-compose.yml` as the deployment target, `supervisord.conf`, a systemd unit | No containers, no process supervisor, no always-on daemon |
| `laravel/octane`, `swoole`, `roadrunner`, FrankenPHP | All need a runtime that is not here |
| `DB_CONNECTION=pgsql`, `mongodb`, `elasticsearch`, `meilisearch`, `typesense` pointing at localhost | MariaDB is the only engine; those services do not run locally |
| A WebSocket server, `laravel/reverb`, `socket.io` server side | Managed Server only, on request. On anything else it is a blocker |
| `queue:work` as a long-running process, a daemon worker | LVE kills it. A cron-driven short-lived worker is the replacement |

Docker used only as a local development environment is not a blocker. Read what actually
serves production before flagging it.

## What replaces a blocker

When a project moves here from Vercel, Netlify, Heroku or a VPS, most blockers have a
replacement on cyon. Name it next to the finding, and say plainly when there is none.

| Blocker | Replacement on cyon | When there is none |
| --- | --- | --- |
| Server-side rendering | A static export, when the pages need no per-request data; per-request data from a small PHP endpoint under `/api/`. See `cyon-static-site` | Pages rendered per visitor on the server: another host |
| A Node or other API server | The same endpoints in PHP, which is a rewrite, not a port | Too large to rewrite: an external host for the API |
| Redis or Memcached as cache | A file cache; APCu within the web tier only, since cron does not share it | None needed |
| Redis for sessions | `file` or a database table | None needed |
| A queue worker daemon | A database queue on MariaDB, worked off by cron every minute with an overlap guard | Sub-second latency needs: a hosted queue |
| Supervisor, systemd, a Docker service | A short-lived cron job per task | A process that must never stop: another host |
| Octane, Swoole, RoadRunner, FrankenPHP | Plain per-request PHP. Most code runs unchanged once the package is removed | None |
| PostgreSQL | SQLite or MariaDB, whichever the software supports: the ORM usually carries it, raw SQL and Postgres types need review | A hard Postgres dependency: an external database |
| Elasticsearch, Meilisearch, Typesense | SQLite FTS5, or MariaDB `FULLTEXT` | Typo-tolerant or faceted search at scale: a hosted search service |
| WebSockets, long-lived SSE | Polling on Webhosting and Webhosting Pro | Real-time as a requirement: a Managed Server on request, or a hosted service |

Two things always move with the project and are not blockers:

- **Build steps** that ran on the old platform run locally or in CI now. Ship the artefacts.
- **Environment variables** from the old platform's dashboard become a `.env` beside the
  document root, and database credentials come from the file `database_create` writes.

The domain's DNS moves last and is the customer's step. A certificate is issued only once the
domain points here, which `certificate_get` shows.

## Provisioning findings

| Look for | What has to be enabled |
| --- | --- |
| `proc_open`, `shell_exec`, `symfony/process` | The function, per function, and the account `php.ini` |
| `exec(`, `/usr/bin/convert` | `exec()`, enabled in the account `php.ini`. The `imagick` extension and GD need neither |
| A `public/` or `web/` directory as the serve root | The document root, set per domain |
| A PHP version the code requires | The version for that directory, and pinned in `.htaccess` if the document root is deeper than one level |
| Mail in a loop, a newsletter, more than 100 recipients an hour | A raised mail limit, or an email service provider |

## Adjustments

- **Cron entries calling `php`** need the absolute versioned path, and
  `PHP_INI_SCAN_DIR` exported if the job depends on custom ini settings.
- **`storage:link`** fails when `symlink()` is disabled; use `ln -s`.
- **Anything cached in APCu and read from cron**, or the reverse, is invisible across that
  boundary. Move it to a file or the database.
- **`.env`, `vendor/`, `storage/`, `.git`, SQLite files inside the document root** have to move
  beside it, with `.htaccess` denial as a second line.
- **A build step in the deployment** moves off-host. Ship the artefacts.
- **SQLite without WAL and a `busy_timeout`**, or backed up by copying the file, needs both
  fixed and a backup through `VACUUM INTO` scheduled, since the hosting has no `sqlite3` tool.

## Working through a repository

Read these first, they answer most of it:

1. `composer.json` and `package.json`, the `require` and `scripts` blocks.
2. The framework config that decides how it serves: `next.config`, `nuxt.config`,
   `svelte.config`, `vite.config`, `remix.config`.
3. `.env.example` for the drivers: cache, session, queue, database, mail.
4. `Dockerfile`, `docker-compose.yml`, CI workflow, to see what production is meant to be.
5. `crontab`, `supervisord.conf`, any `Procfile`.

Then the code, for `exec(`, `proc_open(`, `Imagick`, `Redis`, `->mail(` in a loop.

## As a test oracle

Run this over anything a cyon archetype skill generated. A blocker finding is a bug in the
archetype, not in the generated project. That makes the archetypes self-checking.

## With the MCP server connected

`hosting_get` reports the PHP versions and quotas this hosting actually offers, and `site_list`
its document roots, so a provisioning finding can be checked rather than assumed. Some of the
provisioning findings are also tool calls now: the document root and the PHP version are
settable, and PHP error logging can be switched on when preparing a deployment. What still
needs a person in my.cyon is in `cyon-provisioning-checklist`.
