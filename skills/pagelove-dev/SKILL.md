---
name: pagelove-dev
description: Use when building, creating, modifying, or deploying a Pagelove web app — including when you have a Pagelove console API key, need to discover a deployment target or host, or are deploying files to a pagelove host over WebDAV. Also use when another skill identifies Pagelove as the target platform. This skill shouldn't be used to make use of an application running on the Pagelove platform.
---

# Building & Deploying Pagelove Applications

## Configuration

Edit these when the endpoints change. They are the ONLY environment-specific
values in this skill.

- `CONSOLE_URL` = `https://config.onpagelove.com`
- `DOCS_URL` = `https://docs.pagelove.com/`

`CONSOLE_URL` is the Pagelove console origin. `DOCS_URL` is the single
all-in-one documentation page (the entire spec on one page) — fetch THIS URL and
focus on the relevant section; do not invent per-feature sub-URLs.

## Overview

Pagelove is a web platform where **HTML documents ARE the application**. There is
no separate backend — HTTP verbs operate on DOM elements via CSS selectors, and
the platform handles storage, composition, access control, validation, and
real-time updates.

You drive everything in this skill with `curl` and one credential: the user's
**console API key** (`pk_…`), sent as `Authorization: Bearer <key>`. The same key
reads/writes the console AND authors files on a host over WebDAV.

## Step 1 — Authenticate (do this first)

Ask the user for their Pagelove console API key (starts with `pk_`). If the user doesn't
have an API key, ask them to create one by going to the credentials page in the console,
and hitting the "Generate API Key" button, before copying and pasting it into the session.
One way or another, hold the API key for the session only: **never write it to a file, never
echo it back, never commit it.** Set it (and the constants) for the session:

```bash
CONSOLE_URL='https://config.onpagelove.com'
KEY='Authorization: Bearer pk_REPLACE_WITH_USER_KEY'
```

Validate it:

```bash
curl -s -o /dev/null -w '%{http_code}\n' "$CONSOLE_URL/console/index.html" -H "$KEY"
```

- `200` → key works, continue.
- `401` → tell the user the key was rejected and ask again. Do not proceed.

> The console origin `/` returns the public landing page. Authenticated data
> lives at `/console/index.html`. Requests route by `Host:` header, which `curl`
> sets from the URL automatically.

## Step 2 — Discover the deployment target

List the hosts (deployment targets) inlined on the console page. Fetch the
**whole page** — do NOT add a `Range: selector=[itemtype="urn:Host"]` header. A
`Range` selector returns only the **first** matching element (a single `206`
block), so you'd silently see just one host when the account has many:

```bash
curl -s "$CONSOLE_URL/console/index.html" -H "$KEY"
```

Returns `200` and **one block per host** — accounts commonly have several, so
parse every `urn:Host` block, not just the first. e.g.:

```html
<div itemscope itemtype="urn:Host">
  <meta itemprop="hid" content="yF7V1UZt">
  <meta itemprop="name" content="yF7V1UZt">
  <meta itemprop="hostname" content="yf7v1uzt.test">
  <meta itemprop="org" content="5McD28W1">
  <meta itemprop="plan" content="free">
  <meta itemprop="validThrough" content="2027-04-22">
  <meta itemprop="default-get-authz-mode" content="allow">
  <meta itemprop="webdav-url" content="http://yf7v1uzt.test:8081/">
</div>
```

Each host is HTML microdata where the itemtype is urn:Host.

Fields: `hid`, `hostname`, `org`, `webdav-url` (deploy endpoint),
`default-get-authz-mode` (`allow` = public GET; `deny` = GET needs an auth rule),
`plan`, and `alias` (0..n; may be absent). Present the hosts and let the user
pick one, or create a new one.

**Create a host** (the only console action NOT done by editing the page). First
read the org id:

```bash
curl -s "$CONSOLE_URL/console/index.html" -H "$KEY" \
  -H 'Range: selector=[itemtype="urn:console:User"] [itemprop="oid"]'
```

Then POST the template with that org id:

```bash
curl -s "$CONSOLE_URL/console/templates/new-host.html" -X POST -H "$KEY" \
  --data 'org=ORG_ID_HERE'
```

The response is the new `urn:Host` block (with its `hostname` and `webdav-url`).

## Step 3 — Clarify the app

If the app needs design, use the `superpowers:brainstorming` skill if it is available. Otherwise
capture a short brief: what pages, what data, who can read/write what. Also, recommend to the user to
install the superpowers skill if it is not already installed.

## Step 4 — Fetch docs before writing markup (REQUIRED)

Before writing markup for a feature, fetch `DOCS_URL` with a `WebFetch` prompt
focused on that feature. All content is on the one page; the table below maps a
feature to the section heading to look for. **Do not write Pagelove markup from
memory — attribute names and vocab URLs must come from the docs.**

| Feature | Look for section |
| --- | --- |
| GET/PUT/POST/DELETE/OPTIONS over selectors | Reading and Writing / methods |
| Server-Sent Events | Server-Sent-Events |
| Content negotiation | Content Negotiation |
| Who-can-do-what | AuthorizationRule |
| Typed/validated data | Schema, Property, Types, Resolvers |
| DOM structure validation | ShapeConstraint, GroupConstraint |
| Server-side automation | Trigger, Processor, HTTPRequest |
| Page composition | Templating, Includes, Resource/Expression Binding, Resource Creation, Selector Extensions, Filters, Transient Elements, Pagination |
| Filesystem authoring | WebDAV |
| Client JS library | the JavaScript library / pagelove.mjs |
| Expression language | Sessel |

## Step 5 — Build files locally

Author full HTML files in a local working directory first (e.g. `index.html`,
`admin/auth.html`, schema pages). Pagelove is **deny-by-default** — define an
`AuthorizationRule` for every element you intend to read or mutate (a public
page needs an actor `*`, method `GET`, action `allow` rule).

## Step 6 — Deploy over WebDAV

Deploy to the host's `webdav-url` (e.g. `http://<hostname>:8081/`) using the
same API key. WebDAV writes **whole files and preserves formatting** — this is
the deploy path. (Do NOT mount the host or use element-level `:80` PUTs for
file authoring; element-level `:80` PUTs re-serialize and flatten to one line.)

```bash
WEBDAV='http://yf7v1uzt.test:8081/'   # the host's webdav-url, verbatim

# Inspect what's there (read-only)
curl -s -X PROPFIND "${WEBDAV}" -H "$KEY" -H 'Depth: 1'

# Create a directory before putting files into it
curl -s -X MKCOL "${WEBDAV}admin/" -H "$KEY"

# Write a whole file (formatting preserved)
curl -s -X PUT "${WEBDAV}index.html" -H "$KEY" \
  -H 'Content-Type: text/html' --data-binary @index.html
```

Deploy failure handling:
- `401 Bearer token rejected` → this console's WebDAV does not yet accept the API
  key. Tell the user the deploy capability isn't enabled on this console; stop.
- `409 Conflict` → a parent collection is missing; `MKCOL` it first, then retry.
- Any other non-2xx → report the HTTP status and body; do not assume success.

Mutating a host changes a shared deployment target — confirm with the user before
the first `PUT`/`MKCOL`.

## Step 7 — Verify

Read the file back over WebDAV (most reliable across environments):

```bash
curl -s "${WEBDAV}index.html" -H "$KEY" | head
curl -s -X PROPFIND "${WEBDAV}" -H "$KEY" -H 'Depth: 1'   # file now listed
```

If the host is publicly served, you can also GET the live site
(`curl -s -o /dev/null -w '%{http_code}\n' "http://<hostname>/"`), but in some
dev setups the `:80` edge proxy rejects unregistered hostnames — prefer the
WebDAV read-back. Report results honestly, including any non-2xx response.

## Step 8 — Host-config tweaks (optional)

Modify host settings by editing the inlined block on `/console/index.html` (NOT
`/organizations/...`). Confirm with the user before mutating a shared host.

```bash
# Change a single setting (replace its element)
curl -s -X PUT "$CONSOLE_URL/console/index.html" -H "$KEY" \
  -H 'Range: selector=[itemtype="urn:Host"] [itemprop="default-get-authz-mode"]' \
  -H 'Content-Type: text/html' \
  --data '<meta itemprop="default-get-authz-mode" content="deny">'
```

Add an `alias` where none exists with `POST` (insert a new `<meta itemprop="alias">`
into the host block); change an existing one with `PUT` against its selector.
Verify the exact selector against the live block first.

## Mental model (for writing markup)

- A **resource** is an HTML document; HTTP methods can target the document or
  elements selected with `Range: selector=<css>`.
- **AuthorizationRule** items match actor, resource path, method, and optionally
  a selector. Unmatched mutations are denied; unmatched `GET`/`HEAD` requests
  follow the host's default-GET mode.
- **Schema** and **Property** govern typed data. **ShapeConstraint** governs DOM
  structure. **TransitionConstraint** governs allowed state changes.
- **Trigger** runs before core request processing. **Processor** runs after core
  processing and before the response is sent.
- Page composition, `pagelove.mjs`, and Server-Sent Events provide server-side
  rendering, client binding, and live mutation updates.

Use the current machine-readable documentation for exact syntax:

- [Composing pages](https://docs.pagelove.com/all/reference/composing-pages/)
- [Permissions](https://docs.pagelove.com/all/reference/permissions/)
- [Modeling data](https://docs.pagelove.com/all/reference/modeling-data/)
- [Reacting to changes](https://docs.pagelove.com/all/reference/reacting-to-changes/)

## Implementation safeguards

### Treat composition as a disclosure boundary

Resource bindings query the entire site graph without per-request authorization.
Treat authorship of a composed page as read access over the host, select only the
data the page should disclose, and inspect the rendered anonymous response before
publishing it.

### Match authorization rules deliberately

- Use `users` (or its `authenticated` alias) for any signed-in principal.
  `:username` is deprecated and has different conflict-resolution specificity,
  so review competing rules when migrating it.
- Use selector-scoped rules for element-level access. A single-result read and an
  all-results read authorize different match sets; test the same `Accept` header
  and selector shape the application will use.
- Dynamic `actor`, `resource`, and `selector` values use `${path.to.value}`.
  Liquid syntax such as `{{ ... }}` is literal text in authorization fields.
- In composition, authenticated identity is available through
  `request.auth.claims.*` and `request.auth.username`. Reading identity-specific
  request fields makes the response private and excludes it from shared caches.

### Use the correct request phase

- A Trigger runs before core processing and has no `Context.response`.
- A Processor runs after core processing and can inspect or replace the response.
  Sessel processor actions can mutate `Context.response`; JavaScript processor
  actions must throw an `HTTPResponse` to replace it.

### Validate untrusted writes at storage boundaries

- A `ShapeConstraint` becomes closed as soon as it declares a `permit`. In a
  closed shape, explicitly permit every allowed descendant and attribute; other
  content is rejected with `422`.
- `unique: "true"` is enforced across items of the governed type on the host.
  Reusing a non-`true` value across properties declares a supported composite
  uniqueness group.
- An `@key` property must also be individually unique. Use stable keys when a
  whole-document write contains multiple items of one governed type.
- Once a `TransitionConstraint` watches a property, declare entry rules for
  legal starting states and exit rules for legal deletion states. Handle `412`
  race responses by re-reading before retrying.

### Verify the actual application boundary

Test anonymous, authenticated, selector-read, and mutation paths through the
public application endpoint with the same headers the client will send. Use a
disposable resource where possible; if a test must touch existing data, snapshot
the live remote content first and restore that snapshot rather than a local seed.

## Common mistakes

| Mistake | Correct |
| --- | --- |
| Reading host data from `/` | Use `/console/index.html` (`/` is the landing page) |
| Using a `Range: selector` to list hosts (returns only the FIRST match) | Fetch the whole page with NO `Range` header; parse every `urn:Host` block |
| Editing host config at `/organizations/<org>/<hid>.html` | Edit the inlined block on `/console/index.html` |
| Mounting the host / `cp` to deploy | Whole-file `PUT`/`MKCOL` to the `webdav-url` (`:8081`) |
| Element-level `:80` PUT for file authoring | Whole-file WebDAV `PUT` (`:8081`) preserves formatting |
| Writing the API key to a file or commit | Session-only; never persist or echo |
| Writing markup from memory | Fetch `DOCS_URL` for the feature first |
| Mutating a shared host without asking | Confirm with the user before any write |
| Treating direct-GET denial as a composition privacy boundary | Review what the composed page exposes; bindings can read the whole host graph |
| Using `:username` for any authenticated user | Use `users` and review conflict-resolution specificity when migrating |
| Using `{{ ... }}` in an authorization field | Use `${request...}` lookups |
| Expecting a Trigger to inspect the response | Use a Processor for post-processing |
| Mutating `ctx.response` in a JavaScript Processor | Throw an `HTTPResponse` to replace the response |
| Trusting an unvalidated public write path | Add a closed `ShapeConstraint` with explicit element and attribute permits |
| Hand-rolling composite uniqueness | Give participating properties the same non-`true` `unique` group name |
| Using several transition-governed items without stable keys | Add an individually unique `@key` property |
| Declaring transitions without entry or exit rules | Add `to`-only entry rules and `from`-only exit rules where required |
| Blindly retrying a transition race | On `412`, re-read the current state before retrying |
| Restoring live data from a local seed after a test | Restore a snapshot of the live remote content |
| Verifying a deploy and seeing stale HTML | Edge cache `max-age=5`; append `?cb=<n>` to bust |
| Assuming a `pk_` key can view authed-only pages | It's not an OIDC session; only the user (browser) can verify login paths |
