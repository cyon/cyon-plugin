---
name: cyon-cms
description: Install, deploy or run WordPress, WooCommerce or a flat-file CMS such as Kirby on cyon hosting. Use when content is edited in the browser, when plugins or themes are installed through an admin, or when uploads must survive a deployment. Covers which paths are persistent and who owns them, updating core, plugins and themes with a backup first, LiteSpeed Cache, replacing wp-cron with a real cron, and what a WooCommerce shop needs on top.
cyon-facts-version: 2026-10-14
---

# A CMS on cyon

A CMS differs from every other project in one way: people change the site while it runs. An
editor uploads a picture, an administrator installs a plugin, and none of it is in any
repository. A deployment that does not know that deletes it.

Read `cyon-platform` first. A CMS is a PHP application underneath, so `cyon-php-app` applies
for layout, cron and credentials; this skill is what a CMS adds.

## Decide who owns each path

Every path has one owner, either the deployment or the running site. Mixing them is how a
deployment wipes a plugin someone installed on Friday.

**WordPress**

| Path | Owner |
| --- | --- |
| `wp-content/uploads/` | The site, always persistent |
| `wp-config.php` | The site: it holds the database credentials and salts |
| `wp-content/plugins/`, `wp-content/themes/` | Either. Pick one |

If plugins and themes come from a repository, the admin must not install them, so set
`define('DISALLOW_FILE_MODS', true);` in `wp-config.php`. If the admin installs them, they are
persistent paths and never deployed over. Say which way this site works, and write it into the
journal.

**Kirby**

| Path | Owner |
| --- | --- |
| `content/` | The site, when editors use the Panel. The deployment, only when all content comes from Git |
| `site/accounts/`, `site/sessions/` | The site |
| `media/` | Neither: Kirby regenerates it, so it can be deleted but never needs deploying |

The same rule holds for any flat-file CMS: content that editors touch is state, even though
it is a file.

## Installing WordPress

- A MariaDB database from `database_create`, whose credentials file fills `wp-config.php` on
  the hosting without the password passing through the conversation. See `cyon-php-app`.
- WP-CLI is not preinstalled. It installs into `~/bin` the way Composer does.
- Offer a fresh subdomain for a new site, and never install into a directory that already
  serves something.

## Replace wp-cron with a real cron

WordPress runs its scheduled tasks when someone visits, so a quiet site misses them and a busy
one runs them on visitor requests. Switch that off and run it from cron:

```php
define('DISABLE_WP_CRON', true);
```

```cron
*/5 * * * * /opt/alt/php83/usr/bin/php /home/<login>/<site>/wp-cron.php >/dev/null 2>&1
```

The absolute PHP path matters: plain `php` in cron runs the default version, not the site's.
The crontab is shared by every site on the account, so add the line the way `cyon-operations`
describes under Cron hygiene.
For WooCommerce this is not optional, because its Action Scheduler processes orders through
these tasks.

## Caching

**LiteSpeed Cache is the main performance lever**, a full-page cache for anonymous visitors,
available on every product. For WordPress that is the LiteSpeed Cache plugin. Other CMSs bring
their own page cache; use that before anything else. Check that OPcache is on, which it is not
on every server without the account's own ini; see `cyon-platform`.

Logged-in editors and shop carts are not cached, which is correct. A site that is slow only
for them is a PHP problem, not a cache problem; see `cyon-operations`.

## Updates

Core, plugin and theme updates change the files and often the database of the running site.
cyon takes no snapshot before an update, and its daily backup guarantees no restore point, so
the backup is the customer's, and you take it for them over SSH before the first change. Tell
the customer that before you start, and afterwards tell them where the backup is.

Updating needs a shell. A client without one cannot take the backup, so the customer updates
in the WordPress admin after backing up the site themselves.

Never update during a deployment, or the two overwrite each other.

**1. WP-CLI with the site's PHP.** `deploy_key_add` for the session; the key lasts fifteen
minutes, so a long run needs a fresh one. WP-CLI goes into `~/bin` and runs with the PHP
version `site_get` reports for the site (`php83` below), because plain `php` is the account's
default. `DOCROOT` is the document root from `site_get`:

```sh
mkdir -p ~/bin ~/backups
curl -sSfo ~/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
DOCROOT=<document root>
WP="/opt/alt/php83/usr/bin/php $HOME/bin/wp --path=$DOCROOT"
```

**2. The backup, outside the document root.** A database dump and the site's files without
`wp-content/uploads/`, which an update does not touch:

```sh
STAMP=$(date +%F-%H%M)
$WP db export ~/backups/<site>-$STAMP.sql --single-transaction
tar -czf ~/backups/<site>-$STAMP.tar.gz --exclude=./wp-content/uploads -C "$DOCROOT" .
```

**3. What is pending.** Show the list to the customer, with the version each component has now,
and ask before updating:

```sh
$WP core check-update
$WP plugin list --update=available
$WP theme list --update=available
```

**4. One component at a time:** plugins, then themes, then core, and check the site after each.
A PHP fatal error shows visitors WordPress's critical error page with status 500:

```sh
$WP plugin update <plugin>
curl -s -o /dev/null -w '%{http_code}\n' https://<site>/
curl -s -o /dev/null -w '%{http_code}\n' https://<site>/wp-login.php
```

Core is `$WP core update` followed by `$WP core update-db`. With LiteSpeed Cache active, finish
with `$WP litespeed-purge all`, or visitors keep seeing pages from before.

**5. When a step breaks the site**, put that one component back to the version from step 3:
`$WP plugin install <plugin> --version=<old> --force`. That works for plugins from
wordpress.org only. A premium plugin, or a site that is still broken, goes back to the backup:
`$WP db import ~/backups/<site>-<stamp>.sql` and
`tar -xzf ~/backups/<site>-<stamp>.tar.gz -C "$DOCROOT"`. Importing the dump discards
everything written since, on a shop the orders and customer accounts too, so ask the customer
before importing it and prefer putting back the one component.

Write the versions before and after and the backup paths into `.cyon/journal.md`. The backups
count against the hosting's storage, and deleting old ones is up to the customer.

cyon runs Patchman on every server, which patches known vulnerabilities in customer
applications. It is a net under the site, not a replacement for updating it.

## What WooCommerce adds

WooCommerce is WordPress, with the concerns of a shop from `cyon-php-app`:

- **Order mail against the limit.** Script mail is capped at 100 per hour, 30 in a new
  hosting's first 30 days. Raise the limit in my.cyon before launch, and send newsletters
  through an email service provider, never through the shop.
- **A real mailbox for the sender.** Order mail goes out from an address on a domain of the
  hosting, and customer replies and bounces need a mailbox there to land in. WordPress reports
  every mail as sent either way. The MCP server cannot create a mailbox: the customer creates it
  in my.cyon under E-Mail before launch.
- **Payment webhooks** come through WooCommerce's payment plugins. The real cron above is what
  keeps their follow-up work moving.
- **Backups under writes**: a dump with `--single-transaction`, never a file copy of a database
  that takes orders.

## With the MCP server connected

1. `subdomain_create` for a new site, `domain_add` for a domain the customer already registered,
   or `site_list` and `site_get` to find an existing one and see whether its directory is empty.
2. `php_version_set` and `database_create`.
3. `deploy_prepare` with `type: dynamic` and the persistent paths decided above. Before an update
   on a live site, back up its database and uploads over SSH, as `cyon-operations` describes:
   cyon takes no snapshot of a deployment, and its daily backup guarantees no restore point.
4. `deploy_key_add` for the upload, `https_enable` and `certificate_get` for the certificate.
   Until `certificate_get` reports it active, HTTPS is pending, not done; with
   `dns_points_here` false it stays pending until the name resolves here.

The rsync command `deploy_prepare` returns for a dynamic deployment does not delete, and for
WordPress it must stay that way: WordPress writes `.htaccess`, `wp-content/litespeed/`,
`advanced-cache.php` and its upgrade folders itself, beyond any path declared as persistent.

Record who owns which path in `.cyon/journal.md`, because the next person to deploy needs to
know.
