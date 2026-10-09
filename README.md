# cyon plugin for Claude Code and Codex

Skills that know what cyon hosting runs, and the connection to the cyon MCP server, through which
the assistant reads your hostings and creates subdomains, databases and certificates and
publishes a site.

## Installing

Claude Code:

```
claude plugin marketplace add cyon/cyon-plugin
claude plugin install cyon@cyon
```

Codex:

```
codex plugin marketplace add https://github.com/cyon/cyon-plugin.git
codex plugin add cyon@cyon
codex mcp login cyon
```

At the first tool call, or at `codex mcp login`, your browser opens my.cyon. Log in and approve
the connection, and the tools appear. You choose there which hostings the assistant may reach.

## Contents

The skills are in `plugin/skills/`, what changed in each version is in `plugin/CHANGELOG.md`.

This repository is published from cyon's own repository, so pull requests are not merged here.
Questions and problems go to cyon support.
