---
name: cyon-static-site
description: Build and publish a static site on cyon hosting, from plain HTML to Astro, Vite, Eleventy and Hugo, or Next, Nuxt and SvelteKit as a static export. Use when a project needs no server-side logic, when a contact form is its only dynamic part, or when a static frontend gets a small PHP API beside it. Covers the local build, the disposable publish target, .htaccess for redirects and headers, and the form endpoint.
cyon-facts-version: 2026-09-29
---

# A static site on cyon

The cheapest thing to run here and the hardest to break: files built on the customer's
machine, served by LiteSpeed. Most first projects are this, including many that were planned
as something heavier.

Read `cyon-platform` first. If the design needs a page rendered per request by Node, it is not
a static site and does not fit; say so before building.

## Does it fit

Static means every page can be built in advance. It still fits when:

- content comes from Markdown, a headless CMS or an API **at build time**,
- the browser fetches data at run time from a public API,
- one form posts to a small PHP endpoint, see below.

It stops fitting when a page must be rendered per visitor on the server, or when there are
logins and per-user data. That is `cyon-php-app` or `cyon-cms`.

## Framework settings for a static export

| Framework | Setting | Trap |
| --- | --- | --- |
| Astro | Default static output | An adapter such as `@astrojs/node` turns it back into a server |
| Vite, plain SPA | `vite build`, publish `dist/` | Client-side routing needs the fallback rule below |
| Eleventy, Hugo | Default output | None worth naming |
| Next | `output: 'export'` | `next/image` needs `images: { unoptimized: true }`, since its default loader is a server. `trailingSlash: true` writes `page/index.html`, which LiteSpeed serves without rewrites |
| Nuxt | `ssr: false`, then `nuxi generate` | `nuxi build` produces a server, not files |
| SvelteKit | `@sveltejs/adapter-static` | `adapter-auto` and `adapter-node` produce a server |

Build locally or in CI, never on the hosting: a build under LVE is slow and fragile, and Node
is installed as a command line tool only.

## The publish target

Publish into a directory that holds **only** the build output, so replacing it is always safe:

- a fresh subdomain for a new project, whose directory is the build output and nothing else,
- or a dedicated directory an existing domain points at.

Never publish over a directory that holds anything the build did not make: uploads, a CMS,
another site. When unsure, inspect it first or use a fresh subdomain.

Replacing the whole target is then one command, run from the build directory:

```bash
rsync -az --delete --chmod=Dgo+rx,Fgo+r --exclude '.well-known/' ./dist/ <login>@<host>:<document root>/
```

`--delete` is what keeps removed pages from lingering, and it is only safe because the target
is disposable. The exclusion keeps certificate validation files alone. `--chmod` matters because
`-a` copies the mode of `dist/` onto the document root: a build folder made in a private temp
directory is 700, and the whole site then answers 404.

Keep `.htaccess` in the build's public folder, so it ships with every deployment rather than
living only on the server.

## .htaccess for a static site

LiteSpeed reads `.htaccess` and `mod_rewrite` works. Sending visitors to HTTPS is a setting
on the site, not a rule here; do not write one.

```apache
# a 404 page from the build
ErrorDocument 404 /404.html

# client-side routing: unknown paths go to the app, real files are served as they are
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ /index.html [L]

# fingerprinted assets never change, so they can be cached for a year
<FilesMatch "\.(js|css|woff2|png|jpe?g|webp|avif|svg)$">
  Header set Cache-Control "public, max-age=31536000, immutable"
</FilesMatch>
```

Only cache for a year what the build fingerprints. HTML must stay short-lived, or visitors
keep an old page that points at assets the next deployment deleted. After publishing, check
the headers with `curl -sI` rather than assuming the rules took.

## A contact form

The one dynamic piece most static sites need, and it does not need a framework: a small PHP
script beside the pages, which PHP on every product runs per request.

- **Sender and reply.** Exim only sends from an address on a domain of the same hosting. Send
  from that address and put the visitor's address in `Reply-To`, never in `From`.
- **Limits.** Script mail is capped at 100 per hour, and a hosting in its first 30 days at 30.
  A contact form stays far below; a form that mails a list does not belong here.
- **Spam.** A hidden honeypot field and a minimum time between page load and submit stop most
  of it without a third-party captcha. Add one only if that is not enough.
- **Keep a copy.** Mail gets lost. Store each submission as well, and SQLite is the natural
  store for that here: one file outside the document root, WAL mode, `busy_timeout` of a few
  seconds. The file is state, so it never lives inside the disposable target.
- **Validate on the server**, escape everything that goes into the mail, and answer with a
  redirect to a thank-you page rather than an echo of the input.

## A static frontend with a PHP API

When the frontend needs more than a form, the API is a PHP application in its own right, see
`cyon-php-app`. Two shapes:

- **Same origin, preferred.** The API lives under `/api/` of the same site, in its own
  directory with its own PHP. No CORS, no second certificate.
- **A subdomain**, such as `api.example.ch`, when the API must deploy on its own schedule. It
  then needs CORS headers naming the frontend's origin exactly, never `*` for anything that
  carries a cookie or a token.

Either way the frontend stays a disposable target and the API does not: the API has state and
deploys the way `cyon-php-app` describes.

## With the MCP server connected

In this order:

1. `domain_list` and `site_list` to resolve the hosting and the exact site, or
   `subdomain_create` for a fresh one, which is the default for a new project.
2. `php_version_set` only if the site has a form or an API endpoint.
3. `deploy_prepare` with `type: static` and `dedicated_static_output: true`, which you set only
   after confirming the target holds nothing but build output.
4. `deploy_key_add` with a key made by `ssh-keygen`, sending its `.pub` file and the private
   key's path as `environment.identity_file`, then the upload command it returns. The key gives
   shell access to the whole account: ask the customer first, and send
   `confirm_shell_access: true` only after a yes.
5. `https_enable`, then `certificate_get` until the certificate is active, then `https_enable`
   once more so the redirect to HTTPS applies.

Record the site, the target and the commands in `.cyon/journal.md`.

## Without a shell

Claude on the web, Claude Desktop and ChatGPT have no terminal, so there is no build and no
SSH. They publish with `artifact_publish`, which writes the files it is sent into the site's
document root:

1. `site_list` for the exact site, or `subdomain_create` for a fresh one, the default here too.
2. `artifact_publish` with the files, each as a path relative to the document root and its
   complete text. It creates directories and overwrites a file of the same name, so ask the
   customer before publishing into a site that already has content.
3. `https_enable` and `certificate_get` as above.

Plan the site for what the tool takes:

- **Text files only.** HTML, CSS, JavaScript, SVG, JSON, text, XML and web manifests. An image
  is an SVG, a CSS drawing or a URL on another host. PHP is refused, so a contact form needs a
  client with a shell.
- **No `.htaccess`.** No rewrite fallback, so a single-page app uses hash routing, and no cache
  headers, so asset names carry no fingerprint either.
- **Every byte is generated.** A call carries at most 1 MiB, and in practice far less fits in
  one response. Send a larger site in several calls, each with its own idempotency key, and
  send a file again only when it changed.
- **Nothing is deleted.** A page left out of the next publish stays online. Overwrite it, or
  start over on a fresh subdomain.

The result lists the URL of each file. Open the start page for the customer, and say where the
site lives.
