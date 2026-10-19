---
name: cyon-platform
description: What cyon hosting actually runs, and whether a planned stack can run on it. Load this BEFORE choosing a framework, a data store, a cache or a deployment method for a site hosted at cyon, and before answering "can I build X on cyon". Covers the PHP, LiteSpeed, MariaDB and CloudLinux runtime, what is not available at any price (Node as a web runtime, SSR, Redis, Docker, always-on workers), the mail and process limits people underestimate, and how to say no to a requirement that does not fit. Every other cyon skill depends on it.
cyon-facts-version: 2026-09-29
---

# The cyon platform

cyon is a LiteSpeed/PHP/MariaDB platform on CloudLinux. Coding agents are trained on a
different stack, so the failure mode here is confident and expensive: an application that
cannot run, discovered at deployment.

Read the fit check first. If the requirement does not fit, say so before designing anything.

The runtime names below are for your decisions. Speak to the customer in the words of cyon.ch:
a plan has Rechenleistung in percent (a Triple has 150%), RAM in GB and SSD-Webspace, and LVE,
CloudLinux and software versions come up only when the customer asks something technical.

## Fit check, before any architecture

Run the requirement against the "no" list below. If it genuinely needs something absent,
name the options rather than engineering around it:

- change the requirement,
- move to a Managed Server, where WebSockets are available on request,
- use a hosted third-party service for that one piece,
- use a different host for the whole thing.

An honest "not here, and here is why" is worth more to the customer than a workaround that
fails in month three. Do not architect around a missing runtime.

## One capability model, two resource scales

Webhosting, Webhosting Pro and Managed Server are the same platform with the same
capabilities. A larger product gives more of the same resources, not new abilities. The one
genuine difference is WebSockets, available on a Managed Server on request.

So there is one architecture to design for, and one escape hatch. "You need a bigger product
to do X" is almost always false here. The honest phrasing is "you need a bigger product to do
X more". Per-plan numbers drift, so ask which plan the hosting is on rather than assuming.

## Runtime

| Component | Value |
| --- | --- |
| OS | CloudLinux, with LVE resource isolation per account |
| Web server | LiteSpeed, Apache-compatible: `.htaccess` and `mod_rewrite` work |
| PHP | Several versions, selectable per directory. Server default 8.3 |
| Database | MariaDB, 10.6 when this was written. No other engine, no version choice; `database_list` reports the version live |
| MTA | Exim |
| Filesystem | Local, with reliable POSIX locking |
| Node and npm | Installed as a CLI only, not wired to the web server |
| ImageMagick | The `imagick` PHP extension is loaded by default. Only shelling out to `/usr/bin/convert` needs `exec()` |
| Git | Preinstalled. Composer is a `~/bin` install and is memory-hungry under LVE |

## What is not available, on any product

- Node, Deno or Bun as an HTTP runtime, and therefore server-side rendering of any kind:
  Next, Nuxt, Remix and SvelteKit in server mode do not serve. Client-side Inertia is fine,
  Inertia SSR is not.
- Redis, Memcached or any external object cache.
- Elasticsearch, Meilisearch, Typesense as local services.
- Supervisor, systemd, Docker, or any always-on daemon.
- Laravel Octane, Swoole, RoadRunner, FrankenPHP.
- PostgreSQL, MongoDB.
- Python, Ruby and Java, and the tools that need them, among them FFmpeg, Gifsicle, PDFtk and
  unoconv.
- The hosting as bulk file storage unrelated to the site, which the terms forbid.

Conditional: WebSockets and long-lived SSE, Managed Server only, enabled on request.

## Document root and multiple sites

The home is `/home/<login>/` and the public tree `/home/<login>/public_html/`. The document
root is configurable per domain and may point at a subdirectory of `public_html`, which is what
makes a framework `public/` or `web/` layout work. A document root outside `public_html` is
refused.

Two things bite here:

**LiteSpeed inherits `.htaccess` one level down only.** A PHP version set on `public_html`
does not reach `public_html/site/public`. Any document root deeper than one level pins its own
PHP version in its own `.htaccess`.

**Everything sensitive lives beside the document root, never inside it.** `.env`, `vendor/`,
`storage/`, content files, SQLite databases and `.git` all sit outside, with `.htaccess` denial
as a second line of defence.

Several sites on one hosting is normal: a directory per site, its own document root and PHP
version each. Sites that share a document root share its `.htaccess` and PHP configuration,
which is a trap rather than a feature.

## PHP configuration

The CLI default version is set separately from the web version, which is the documented cause
of "works in the browser, fails in cron". Custom ini settings are read from one place only,
`~/etc/php_settings/conf.d/00-mycyon.ini`, the account's own ini, which the PHP settings page in
my.cyon writes; files anywhere else are ignored.

An account ini set up in my.cyon disables `exec`, `shell_exec`, `system`, `dl`, `passthru`,
`proc_open` and `proc_close`, and leaves `mail` allowed. Without one, the server's own defaults
apply, and they are not the same on every server. So read the site's actual value rather than
assuming, see below. `symfony/process` needs `proc_open` and `proc_close`, and ImageMagick
shell-outs need `exec`. Enabling any of them widens the attack surface for everything else on
the hosting, so treat it as a decision, not a default.

The command line reports its own configuration, so `php -i` in the shell says nothing reliable
about the site. To read the site's values, put a file with a random name in its document root
that prints only what you need, request it once, and delete it:

```php
<?php echo ini_get('opcache.enable'), "\n", ini_get('disable_functions'), "\n";
```

A client without a shell calls `artifact_publish` with `phpinfo: true` instead. It writes a
`phpinfo.php` into the document root that answers for fifteen minutes and then deletes itself.

## Caching

**LiteSpeed Cache is the main performance lever**, a full-page cache for anonymous traffic,
available on every product.

**OPcache** is on in an account ini set up in my.cyon, which is its default there, but not on
every server without one. Check it on the site, and if it is off, switching it on in my.cyon is
the cheapest performance gain there is. APCu is loaded.

APCu is not a shared application cache: web processes of one pool share it, the CLI does not,
so anything a cron job writes is invisible to the web tier. Anything crossing that boundary
goes in a file or the database.

## Data stores

**SQLite is the first choice** for a new project and for any software that supports it. The
filesystem is local, so WAL's shared-memory mapping and POSIX locking behave, and a database
inside the PHP process answers a query without a round trip to a server. It needs no server and
no credentials, and its own test suite covers every branch of its code.

Two things it does not have decide when MariaDB is the better store:

- **No panel.** phpMyAdmin, the my.cyon database tooling and support's database restore work
  with MariaDB only. The assistant queries an SQLite file over SSH instead, through PHP's PDO,
  because the `sqlite3` command line tool is not on every server and old where it is (3.26
  on s141).
- **No network access.** Only processes on the hosting open the file. Anything connecting from
  elsewhere, a second server or a reporting tool, needs MariaDB.

So **MariaDB** when the software requires it, as WordPress, WooCommerce and most shop systems
do, when the database has to be reachable over the network, or when someone will work in
phpMyAdmin. It supports `SELECT ... FOR UPDATE SKIP LOCKED`, which Laravel detects for its
database queue.

Whenever SQLite is used: WAL mode, a `busy_timeout` of 2 to 5 seconds, write transactions kept
short because writers take turns, the file outside the document root, and a scheduled backup
through `VACUUM INTO` or the backup API. A file copy taken mid-write is a corrupt snapshot.

State which store you chose and why. Do not default silently.

## Scheduled and background work

A full crontab with minute granularity is available. Two rules:

- Cron runs the default PHP version unless given an absolute path, so always emit
  `/opt/alt/phpXY/usr/bin/php`.
- Custom ini settings in cron need `PHP_INI_SCAN_DIR` exported.

There is no supervisor, so a worker is short-lived, single-instance and overlap-guarded. A
long-running process is killed by LVE.

## Mail, the limit people underestimate

- 100 mails per hour per mailbox, and a separate 100 per hour for script-generated mail,
  recalculated every minute.
- Raisable to 1000 per hour, up to 4000 depending on product.
- 50 recipients per message.
- A new hosting is capped at 30 per hour for its first 30 days.
- The sender address must belong to a domain on the same hosting, or Exim refuses it.
- Outbound port 25 is blocked. Use 465 or 587.

The consequence is the same in every project: transactional mail through local Exim, bulk and
marketing mail through an email service provider. Never build list management on the hosting.

## Images

GD and the `imagick` extension are both loaded by default, with no ini change and no `exec()`.
Default to GD for resize, crop and WebP work, and use `Imagick` for what GD cannot do, colour
profiles, vector rasterisation, advanced compositing. Only calling `/usr/bin/convert` from PHP
needs `exec()`, which is a provisioning step.

Either way both are CPU- and RAM-heavy under LVE: generate derivatives once, cache them on
disk, never transform per request.

## Deployment, the parts that constrain architecture

SSH is on every product, Git is preinstalled, and four facts shape the design:

- **Build steps run off-host.** Vite, SvelteKit, `composer install --no-dev -o`: all in CI or
  locally. Ship artefacts. Building on a shared LVE is slow and fragile.
- **A document root change is a provisioning action**, not something a deploy script does.
- **Symlinked release directories work**, because symlink following is governed by
  `SymLinksIfOwnerMatch` and everything on a hosting shares one owner. `rsync` in place is the
  fallback.
- **`php artisan storage:link` fails when `symlink()` is disabled.** Use `ln -s` in the shell.

## TLS

Let's Encrypt certificates are free. The limits are 50 domains per certificate and 50
certificates per document root. HSTS is supported.

Sending visitors from HTTP to HTTPS is a setting on the site, not a rewrite in `.htaccess`.
Do not write one: the setting survives a deployment and cannot collide with an application's
own rules. It only works once the certificate is live, and takes a few minutes to apply.

## When the cyon MCP server is connected

The tools answer what is true of *this* hosting and only knowable live: which PHP versions it
offers, its quotas, its sites and their document roots, its databases. Use them instead of
asking the customer, and use `hosting_list` and `domain_list` to resolve a hosting rather than
guessing. This skill stays the source for what the platform can and cannot do at all.

A client without a shell, such as Claude on the web or ChatGPT, publishes with
`artifact_publish`: text files from the conversation, written into the site's document root.
It takes HTML, CSS, JavaScript, SVG, JSON, text, XML and web manifests, and refuses PHP,
dotfiles such as `.htaccess` and binary files such as images. Anything beyond a static site
needs SSH, so it needs a client with a shell or the my.cyon file manager.
