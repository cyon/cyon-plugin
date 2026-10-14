# Changelog

What changed in the cyon plugin for Claude Code and Codex, newest first. Claude Code and Codex
update an installed plugin only when its version moves, so every change here has its own
version. The tools themselves come from the cyon MCP server and change with it, not with the
plugin.

## 0.3.8 (2026-10-14)

- The MCP address setting asks to keep the address unless cyon support says otherwise.

## 0.3.7 (2026-10-14)

- The icon is a 1024 pixel PNG, which the Claude directory asks for; the SVG stays beside it.

## 0.3.6 (2026-10-14)

- `cyon-cms`, `cyon-operations` and `cyon-provisioning-checklist` no longer offer a snapshot before
  an update: cyon takes none per deployment, and its daily backup guarantees no restore point, so
  the backup before a change on a live site is taken over SSH. `cyon-operations` names cyon's daily
  backup and the database dumps of the last three days in `~/backups_cyon/mysql`.

## 0.3.5 (2026-10-14)

- The OpenAI listing names cyon AG as developer, as the other manifests do.

## 0.3.4 (2026-10-13)

- `mcp.json` declares the server with its type and URL only, as the Agent Plugins schema allows;
  the sign-in is discovered from the server. The OpenAI manifest carries the review test cases
  and release notes.
- The README names every destination the assistant reaches: the MCP server, the customer's hosting
  over SSH and the customer's own site over HTTPS.

## 0.3.3 (2026-10-13)

- The terms of service link points at https://www.cyon.ch/legal/agb; the earlier address answered
  404. The Claude marketplace file drops a schema address that does not exist.

## 0.3.2 (2026-10-13)

- The Codex marketplace entry names its installation policy, sign-in on install and category,
  which ChatGPT's workspace import asks for.
- The publisher is cyon AG, in the manifests, the marketplaces and the license.

## 0.3.1 (2026-10-12)

- The listing text, icon and links that ChatGPT and Codex show, and OAuth with client registration
  for the server, so the package can be uploaded to OpenAI's plugin directory.

## 0.3.0 (2026-10-12)

- The server address is the plugin option `mcp_url`, set to `https://mcp.cyon.ch/mcp`. Claude
  Code shows it once when the plugin is installed from `/plugin`. The `CYON_MCP_URL` environment
  variable no longer has any effect: set the option instead, with
  `claude plugin install cyon@cyon --config mcp_url=<url>` or `/plugin configure cyon@cyon`.
- A README for customers, a cyon icon and the listing fields of the Claude plugin directory.
- The plugin is published under the MIT license.

## 0.2.7 (2026-10-09)

From the internal pilot's feedback:

- `cyon-operations` changes the crontab with a copy first and only inside a block marked for the
  site, since the crontab is shared by the whole account and a piped rewrite can empty it. It
  also switches maintenance mode on before a dump or a migration. `cyon-cms` and `cyon-php-app`
  point there.
- `cyon-php-app` puts the application inside `public_html`, where the tools accept a document
  root, and names SQLite's single writer and `BEGIN IMMEDIATE`. `cyon-platform` no longer
  denies the `sqlite3` tool and no longer lists the file copy as a backup.
- `cyon-cms` and `cyon-provisioning-checklist` ask for a real mailbox behind a shop's sender
  address, and `cyon-cms` keeps WordPress uploads without `--delete`.
- `cyon-static-site` and `cyon-php-app` keep the licence notices of third-party code.

## 0.2.6 (2026-10-09)

- `cyon-static-site`, `cyon-php-app`, `cyon-cms` and `cyon-provisioning-checklist` put a domain
  the customer already registered on the hosting with `domain_add`. Registering a new domain stays
  an order in my.cyon.

## 0.2.5 (2026-09-29)

- `cyon-static-site` has a section for clients without a shell, such as Claude on the web and
  ChatGPT: they publish with `artifact_publish`, which writes text files into the document root
  and refuses PHP, dotfiles and binary files.
- `cyon-platform` and `cyon-troubleshooting` read the site's PHP settings without a shell
  through `artifact_publish` with `phpinfo: true`, whose page deletes itself after fifteen
  minutes. `cyon-provisioning-checklist` lists the tool.

## 0.2.4 (2026-09-29)

- `cyon-static-site` and `cyon-provisioning-checklist` have the assistant ask the customer in the
  conversation before a deploy key or the Let's Encrypt terms, and send `confirm_shell_access`
  or `accept_terms` after a yes. The MCP server no longer sends these questions to the client.

## 0.2.3 (2026-09-28)

- This changelog.

## 0.2.2 (2026-09-28)

- `cyon-static-site` uploads with `rsync --chmod=Dgo+rx,Fgo+r`. Without it, `-a` copies the mode
  of the local build folder onto the document root, and a folder made in a private temp
  directory turns the whole site into a 404.
- `cyon-static-site` has the deploy key made with `ssh-keygen` and passes its private key as
  `environment.identity_file`, so the returned upload command offers that key. If the client
  shows no approval question, the customer adds the key in my.cyon under SSH keys instead.
- `cyon-troubleshooting` lists a document root with mode 700 as a cause of a 404 right after a
  deployment.

## 0.2.1 (2026-09-24)

- The README sets the development URL for Claude Code in the `env` block of
  `~/.claude/settings.json`, which reaches Claude Code in an editor and on a Mac that still runs
  bash, where a shell profile does not.

## 0.2.0 (2026-09-24)

- Five new skills: `cyon-static-site`, `cyon-php-app` and `cyon-cms` for the three kinds of
  project, `cyon-troubleshooting` and `cyon-operations` for running them. Migration is a section
  of `cyon-preflight`.
- SQLite is the first choice wherever the software supports it, with its limits named: no
  phpMyAdmin, no access over the network, short write transactions, backups through
  `VACUUM INTO`.
- Corrected platform facts: the account ini is `00-mycyon.ini`, `imagick` needs no `exec()`,
  OPcache and the disabled functions differ per server and are read from the site, the MCP
  accepts the Let's Encrypt terms, and a hosting with more than fifty sites on the old
  certificate workflow needs my.cyon for HTTPS.
- The first version that moves with its content. Earlier installs of 0.1.0 may carry either the
  three-skill or the eight-skill plugin.

## 0.1.0 (2026-09-11)

- The plugin for Claude Code and Codex, with the connection to the cyon MCP server and the skills
  `cyon-platform`, `cyon-preflight` and `cyon-provisioning-checklist`.
