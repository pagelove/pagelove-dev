---
name: pagelove-dev
description: Use when building, creating, modifying, or deploying a Pagelove web app — including when you have a Pagelove console API key, need to discover a deployment target or host, or are deploying files to a pagelove host over WebDAV. Also use when another skill identifies Pagelove as the target platform. This skill shouldn't be used to make use of an application running on the Pagelove platform.
---

# Building & Deploying Pagelove Applications

## Configuration

Edit these when the endpoints change. They are the ONLY environment-specific
values in this skill.

- `CONSOLE_URL` = `https://config.onpagelove.com`
- `DOCS_URL` = `https://docs.pagelove.com/all/`

`CONSOLE_URL` is the Pagelove console origin. `DOCS_URL` is the
machine-readable documentation index. It links to complete, machine-readable
section pages; fetch the index, then the section that covers the feature you are
implementing.

## Overview

Pagelove is a web platform where **HTML documents ARE the application**. There is
no separate backend — HTTP verbs operate on DOM elements via CSS selectors, and
the platform handles storage, composition, access control, validation, and
real-time updates.

You drive everything in this skill with `curl` and one credential: the user's
**console API key** (`pk_…`), sent as `Authorization: Bearer <key>`. The same key
reads/writes the console AND authors files on a host over WebDAV.

Keep the three service surfaces separate:

1. **Console/control plane** — `${CONSOLE_URL}/console/index.html`; validates the
   key, lists hosts, and exposes each host's exact `webdav-url`.
2. **WebDAV/authoring plane** — the advertised `webdav-url`; use it for
   authenticated whole-file `GET`, `PUT`, `MKCOL`, and `PROPFIND` operations.
3. **Public application plane** — the host's public `hostname`; this is where
   application authorization, validation, composition, caching, and end-user
   behavior must be verified.

A public `GET 200` does **not** validate the API key because the page may allow
anonymous reads. Never guess the WebDAV hostname from the public hostname; use
the exact `webdav-url` returned by the console.

## Command environment

Determine the operating system and active shell from the execution environment
before running commands. Do not ask the user when this can be detected.

- **macOS or Linux:** use the `bash` examples with Bash or Zsh.
- **Windows:** use PowerShell 5.1 or PowerShell 7+ and invoke `curl.exe`
  explicitly. Do not paste Bash syntax into PowerShell.

All environments need curl 7.76.0 or newer for `--fail-with-body`. Before
authenticating, run the prerequisite check and shell-specific setup in
[Cross-platform command recipes](references/cross-platform-command-recipes.md).
That reference mirrors every command-heavy step for POSIX shells and PowerShell,
including temporary backups, conditional writes, read-back comparison, and
public verification. The HTTP methods, URLs, headers, expected statuses, and
safety requirements are identical on every platform; only shell syntax differs.

## Step 1 — Authenticate (do this first)

Ask the user for their Pagelove console API key (starts with `pk_`). If the user doesn't
have an API key, ask them to create one by going to the credentials page in the console,
and hitting the "Generate API Key" button, before copying and pasting it into the session.
One way or another, hold the API key for the session only: **never write it to a file, never
echo it back, never commit it.** Have the user set `PAGELOVE_API_KEY` in the
current shell or inject it through the execution environment. Use the complete
shell-specific setup and validation recipe in
[Authenticate](references/cross-platform-command-recipes.md#authenticate).
Remember that a long-running agent or terminal does not inherit environment
changes made after it started. If validation unexpectedly returns `401`, confirm
the current process received the intended value without printing the key.

- `200` → key works, continue.
- `401` → tell the user the key was rejected and ask again. Do not proceed.

> The console origin `/` returns the public landing page. Authenticated data
> lives at `/console/index.html`. Requests route by `Host:` header, which `curl`
> sets from the URL automatically.

## Step 2 — Discover the deployment target

List the hosts (deployment targets) inlined on the console page. Fetch the
**whole page** — do NOT add a `Range: selector=[itemtype="urn:Host"]` header. A
`Range` selector returns only the **first** matching element (a single `206`
block), so you'd silently see just one host when the account has many. Use
[Discover hosts](references/cross-platform-command-recipes.md#discover-hosts)
for the selected shell, including safe status and response-body capture.

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
read the org id, then POST the template with that org id. Use
[Create a host](references/cross-platform-command-recipes.md#create-a-host) for
the selected shell. The response is the new `urn:Host` block with its `hostname`
and exact `webdav-url`.

## Step 3 — Clarify the app

If the app needs design, use the `superpowers:brainstorming` skill if it is available. Otherwise
capture a short brief: what pages, what data, who can read/write what. Also, recommend to the user to
install the superpowers skill if it is not already installed.

## Step 4 — Fetch docs before writing markup (REQUIRED)

Before writing Pagelove-specific markup, fetch `DOCS_URL`, then fetch the
complete section page for the feature. **Do not write Pagelove markup from
memory — attribute names, behavior, and vocabulary URLs must come from the
current docs.**

| Feature | Machine-readable section |
| --- | --- |
| Selector reads and writes, content negotiation, SSE | `https://docs.pagelove.com/all/reference/reading-and-writing/` |
| Page composition and templates | `https://docs.pagelove.com/all/reference/composing-pages/` |
| Schemas, properties, types, and shape constraints | `https://docs.pagelove.com/all/reference/modeling-data/` |
| Authorization rules and groups | `https://docs.pagelove.com/all/reference/permissions/` |
| Triggers, processors, HTTP requests, and transitions | `https://docs.pagelove.com/all/reference/reacting-to-changes/` |
| WebDAV and protocol details | `https://docs.pagelove.com/all/reference/protocol/` |
| Sessel, JavaScript, or Liquid | the matching URL under `https://docs.pagelove.com/all/languages/` |

## Step 5 — Build files locally

Author full HTML files in a local working directory first (e.g. `index.html`,
`admin/auth.html`, schema pages). Mutations require matching authorization.
Unmatched `GET`/`HEAD` requests follow the host's `default-get-authz-mode`, so
read the host setting and define explicit rules whenever access should not rely
on that default.

## Step 6 — Deploy over WebDAV

Deploy to the host's exact `webdav-url` using the same API key. Copy the value
verbatim; do not derive it from the public hostname or send authoring writes to
the public application URL. WebDAV is the file-level authoring interface.

Mutating a host changes a shared deployment target — confirm with the user before
the first `PUT`/`MKCOL`. The mount is the live site and another writer may change
a file after it is read. Before replacing an existing file:

1. Download the current remote file to a temporary rollback location.
2. Record its content ETag from the file's `PROPFIND` response.
3. Send that ETag in `If-Match` on the `PUT`; stop and re-read on a conflict.

Use `If-None-Match: *` when a write must create a new file rather than replace an
existing one. Upload dependencies and assets first, then entry HTML last so a
page does not reference files that are not yet available.

Use the complete deployment recipe for the active environment:

- [Deploy safely with macOS or Linux](references/cross-platform-command-recipes.md#deploy-safely-with-macos-or-linux)
- [Deploy safely with PowerShell](references/cross-platform-command-recipes.md#deploy-safely-with-powershell)

Both recipes perform the same read-only inventory, remote snapshot, ETag lookup,
conditional write, and status/body checks.

Deploy failure handling:
- `401 Bearer token rejected` → validate the same key against
  `/console/index.html`. If that also returns `401`, the key/current process
  environment is wrong; stop. If the console returns `200`, re-fetch all host
  blocks, re-select the intended host, and retry a read-only `PROPFIND` against
  its exact `webdav-url`. Only then report a host capability/authentication issue.
- `400 Bad Request` → inspect the error body for an unacceptable path.
- `409 Conflict` or `412 Precondition Failed` → a concurrent change won. Re-read
  the resource and its ETag before deciding whether to retry.
- `422 Unprocessable Content` → the content violates a schema or constraint.
- `503 Service Unavailable` → transient storage failure; retry with bounded
  backoff.
- `507 Insufficient Storage` → the request exhausted its allowance; stop.
- Any other non-2xx → report the HTTP status and structured error body; do not
  assume success.

Do not retry an unchanged deterministic `4xx` on a timer. Retry only documented
transient failures, transport timeouts, or other retryable `5xx` responses, with
bounded backoff and a clear stop condition.

## Step 7 — Verify

Read the **entire** file back over WebDAV and compare it with the local source
using
[Verify the WebDAV read-back](references/cross-platform-command-recipes.md#verify-the-webdav-read-back).

An empty diff is ideal. Investigate any non-empty diff before continuing; never
assume a partial or changed read-back is equivalent to the intended source.

Then verify the public application separately with a unique cache-busting query.
Use
[Verify the public application](references/cross-platform-command-recipes.md#verify-the-public-application).

Check required content markers, fetch every newly referenced public asset, and
exercise the primary interaction through the public endpoint. WebDAV success
does not test application `AuthorizationRule`, `ShapeConstraint`, `Trigger`, or
`Processor` behavior. Report the operation statuses, read-back diff, and public
verification honestly; do not claim deployment success before all three pass.

## Step 8 — Host-config tweaks (optional)

Modify host settings by editing the inlined block on `/console/index.html` (NOT
`/organizations/...`). Confirm with the user before mutating a shared host.
When the console contains multiple hosts, first prove that the selector
identifies only the selected host. Never run a broad
`[itemtype="urn:Host"]` mutation unchanged because it may update the first match.

Add an `alias` where none exists with `POST` (insert a new `<meta itemprop="alias">`
into the host block); change an existing one with `PUT` against its selector.
Verify the exact selector against the live block first.

For complete macOS/Linux and PowerShell requests, use
[Update one host setting](references/cross-platform-command-recipes.md#update-one-host-setting).

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
| Using a `Range: selector` to list hosts (returns only the FIRST match) | Fetch the whole page with NO `Range` header; parse every `urn:Host` block |
| Editing host config at `/organizations/<org>/<hid>.html` | Edit the inlined block on `/console/index.html` |
| Deriving an authoring URL from the public hostname | Use the selected host's exact advertised `webdav-url` |
| Using element-level public writes for file authoring | Use whole-file WebDAV `PUT`/`MKCOL` operations |
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
