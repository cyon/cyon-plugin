---
name: cyon-provisioning-checklist
description: The steps on a cyon hosting that a person has to perform in my.cyon, as an ordered runbook to hand over. Use when preparing a cyon hosting for a deployment, when a tool call refuses because a prerequisite is missing, or when asked what the customer has to click before a site can go live. Says which steps the cyon MCP server performs itself and which genuinely need a human, so nothing is claimed as automated that is not.
cyon-facts-version: 2026-10-09
---

# Provisioning a cyon hosting

Most of preparing a hosting is tool calls now. This skill is the honest remainder: what still
needs a person, and how to hand it over so they can do it in one pass.

Read `cyon-platform` for what the platform runs, and `cyon-preflight` for auditing a codebase.

## What the MCP server does itself

Do not put these in a runbook for the customer. Call them:

| Step | Tool |
| --- | --- |
| Find the hosting and its sites | `hosting_list`, `domain_list`, `site_list`, `site_get` |
| Read plan, quotas and available PHP versions | `hosting_get` |
| Create a subdomain with its directory and DNS | `subdomain_create` |
| Put a domain the customer already registered on the hosting, with its directory and DNS zone | `domain_add`; registering a new domain is an order in my.cyon |
| Point an existing domain at another directory | `site_docroot_set` |
| Set the PHP version for a document root | `php_version_set` |
| Create a database, its user and the grant | `database_create` |
| Request a Let's Encrypt certificate, then poll it | `https_enable`, `certificate_get` |
| Accept the Let's Encrypt subscriber terms | `https_enable` with `accept_terms`, after the customer accepted them in the conversation; recorded once per contact |
| Send visitors from HTTP to HTTPS | `https_enable` once the certificate is live, which sets it on the site rather than writing `.htaccess` |
| Install an SSH key for the deployment | `deploy_key_add` |
| Validate the layout and get the upload commands | `deploy_prepare` |
| Publish a static site from a client without a shell | `artifact_publish` |
| Switch PHP error logging on | `deploy_prepare`, whose `enable_php_error_logging` covers the **whole hosting**, not the selected site |

Every write needs a one-line summary and an idempotency key, and asks the customer first.

## What still needs a person in my.cyon

Six things, and each has a symptom you will meet if it was skipped:

| Step | Where | Symptom if missing |
| --- | --- | --- |
| Set up the account's own `php.ini` | Einstellungen, PHP-Einstellungen | Custom ini settings are ignored silently. It also starts with OPcache on, which is not on every server without it |
| Enable a disabled function, such as `exec` or `proc_open` | Einstellungen, PHP-Einstellungen | `symfony/process` or a shell-out to `/usr/bin/convert` fails at runtime |
| Activate HTTPS on a hosting with more than fifty sites on the old certificate workflow | Sicherheit, Kostenlose SSL-Zertifikate | `https_enable` refuses with `https_unavailable` and requests nothing; the portal activates per site and works there |
| Raise the mail limit above 100 per hour | E-Mail, Versandlimit | Mail is throttled silently, order confirmations disappear |
| Create a mailbox for a shop's or a form's sender address | E-Mail, E-Mail-Konten | Replies and bounces land nowhere, while the site still reports every mail as sent |
| WebSockets on a Managed Server | On request, through support | A WebSocket server does not serve |

The Let's Encrypt terms need the customer's yes, not a click in my.cyon: `https_enable` refuses
with `terms_not_accepted` until it carries `accept_terms: true`, which the assistant sends after
showing the customer the terms and hearing that they accept them. The acceptance is recorded
once per contact.

When the MCP switches PHP error logging on, it writes into the same account ini the PHP
settings page manages, keeping what is already there.

Enabling a function is a security decision, not a checkbox. Say what it widens before asking
for it, and prefer the alternative where there is one: the `imagick` extension, which is loaded
by default, instead of shelling out to ImageMagick.

## Handing the runbook over

Order the steps so the customer does one pass through my.cyon, not three:

1. Everything under PHP-Einstellungen together, the `php.ini` first because the rest depends on it.
2. HTTPS in the portal, only where `https_enable` refused.
3. The mail limit, if the application sends more than a trickle.
4. Anything needing support, which has a lead time.

For each step give the exact place in my.cyon, why it is needed for *this* project, and what
happens if it is skipped. A customer who knows the consequence does the step; a customer given
a bare list of settings does not.

Then say plainly which steps you performed and which are theirs. Never describe a tool call as
something they need to do, and never describe their step as done.

## After provisioning

Before publishing, confirm the layout with `deploy_prepare`: it validates the declared
persistent paths, checks Git is not tracking them, and returns the upload command that matches
the shell the assistant actually has. Snapshots are optional and cover only the declared paths,
so do not promise a restore that does not exist.
