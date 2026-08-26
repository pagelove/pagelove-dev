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
| Deriving an authoring URL from the public hostname | Use the selected host's exact advertised `webdav-url` |
| Using element-level public writes for file authoring | Use whole-file WebDAV `PUT`/`MKCOL` operations |
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
