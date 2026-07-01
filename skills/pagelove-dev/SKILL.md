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

List the hosts (deployment targets) inlined on the console page:

```bash
curl -s "$CONSOLE_URL/console/index.html" -H "$KEY" \
  -H 'Range: selector=[itemtype="urn:Host"]'
```

Returns `206` and one block per host, e.g.:

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

- A **resource** is an HTML page; HTTP methods target **elements** via
  `Range: selector=<css>`.
- **AuthorizationRule** microdata controls who can do what to which elements
  (deny-by-default).
- **Schema** + **Property** validate typed data; **ShapeConstraint** validates
  DOM structure on mutations.
- **Trigger**/**Processor** run **Sessel** expressions before/after a request.
- **pagelove.mjs** gives declarative two-way binding; **SSE** streams mutations.
- Microdata vocab URLs are `https://pagelove.org/<Thing>` (e.g.
  `AuthorizationRule`, `Schema`, `Property`, `Trigger`, `Processor`, `Sessel`).

Fetch `DOCS_URL` for exact syntax — do not guess attribute names.

## Verified patterns & gotchas

Battle-tested on real builds. Still fetch `DOCS_URL` for syntax — these are the
things the docs under-state or get subtly wrong.

### Composition runs with ELEVATED rights

A `<p:stamp>`/`<p:include>`/binding composes a page by reading other documents
**even when the requester cannot directly GET them**. (Verified: a public page
that stamps a doc which denies anonymous GET still renders that doc for anon.)

- This is the idiom for **private data + curated views**: keep raw data private
  with `deny * GET /data/*` (+ `allow :username GET /data/*` for the editor), and
  the composed pages (param route, home, index) still render because they compose
  server-side. Visitors can only reach data through the views you build.
- Note: this contradicts the Resource-Binding doc's "binding succeeds only if the
  request has access" line — empirically composition is **not** requester-scoped.

### Write-through authorization is on the COMPOSED PAGE, not the origin

When you `POST`/`PUT`/`DELETE` a stamped/included element (e.g. a comment into a
stamped comments list), authorize it on the **composed page path you address**
(e.g. `/posts/*`), NOT the data origin (`/data/posts/*`). The origin needs no
public write rule — and shouldn't have one. (Shape constraints & ETags, however,
*do* evaluate against the origin.) Always POST to the **stamped/composed
resource**, never the raw data path.

### Parameterized routes (`/posts/:slug.html`)

- A whole-document `GET` with no literal doc resolves the route and exposes
  `request.params.<name>`.
- **Selector writes** (`Range: selector=…`) to a concrete route URL resolve the
  route and write **through to the stamped/included origin** — this is how a
  comment POSTed to `/posts/hello.html` reaches the post's data file.
- Whole-document writes are **literal** (they edit the template doc itself).

### Authenticated identity IS available to composition

Use `request.auth.claims.email`, `request.auth.claims.name`,
`request.auth.username` (= OIDC `sub`) in expression bindings / templates. The
**bare** `auth.claims.*` is authorization-rule context only and throws
`undefined variable` in a binding. Anonymous → empty/falsy.

- Gate author-only content with a ternary binding, e.g.
  `e:post="request.auth.claims.email ? ${…any…}.first() : ${…published-only…}.first()"`.
- Actor model: `:username` matches **any authenticated user**; `actor` also
  matches OIDC roles, which include the user's email and group claims. End-user
  login is the per-host `OIDCAuthentication` feature (configurable
  `login-path`/`logout-path`/`callback`). A `pk_` key is **not** an OIDC end-user
  session — you can't reach an authenticated-only composed page with it; only the
  user, in a browser, can verify those paths.

### Triggers vs Processors (Sessel automation)

- **Trigger** fires **before** core processing — `Context.request` only (method,
  path, headers, query, body); **no decoded auth claims**. Can throw an
  `HTTPResponse` to short-circuit.
- **Processor** fires **after** — `Context.request` **and** `Context.response`
  (read/set `.status`, `.body`, `.headers`). Use a processor to turn a soft-200
  "not found" composition into a **real 404** (match `status: 200`, then
  `when` checks the body lacks your content marker, action sets `status = 404`).

### Closed ShapeConstraint = validate untrusted writes

To stop arbitrary HTML on a public write path (e.g. comments), add a **closed**
`ShapeConstraint` (any `permit` makes it closed → only declared elements/attrs
allowed, else `422`):

- Tie permits to **elements**, not bare attributes: `strong[itemprop="author"]`,
  not `[itemprop="author"]` (else `<script itemprop="author">` slips through).
- Use **exact** class matching `[class="comment"]` (not `.comment`) so extra
  classes can't ride along. Omit any `style`/`<style>`/`<link>` permit → no
  styling can be injected.
- The constraint fires when the **request target** matches its `selector` (the
  element the `Range` selector hits). A whole-document admin `PUT` targets
  `:root`, so it does **not** fire — only the granular write path is policed.
- Cover the matched root element's own attributes via the `selector`
  (`[itemprop="comments"][id][class]`).

### Sessel-in-HTML & composition gotchas

- Raw `<`/`>` (and sometimes `&&`) inside inline `<script type="text/sessel">`
  can break the renderer. Avoid them: prefer declarative filters (e.g. a
  Processor's `status` meta) over a Sessel `&&`; use `== false` over `!`.
- `e:`/`r:` binding attributes are **stripped** from composed output — handy
  marker that composition ran.

### Liquid (this engine)

- The `date` filter throws "unable to parse date input" — **don't use it**. Store
  human date strings in the data (`displayDate`, `monthLabel`) and bind those;
  sort by the ISO field (string sort = chronological).
- Output must be **balanced** — you can't open a tag in one iteration/`{% if %}`
  and close it in another. Emit self-contained fragments.
- `{{ x | where: "status", "Published" }}` works for filtering bound collections.

### Edge cache & verifying

Composed pages are CDN-cached `cache-control: public, max-age=5`. When verifying a
just-deployed change, append a unique `?cb=<n>` query string (distinct cache key)
to read fresh.

### Don't clobber live data when testing writes

Write-tests mutate real data. **Snapshot the live origin via WebDAV first**
(`curl "${WEBDAV}data/…" -H "$KEY" > /tmp/before.html`), run the test, then PUT
that snapshot back. Never restore from your local seed — it's lossy if the live
data has diverged (e.g. real comments added since).

## Common mistakes

| Mistake | Correct |
| --- | --- |
| Reading host data from `/` | Use `/console/index.html` (`/` is the landing page) |
| Reading hosts without a selector | `Range: selector=[itemtype="urn:Host"]` (→ 206) |
| Editing host config at `/organizations/<org>/<hid>.html` | Edit the inlined block on `/console/index.html` |
| Mounting the host / `cp` to deploy | Whole-file `PUT`/`MKCOL` to the `webdav-url` (`:8081`) |
| Element-level `:80` PUT for file authoring | Whole-file WebDAV `PUT` (`:8081`) preserves formatting |
| Writing the API key to a file or commit | Session-only; never persist or echo |
| Writing markup from memory | Fetch `DOCS_URL` for the feature first |
| Mutating a shared host without asking | Confirm with the user before any write |
| POSTing a write to the raw data path | POST to the **stamped/composed** resource; authz is on the composed page, not the origin |
| Adding an origin write rule for write-through | Origin needs none — rule on the composed page path is necessary & sufficient |
| `auth.claims.email` in a binding (errors) | Use `request.auth.claims.email` / `request.auth.username` |
| Branching composition on login via a Trigger | Triggers have no claims; branch in a binding with `request.auth.*`, or set status in a Processor |
| Expecting a real 404 from composition | Composition can't set status; flip it in a **Processor** |
| Trusting a public write path to be well-formed | Add a **closed** `ShapeConstraint` (element-tied, exact-class permits) → 422 on junk |
| Restoring live data from a local seed after a test | Snapshot the live origin via WebDAV first, restore that |
| Verifying a deploy and seeing stale HTML | Edge cache `max-age=5`; append `?cb=<n>` to bust |
| Assuming a `pk_` key can view authed-only pages | It's not an OIDC session; only the user (browser) can verify login paths |
