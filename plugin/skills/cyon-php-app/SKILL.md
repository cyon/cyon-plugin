---
name: cyon-php-app
description: Build or deploy a PHP application on cyon hosting, from plain PHP and Slim to Laravel and Symfony, with one document root and SQLite or MariaDB. Use when a project has server-side logic, logins, a database, queued jobs or scheduled tasks, including a shop that is not WooCommerce. Covers the layout with public/ as document root, where .env and storage live, Composer off the hosting, the database credentials file, cron-driven workers, and what a shop adds.
cyon-facts-version: 2026-10-09
---

# A PHP application on cyon

PHP per request, the way cyon runs it: no long-running server, no supervisor. A Laravel or
Symfony application runs here unchanged once three things are right: the layout, the store and
the workers.

Read `cyon-platform` first, and run `cyon-preflight` over an existing codebase before deploying
it.

## Layout

The application lives in its own directory inside the public tree, and only its public folder
is served:

```
/home/<login>/public_html/
  <app>/              application root: code, vendor/, .env, storage/
    public/           the document root, and nothing above it is reachable
```

- Point the site's document root at `public_html/<app>/public`. A document root has to be inside
  `public_html`: `subdomain_create` and `site_docroot_set` refuse anything outside it.
- A document root deeper than one level does not inherit a PHP version set higher up, so the
  version is set on the document root itself.
- `.env`, `vendor/`, `storage/`, `.git` and any SQLite file stay above `public/`, never inside
  it. `.htaccess` denial is the second line, not the first.

## Choosing the store

**SQLite**, for a new project and for any framework that supports it. It runs inside the PHP
process, so a query costs no round trip to a server; it needs no database to create and no
credentials to manage.
Laravel's defaults since version 11, SQLite with the `database` driver for sessions, cache and
queue, work here as they are.

It has one writer at a time. A write waits for the one before it, up to `busy_timeout`, so a
background job that holds a write transaction stalls the web requests that want to write. A
transaction that reads first and writes later fails as busy when another one wrote in between;
start such a transaction with `BEGIN IMMEDIATE`. Copying the file is no backup while the site
runs, see `cyon-operations`.

**MariaDB** when the software requires it, when the database has to be reachable over the
network, when background jobs write while visitors do, or when someone will work in
phpMyAdmin, which only knows MariaDB.

Say which you chose and why. With SQLite: WAL mode, a `busy_timeout` of 2 to 5 seconds, short
write transactions, the file outside `public/`, and a backup through `VACUUM INTO`, see
`cyon-operations`.

## Database credentials

`database_create` never returns the password. It writes a file into the account, readable only
by the account, and returns its path:

```
~/.cyon-db-<name>-<suffix>.env
  CYON_DB_HOST, CYON_DB_NAME, CYON_DB_USER, CYON_DB_PASSWORD
```

Build the application's `.env` from it on the hosting, without the password ever passing
through the conversation:

```bash
set -a; . ~/.cyon-db-<name>-<suffix>.env; set +a
cat >> <app>/.env <<EOF
DB_CONNECTION=mysql
DB_HOST=$CYON_DB_HOST
DB_DATABASE=$CYON_DB_NAME
DB_USERNAME=$CYON_DB_USER
DB_PASSWORD=$CYON_DB_PASSWORD
EOF
```

Never print the file, and never copy its contents into the journal.

## Deploying

- **Build off the hosting.** `composer install --no-dev -o` and the asset build run locally or
  in CI. Composer on the hosting is a `~/bin` install and memory-hungry under LVE.
- **Code through Git, artefacts through rsync.** Git for what is in the repository, rsync for
  `vendor/` and compiled assets from the local build. Symlinked release directories work too.
- **Declare what persists.** `.env`, `storage/`, uploads and the SQLite file survive every
  deployment and are never tracked by Git.
- **Never empty the application directory** and never run `git clean -fdx` there: it deletes
  exactly the files that are not in the repository, which are the ones that matter.
- `php artisan storage:link` fails when `symlink()` is disabled; create the link with `ln -s`
  in the shell.

## Scheduled jobs and workers

A full crontab with minute granularity, and no supervisor, so every worker is short-lived,
runs once at a time, and exits on its own:

```cron
* * * * * /opt/alt/php83/usr/bin/php /home/<login>/<app>/artisan schedule:run >> /home/<login>/<app>/storage/logs/cron.log 2>&1
```

- The crontab belongs to the whole account and every site on it shares it. Change it the way
  `cyon-operations` describes under Cron hygiene, never by piping a filtered copy back.
- Always the absolute versioned PHP path. Plain `php` in cron runs the default version, which
  is the classic "works in the browser, fails in cron".
- Export `PHP_INI_SCAN_DIR` when the job depends on custom ini settings.
- **Laravel queues:** schedule `queue:work --stop-when-empty` with `withoutOverlapping()`
  rather than running `queue:work` as a daemon, which LVE kills. The database driver works on
  SQLite and on MariaDB, where it uses `SKIP LOCKED`.
- **Symfony Messenger:** `messenger:consume --time-limit=55` from cron every minute.
- APCu is not shared between cron and the web, so nothing a job writes to APCu reaches a
  request.

## What a shop adds

A shop on a PHP framework, rather than WooCommerce, which is `cyon-cms`. It is a PHP
application where four things go wrong expensively:

- **Order mail against the limit.** Confirmation mail through local Exim is right, but script
  mail is capped at 100 per hour, 30 in a new hosting's first 30 days. A sale or a newsletter
  exceeds that silently. Raise the limit in my.cyon before launch, and send anything bulk
  through an email service provider.
- **Payment webhooks.** The provider retries, so the handler must be idempotent, verify the
  signature, answer quickly, and leave the slow work to a queued job.
- **The database follows the shop software.** Shopware, Sylius and Magento need MariaDB. A
  hand-built shop can use SQLite like any application; keep checkout and webhook transactions
  short, so they do not wait on each other.
- **Backups while orders come in.** A dump with `--single-transaction` on MariaDB, `VACUUM
  INTO` on SQLite, never a copy of files that are being written.

## Third-party code

Code taken from elsewhere, a library, a snippet, a font or an icon set, keeps its licence notice
when it is published. Leave the notice in the file or add it where the licence asks, and name
the third-party parts and their licences in the answer, so the customer knows what they are
publishing.

## With the MCP server connected

1. `domain_list` and `site_list`, or `subdomain_create` for a new project, or `domain_add` when the
   customer brings a domain they already registered.
2. `php_version_set` for the document root, after checking the versions in `hosting_get`.
3. `database_create` if the project uses MariaDB.
4. `deploy_prepare` with `type: dynamic`, the application root, the document root,
   `persistent_paths`, `git: true`, and `enable_php_error_logging`, which covers the whole
   hosting; say so when asking.
5. `deploy_key_add`, then the commands `deploy_prepare` returned.
6. `https_enable`, `certificate_get` until active, `https_enable` again for the redirect.

Record the layout, the persistent paths and the cron entries in `.cyon/journal.md`.
