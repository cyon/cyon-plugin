# The cyon plugin

Two payloads: the skills, which are platform knowledge that runs on the customer's machine,
and the declaration of the cyon MCP server's URL. No credential, which is why this can be
published openly and why setup is two steps.

## Installing

While the plugin is internal it is served from a cyon hosting, which asks for no account, no
token and no SSH key. Customers will later install the same thing from the GitHub mirror, where
the source shortens to `cyon/cyon-plugin`.

Claude Code:

```
claude plugin marketplace add http://mvtester.cyon.site/cgi-bin/git/cyon-plugin.git
claude plugin install cyon@cyon -y
```

Codex, which reads `plugin.json` and `mcp.json` from this directory through the marketplace
file under `.agents/plugins/`:

```
codex plugin marketplace add http://mvtester.cyon.site/cgi-bin/git/cyon-plugin.git
codex plugin add cyon@cyon
codex mcp login cyon
```

The selector is `cyon@cyon` in both: the marketplace and the plugin are both named `cyon` in
the manifests, which is independent of what the repository is called or where it is served
from.

That URL goes through `git-http-backend` as CGI, not to a git host. Both installers clone
shallow, which plain file serving cannot answer, so `http://mvtester.cyon.site/cyon-plugin.git`
is the same repository for a manual `git clone` but is refused by the installers.

Then sign in: the client opens my.cyon, you log in, approve the connection, and the tools
appear. Claude starts that flow at the first tool call, Codex at `codex mcp login`. Nothing is
written to a configuration file by hand, and a remote server pasted into
`claude_desktop_config.json` is ignored, because connectors live in the app's own state.

Found a problem? Report it in `dev/froox/mycyon-mcp-server`, which is where the plugin is
edited. Changing the published copy is pointless, the next publish overwrites it.

## Layout

| Path | For |
| --- | --- |
| `.claude-plugin/plugin.json` | Claude Code: name, version, description |
| `.mcp.json` | Claude Code's MCP declaration. An `mcpServers` block inline in the plugin manifest is ignored |
| `plugin.json` | Agent Plugins 1.0.0, which Codex reads |
| `mcp.json` | Agent Plugins MCP declaration, `streamable-http` |
| `skills/<name>/SKILL.md` | Both ecosystems, discovered automatically |

## Pointing it at another server

The plugin ships the production URL, which is not the one an internal tester wants: during the
pilot the server runs on staging, at `https://mcp-staging.cyon.lan/mcp`, reachable over the cyon
VPN. People working on the service itself point at their own stack instead, usually
`https://mcp.cyon.tld/mcp`. Either way it is the same override, and the two ecosystems differ in
how it is spelled, because Claude Code expands environment variables in a plugin's MCP URL and
Agent Plugins 1.0.0 forbids it (§7.2: clients "MUST NOT perform placeholder or
environment-variable expansion in `url`").

**Claude Code.** Set the variable, and the plugin connects there instead:

```
export CYON_MCP_URL=https://mcp-staging.cyon.lan/mcp
```

Set it where the client runs, not in a container, and in the shell's own configuration file so
it survives a restart. The same value in the `env` block of `~/.claude/settings.json` works in every shell and when
Claude Code runs inside an editor, which reads no shell profile. Unset, it uses the production
URL. The server keeps the name `cyon` either way, so tool names
are `mcp__cyon__*` in both environments and an eval written against one runs against the other.

**Codex.** There is no partial override for a plugin-provided server, so shadow it with a
complete block. `codex mcp add cyon --url https://mcp.cyon.tld/mcp` writes one into
`~/.codex/config.toml` and that entry wins over the plugin's, which `codex mcp list` then
confirms. By hand, project configuration wins over the global one:

```toml
[mcp_servers.cyon]
url = "https://mcp.cyon.tld/mcp"
```

## Skills

Phase 1 of five, the foundation the rest depends on:

| Skill | Purpose |
| --- | --- |
| `cyon-platform` | What the platform runs, what it does not, and the fit check before any architecture |
| `cyon-preflight` | Audits an existing codebase, splitting blockers from provisioning steps from adjustments |
| `cyon-provisioning-checklist` | The steps that still need a person in my.cyon, and which ones the tools do instead |

Each carries a `cyon-facts-version`, the snapshot of the platform contract it was written
against, so a stale skill is detectable.
