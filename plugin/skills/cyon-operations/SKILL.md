---
name: cyon-operations
description: Keep a site on cyon hosting running once it is live, with backups that restore, cron jobs that do not overlap, the checks to repeat after a change, what to measure when it gets slow, and what to hand over to the customer. Use after a deployment, when setting up maintenance, or when a working site has become slow. Says where the assistant stops and the customer or cyon takes over.
cyon-facts-version: 2026-09-23
---

# Operating a site on cyon

A deployment ends when the site runs, is backed up, and the customer knows how to live with
it. This skill is that last part, and where the assistant stops.

Read `cyon-platform` first.

## After every deployment

Check what a visitor gets, not what the deployment reported:

- `curl -sI` on the site: status 200, and a redirect to HTTPS once the certificate is active,
- `certificate_get` reports the certificate active,
- `site_get` shows the document root and PHP version you intended,
- one real request through whatever matters: the form, the login, a test order,
- `crontab -l` shows every job with the absolute PHP path,
- no debug files left behind: a `phpinfo()` page, a test script, a copy of the database.

Record the result in `.cyon/journal.md`. A check that was not recorded was not done, for
whoever comes next.

## Backups

A backup is proven only once something has been restored from it. Until then it is a hope.

**MariaDB**, as a nightly dump. The password stays in a file readable only by the account,
never on the command line where the process list shows it:

```bash
# once: a client config built from the credentials file database_create wrote
set -a; . ~/.cyon-db-<name>-<suffix>.env; set +a
umask 077; printf '[client]\nhost=%s\nuser=%s\npassword=%s\n' "$CYON_DB_HOST" "$CYON_DB_USER" "$CYON_DB_PASSWORD" > ~/.my-<name>.cnf
```

```cron
30 2 * * * mysqldump --defaults-extra-file=/home/<login>/.my-<name>.cnf --single-transaction <database> | gzip > /home/<login>/backups/<name>-$(date +\%u).sql.gz
```

`--single-transaction` gives a consistent dump while the site keeps writing. The day of the
week in the name keeps seven and overwrites the oldest. Check `command -v mysqldump` once
before relying on the entry.

**SQLite** has no command line tool on the hosting, and copying the file mid-write gives a
corrupt snapshot. Let PHP take a consistent copy, with a small script in `~/bin`:

```php
<?php
// ~/bin/sqlite-backup.php <database> <target>: a consistent copy while the site keeps writing.
// VACUUM INTO refuses an existing target, so last week's copy goes first.
@unlink($argv[2]);
$db = new PDO('sqlite:' . $argv[1]);
$db->exec('VACUUM INTO ' . $db->quote($argv[2]));
```

```cron
45 2 * * * /opt/alt/php83/usr/bin/php /home/<login>/bin/sqlite-backup.php /home/<login>/<app>/data/app.sqlite /home/<login>/backups/app-$(date +\%u).sqlite
```

Either way:

- keep backups outside the document root,
- copy them off the hosting regularly, since a backup on the same account shares its fate,
- cyon's own restore path runs through support and assumes MariaDB; an SQLite file is covered
  only by the application's own backups,
- a `deploy_prepare` snapshot is not a backup: it is optional, covers only the declared paths
  and databases, and has no automatic restore.

## Cron hygiene

- The crontab belongs to the whole account, and every site on the hosting shares it. Keep a copy
  first, change only a block marked for this site, show the customer the difference, and check
  the result:

  ```sh
  crontab -l > ~/crontab.bak
  { sed '/^# BEGIN <site>$/,/^# END <site>$/d' ~/crontab.bak
    echo '# BEGIN <site>'; echo '<the new lines>'; echo '# END <site>'; } > ~/crontab.new
  diff ~/crontab.bak ~/crontab.new
  crontab ~/crontab.new && crontab -l
  ```

  Never pipe a filtered `crontab -l` straight back into `crontab -`: when the first command fails
  halfway, the account's crontab is replaced by an empty one. `crontab ~/crontab.bak` restores it.
- The absolute versioned PHP path in every entry, and `PHP_INI_SCAN_DIR` where custom ini
  settings matter.
- One instance at a time. A job that can run longer than its interval takes a lock:
  `flock -n /home/<login>/.locks/<job>.lock <command>`, or the framework's own
  `withoutOverlapping()`.
- Output into a log file, and set `MAILTO` deliberately. Every cron mail goes out through the
  hosting's mail and counts against its limit, so a job that writes a line every minute can use
  it up.
- Nothing runs continuously. A worker that should be always-on is a cron job every minute that
  finishes its batch and exits.

## Changes that need the site to stand still

A schema migration, a move between databases or a dump that has to be consistent needs a site
that writes nothing in the meantime. Switch maintenance mode on first, then dump, migrate or
move, check, and switch it off last. The framework's own switch does it where there is one,
`php artisan down` in Laravel; a plain site gets a static maintenance page through `.htaccess`.
Say how long the site will be down before switching it off for visitors.

## When it gets slow

Measure before changing anything. `curl -o /dev/null -s -w '%{time_starttransfer}\n'` on one
page, first for a visitor without a cookie and then logged in, separates a caching problem from
a PHP problem.

In order of effect:

1. **LiteSpeed Cache** for anonymous traffic is the main lever, on every product.
2. **OPcache on.** It is on in an account ini set up in my.cyon, not on every server without one.
   Read the site's value first, see `cyon-platform`; switching it on costs nothing.
3. **Less work per request.** No image resizing, no remote API call, no full-table query on a
   page view. Generate derivatives once and cache them on disk; `jpegoptim`, `optipng` and
   `pngquant` are installed for that.
4. **The query behind the slow page.** Indexes help MariaDB here as anywhere.
5. **A bigger product**, which gives more of the same resources. It does not fix code that is
   slow for a reason above.

The assistant cannot see the hosting's LVE usage; if throttling is the suspicion, cyon support
can.

## Handing over

Write it down for the customer, in their language, not the assistant's:

- where the site lives: hosting, document root, application root,
- how to deploy again, as the exact commands,
- where the backups go, how often, and how to restore one,
- every cron job and what it does,
- every credentials file by path, never by content,
- what was switched on in my.cyon and why, from `cyon-provisioning-checklist`,
- that PHP error logging is on, if it was switched on.

## Where the assistant stops

Not a monitor, not an on-call service, and nothing that keeps running. Once the checks pass and
the hand-over is written, the work is done. The deployment key has already expired after its
fifteen minutes; the connection itself the customer can revoke on the connections page in
my.cyon when they no longer need the assistant on this hosting.
