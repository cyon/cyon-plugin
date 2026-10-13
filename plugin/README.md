# cyon

Build and publish a website on your cyon hosting with Claude or Codex. The plugin brings two
things: skills that know what cyon hosting runs and how to deploy to it, and the connection to
the cyon MCP server, through which the assistant works on your hostings after you approve it in
my.cyon.

## What the assistant can do

- Check whether a project fits cyon before anything is built: PHP, static sites, WordPress and
  other CMS, SQLite or MariaDB, cron jobs, and what does not run here, such as a Node server.
- Read your hostings, domains, sites, databases, PHP versions and certificates.
- Create a subdomain, put a domain you already registered on a hosting, create a database, set
  the PHP version, and switch on HTTPS with a Let's Encrypt certificate.
- Publish a site: over SSH with a deploy key that expires after fifteen minutes, or, in a chat
  client without a shell, by sending the files of a static site.
- Search cyon's support articles.

Every change asks for a one-line reason and is listed in my.cyon under KI-Agent, where you also
revoke the connection.

## Installing

Claude Code:

```
claude plugin marketplace add cyon/cyon-plugin
claude plugin install cyon@cyon
```

Codex:

```
codex plugin marketplace add cyon/cyon-plugin
codex plugin add cyon@cyon
codex mcp login cyon
```

At the first tool call, or at `codex mcp login`, your browser opens my.cyon. Log in, choose which
hostings the assistant may reach, and approve. Claude Code asks once for the server address when
you install the plugin from `/plugin`; keep the address it suggests.

## What it connects to and sends

The plugin runs nothing on your machine. The skills are text the assistant reads, and they send
nothing by themselves. The assistant reaches three destinations, all of them yours or cyon's:

- **cyon's MCP server, `https://mcp.cyon.ch/mcp`**, the connector the plugin declares. It signs you
  in through my.cyon with OAuth and acts only on the hostings you approved. A tool call sends the
  arguments the assistant chose, such as a domain name, a one-line reason, or the files of a
  static site. cyon keeps a journal of every change for six months and request logs for 30 days.
- **Your cyon hosting over SSH**, in a client with a shell such as Claude Code: `ssh`, `rsync`
  or `git push` to the server that `connection_get` names, to upload your project and run
  commands there, with a deploy key that expires after fifteen minutes. Your client asks for
  permission before running a command, as its settings define.
- **Your own website over HTTPS**, with `curl`, to check its status, headers and response time.

Nothing goes anywhere else, and the plugin sends no analytics. Your conversations are not sent to
cyon.

Revoke access at any time in my.cyon under KI-Agent.

## Support

Questions about your hosting go to cyon support. Problems with the plugin or the tools can be
reported in this repository's issues.
