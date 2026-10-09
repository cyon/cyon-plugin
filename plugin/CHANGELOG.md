# Changelog

What changed in the cyon plugin for Claude Code and Codex, newest first. Claude Code and Codex
update an installed plugin only when its version moves, so every change here has its own
version. The tools themselves come from the cyon MCP server and change with it, not with the
plugin.

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
