---
name: building-pagelove-apps
description: "Use when building, creating, or modifying a Pagelove application, or when the working directory is a WebDAV mount for a pagelove.cloud site. Also use when another skill identifies Pagelove as the target platform."
---

# Building Pagelove Applications

## Overview

Pagelove is a web platform where HTML documents ARE the application. There is no separate backend — HTTP verbs operate directly on DOM elements via CSS selectors, and the platform handles concurrency, authorization, and validation.

**Before writing any Pagelove-specific markup, fetch the relevant spec from `https://docs.pagelove.org/`.**

## Core Mental Model

- A **resource** is a file (HTML page)
- HTTP methods target **elements within a page** using `Range: selector=<css>` headers
- **AuthorizationRule** microdata controls who can do what to which elements
- **ShapeConstraint** microdata validates DOM structure on mutations
- **SSPI** (includes, templating, resource binding) runs server-side before delivery
- **Pagelove Primitives** is a JS library that attaches HTTP methods to DOM elements

## Documentation Map

Fetch these pages from `https://docs.pagelove.org/` as needed:

| Need | Fetch |
|------|-------|
| HTTP method details | `/http/GET-method/`, `/http/PUT-method/`, `/http/POST-method/`, `/http/DELETE-method/`, `/http/OPTIONS-method/` |
| Authorization rules | `/schema/AuthorizationRule/` |
| Shape constraints | `/schema/ShapeConstraint/` |
| Group membership | `/schema/GroupMembership/` |
| Server-side includes | `/sspi/Includes/` |
| Resource binding | `/sspi/Resource-Binding/` |
| Templating (Liquid) | `/sspi/Templating/` |
| Resource creation | `/sspi/Resource-Creation/` |
| JS Primitives API | `/JavaScript/` |
| WebDAV access | `/WebDAV/` |

**REQUIRED: Fetch the relevant doc pages before writing markup for that feature.**

## Default Application Structure

New apps start with two files:

```
/index.html            — application page
/admin/
  auth.html            — AuthorizationRule + ShapeConstraint microdata
```

- `/admin/auth.html` is inaccessible by default (deny-by-default — no auth rule grants access to it)
- AuthorizationRule elements are discovered wherever they exist across the site — they are not limited to `auth.html` or any specific file. The platform finds and evaluates all AuthorizationRule microdata regardless of which page contains it. The `admin/auth.html` convention is just organizational convenience.
- SSPI features (includes, templating, resource binding) are added only when needed
- Additional pages go at whatever paths make sense for the app

## Implementation Workflow

Follow this order — auth rules must exist before mutations work:

1. **Fetch docs** for features being used
2. **Scaffold or assess** — create structure if empty, read existing files if not
3. **Build application HTML** — semantic markup with schema.org microdata, IDs on containers
4. **Define authorization rules** — deny-by-default, allow only what's needed
5. **Define shape constraints** — for any elements accepting POSTed content
6. **Add client-side JS** — Pagelove Primitives for interactivity
7. **Test via HTTP** — curl against the live site after each step

## Critical Reference: Correct Syntax

These are the most common errors. Get these right:

### Schema URLs (NOT pagelove.cloud)
```
https://pagelove.org/AuthorizationRule
https://pagelove.org/ShapeConstraint
https://pagelove.org/GroupMembership
```

### AuthorizationRule Properties
Properties are: `actor`, `resource`, `method`, `selector`, `action` (allow/deny).
Multiple methods use separate elements, NOT comma-separated:
```html
<tr itemscope itemtype="https://pagelove.org/AuthorizationRule">
    <td itemprop="actor">*</td>
    <td itemprop="resource">/index.html</td>
    <td>
        <ul>
            <li itemprop="method">PUT</li>
            <li itemprop="method">DELETE</li>
        </ul>
    </td>
    <td itemprop="selector">li</td>
    <td itemprop="action">allow</td>
</tr>
```

### Liquid Interpolation in AuthorizationRules
All AuthorizationRule properties support Liquid template expressions, evaluated per-request against the request context. This requires no SSPI namespace declaration — interpolation is implicit on AuthorizationRule elements. Liquid filters work normally within these expressions.

This enables dynamic authorization patterns — scoping rules by resource path, element ownership, or any combination. For example, this rule restricts staff members to their own page and their own records:

```html
<tr itemscope itemtype="https://pagelove.org/AuthorizationRule">
    <td itemprop="actor">staff</td>
    <td itemprop="resource">/team/{{request.auth.claims.name | split: " " | first | downcase }}.html</td>
    <td><ul>
        <li itemprop="method">GET</li>
        <li itemprop="method">POST</li>
        <li itemprop="method">PUT</li>
    </ul></td>
    <td itemprop="selector">[id][itemtype="http://pagelove.com/TeamMember"]:has([itemprop=owner][content='{{request.auth.claims.email}}']) [itemprop]</td>
    <td itemprop="action">allow</td>
</tr>
```

Here the resource is templated to resolve to a per-user page (e.g., a staff member named "Jane Smith" can only access `/team/jane.html`), and the selector further restricts mutations to elements where the owner itemprop matches their email. Both expressions are expanded at evaluation time per request.

For ownership patterns to hold, POSTed elements must include the ownership property — enforce this with a ShapeConstraint:

```html
<div hidden itemscope itemtype="https://pagelove.org/ShapeConstraint">
    <span itemprop="selector">#team [itemtype="http://pagelove.com/TeamMember"]</span>
    <span itemprop="constraint">:has([itemprop=owner])</span>
</div>
```

### ShapeConstraint Properties
Properties are: `resource` (optional), `selector`, `constraint` (repeatable):
```html
<div hidden itemscope itemtype="https://pagelove.org/ShapeConstraint">
    <span itemprop="selector">#my-list li</span>
    <span itemprop="constraint">:has([itemprop=name])</span>
    <span itemprop="constraint">:has([itemprop=email])</span>
</div>
```

### Pagelove Primitives JS
Two separate modules. PLDocument is instantiated, not static:
```javascript
import { PLDocument } from "https://cdn.pagelove.net/js/pagelove-primitives/1a5a161/index.mjs";
import { DOMSubscriber } from "https://cdn.pagelove.net/js/dom-subscriber/cde4007/index.mjs";

const doc = new PLDocument();
doc.OPTIONS();
```

**`doc.OPTIONS()` discovers HTTP capabilities and attaches `.PUT()`, `.POST()`, `.DELETE()` to DOM elements. It does NOT provide user/auth information.** Do not attempt to read `options.user` or similar — there is no such property. For auth-aware UI, use Liquid templates instead (see below).

### Auth-Aware UI: Use Liquid Templates, Not JS
PLDocument.OPTIONS() does not expose the authenticated user. To show/hide UI based on auth state, use Liquid templates with `pagelove:template="text/liquid"`:
```html
<div xmlns:pagelove="https://pagelove.org/1.0" pagelove:template="text/liquid">
{% if request.auth.username %}
    <span>{{ request.auth.claims.name }}</span> | <a href="/auth/logout">Logout</a>
{% else %}
    <a href="/auth/login">Login</a>
{% endif %}
</div>
```
Available Liquid variables for auth:
- `request.auth.username` — truthy when authenticated
- `request.auth.claims.name` — user's display name
- `request.auth.claims.email` — user's email address
- `request.auth.roles` — user's roles/groups

### DOMSubscriber API
Subscribes to elements matching a selector. Callback receives the element:
```javascript
DOMSubscriber.subscribe(document, 'button.delete', (button) => {
    button.addEventListener('click', () => {
        button.closest('li').DELETE();
    });
});
```

### SSPI Namespace
Documents using SSPI features must declare the namespace:
```html
<html lang="en" xmlns:pagelove="https://pagelove.org/1.0">
```

## Canonical Example: Todo List

Source: `https://todo-list.pagelove.cloud/`

### index.html (application page)
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta content="width=device-width, initial-scale=1.0" name="viewport">
    <title>Pagelove Todo List Example</title>
    <style>
        li:has(> input:checked) { text-decoration: line-through; }
    </style>
</head>
<body>
    <main>
        <h1>Pagelove Todo Example</h1>
        <p>This todo list application is running on Pagelove.</p>
        <h2>Todo List</h2>
        <ul id="todo-list">
            <li itemscope itemtype="http://schema.org/ListItem">
                <input type="checkbox">
                <span itemprop="name">Build something great with Pagelove</span>
                <button class="delete">🗑</button>
            </li>
        </ul>
        <form>
            <input type="text" name="item" placeholder="New item...">
            <button>Add</button>
        </form>
    </main>

    <script type="module">
        import { PLDocument } from "https://cdn.pagelove.net/js/pagelove-primitives/1a5a161/index.mjs";
        import { DOMSubscriber } from "https://cdn.pagelove.net/js/dom-subscriber/cde4007/index.mjs";

        document.addEventListener('change', (e) => {
            if (e.target.checked) {
                e.target.setAttribute('checked', 'true');
            } else {
                e.target.removeAttribute('checked');
            }
            if (e.target.parentNode.PUT) e.target.parentNode.PUT();
        });

        document.querySelector('form').addEventListener('submit', (e) => {
            e.preventDefault();
            const itemText = e.target.item.value.trim();
            if (itemText.length === 0) return;
            const li = document.createElement('li');
            li.setAttribute('itemscope', '');
            li.setAttribute('itemtype', 'http://schema.org/ListItem');
            li.innerHTML = `<input type="checkbox"> <span itemprop="name">${itemText}</span> <button class='delete'>&#128465;</button>`;
            document.getElementById('todo-list').POST(li);
            e.target.reset();
        });

        DOMSubscriber.subscribe(document, 'button.delete', (e) => {
            e.addEventListener('click', (e) => {
                const li = e.target.closest('li');
                li.DELETE();
            });
        });

        const doc = new PLDocument();
        doc.OPTIONS();
    </script>
</body>
</html>
```

### admin/auth.html (authorization + constraints)
```html
<!DOCTYPE html>
<html lang="en" xmlns:pagelove="https://pagelove.org/1.0">
<head>
    <meta charset="UTF-8">
    <title>Authorization Rules</title>
</head>
<body>
    <main>
        <h1>Authorization Rules</h1>
        <table>
            <thead>
                <tr>
                    <th>Actor</th><th>Resource</th><th>Method</th><th>Selector</th><th>Action</th>
                </tr>
            </thead>
            <tbody>
                <tr itemscope itemtype="https://pagelove.org/AuthorizationRule">
                    <td itemprop="actor">*</td>
                    <td itemprop="resource">/index.html</td>
                    <td><ul><li itemprop="method">GET</li></ul></td>
                    <td itemprop="selector"></td>
                    <td itemprop="action">allow</td>
                </tr>
                <tr itemscope itemtype="https://pagelove.org/AuthorizationRule">
                    <td itemprop="actor">*</td>
                    <td itemprop="resource">/index.html</td>
                    <td><ul>
                        <li itemprop="method">PUT</li>
                        <li itemprop="method">DELETE</li>
                    </ul></td>
                    <td itemprop="selector">li, input</td>
                    <td itemprop="action">allow</td>
                </tr>
                <tr itemscope itemtype="https://pagelove.org/AuthorizationRule">
                    <td itemprop="actor">*</td>
                    <td itemprop="resource">/index.html</td>
                    <td><ul><li itemprop="method">POST</li></ul></td>
                    <td itemprop="selector">ul</td>
                    <td itemprop="action">allow</td>
                </tr>
            </tbody>
        </table>

        <div hidden itemscope itemtype="https://pagelove.org/ShapeConstraint">
            <span itemprop="selector">#todo-list li</span>
            <span itemprop="constraint">:has(input[type=checkbox])</span>
            <span itemprop="constraint">:has([itemprop=name])</span>
            <span itemprop="constraint">:has(button.delete)</span>
        </div>
    </main>
</body>
</html>
```

### Why It Works This Way
- **`ul#todo-list`**: ID makes it a stable POST target
- **`li, input` for PUT**: checkbox changes AND item updates both need PUT
- **ShapeConstraint on `#todo-list li`**: prevents malformed items from being POSTed
- **`DOMSubscriber`**: handles delete buttons on elements that don't exist yet (POSTed later)
- **`doc.OPTIONS()` on load**: discovers capabilities, attaches PUT/POST/DELETE to matching elements
- **`if (e.target.parentNode.PUT)`**: guard — only call PUT if capability was discovered

## Important Behavior Notes

### PUT sends full outerHTML
`element.PUT()` sends the element's entire `outerHTML` to the server. Any hidden UI elements (edit forms, temporary state) inside the element WILL be included. Design elements so that their resting DOM state is clean — avoid nesting hidden editing UI inside data elements that will be PUT.

### Shape constraints enforce data integrity, not UI structure
Constrain what matters for data correctness: `itemprop` attributes, required semantic children. Avoid constraining CSS classes or UI-only elements — those are presentation concerns that should be free to change.

### Resource Binding namespace requires `/1.0/`
The Resource namespace MUST include `/1.0/`:
```html
<html xmlns:resource="https://pagelove.org/1.0/Resource">
```
NOT `https://pagelove.org/Resource` (attributes won't be stripped and binding won't work).

### Liquid templates and HTML `<table>` are incompatible
HTML foster parenting moves non-table content (like Liquid `{% %}` tags) outside `<table>/<tbody>` before Liquid processes them, causing "Unknown variable" errors. **Use CSS grid with `<div>` elements instead of `<table>` for any template-driven listings.**

### Wrap JavaScript in `{% raw %}` when body uses Liquid
When `<body>` has `pagelove:template="text/liquid"`, Liquid processes the ENTIRE body including `<script>` tags. Template literals (`${}`) and any `{{ }}` in JS will be mangled. Wrap scripts:
```html
{% raw %}
<script type="module">
    // JS with template literals is safe here
</script>
{% endraw %}
```

### Resource creation: form and template are separate pages
The form page is a plain HTML file that POSTs to the template. The template uses `<base href>` to determine the new page's URL. A successful POST returns 301 redirect to the new resource. Authorization needs BOTH:
- POST on the template path (e.g., `/templates/new-bug.html`)
- PUT on the target path (e.g., `/bugs/*`) without selector (for the resource creation write)

### Template request object structure
In Liquid templates for resource creation, the request object provides:
- `request.body.<field>` — form field values
- `request.auth.claims.name` — authenticated user's display name
- `request.auth.roles` — user's roles/groups
- `request.method`, `request.path`, `request.headers` — HTTP metadata
Note: there is NO `request.actor` property.

### The `users` group
All authenticated users automatically belong to the `users` group. Use `users` as the actor value in AuthorizationRule for "any authenticated user" rules.

### Resource binding picks up template files
If a template file contains microdata (e.g., `itemtype="http://schema.org/Report"`), resource binding will include it with raw Liquid tags as property values. Filter by checking the `@id` for template-specific strings — do NOT use `contains '{{' ` (breaks Liquid parser) or `{% if item.name %}` (unprocessed Liquid tags are truthy strings):
```liquid
{% for item in items %}
{% unless item['@id'] contains 'request.body' %}
    <!-- render item -->
{% endunless %}
{% endfor %}
```

### Writing files to WebDAV mounts
The Write tool may truncate files on WebDAV mounts. Use bash heredocs instead:
```bash
cat > /path/to/webdav/file.html << 'HTMLEOF'
<!DOCTYPE html>
...
HTMLEOF
```

## Testing with curl

After each implementation step, verify with HTTP requests:

```bash
# Check capabilities
curl -si https://myapp.pagelove.cloud/ -X OPTIONS \
  -H "Accept: multipart/mixed" -H "Prefer: return=representation"

# GET an element
curl -si https://myapp.pagelove.cloud/ -H "Range: selector=#my-list"

# POST a new element
curl -si https://myapp.pagelove.cloud/ -X POST \
  -H "Range: selector=#my-list" \
  -H "Content-Type: text/html" \
  -d '<li itemscope itemtype="http://schema.org/ListItem">...</li>'

# PUT (replace) an element
curl -si https://myapp.pagelove.cloud/ -X PUT \
  -H "Range: selector=li:first-child" \
  -H "Content-Type: text/html" \
  -d '<li itemscope itemtype="http://schema.org/ListItem">...</li>'

# DELETE an element
curl -si https://myapp.pagelove.cloud/ -X DELETE \
  -H "Range: selector=li:last-child"

# Test shape constraint violation
curl -si https://myapp.pagelove.cloud/ -X POST \
  -H "Range: selector=#my-list" \
  -H "Content-Type: text/html" \
  -d '<li>malformed</li>'
# Expected: 422 Unprocessable Content
```

## HTTP Response Codes

| Code | Meaning |
|------|---------|
| 200 | OK (OPTIONS) |
| 204 | No Content (successful DELETE) |
| 206 | Partial Content (successful GET/PUT/POST with selector) |
| 207 | Multi-Status (multipart OPTIONS) |
| 404 | Resource not found |
| 416 | Invalid CSS selector |
| 422 | Shape constraint violated |

## Common Mistakes

| Mistake | Correct |
|---------|---------|
| Schema URL `pagelove.cloud` | `https://pagelove.org/AuthorizationRule` |
| Property `principal`/`verb`/`effect` | `actor`/`method`/`action` |
| `PLDocument.OPTIONS()` (static) | `new PLDocument(); doc.OPTIONS()` (instance) |
| Comma-separated methods `"PUT,DELETE"` | Separate `<li itemprop="method">` per method |
| `<meta itemprop>` for auth rules | Use visible elements (`<td>`, `<span>`) with `itemprop` |
| CDN at `pagelove.cloud` | `https://cdn.pagelove.net/js/pagelove-primitives/.../index.mjs` |
| DOMSubscriber in pagelove-primitives | Separate import from `cdn.pagelove.net/js/dom-subscriber/.../index.mjs` |
| Missing SSPI namespace | `xmlns:pagelove="https://pagelove.org/1.0"` on `<html>` when using SSPI |
| Resource namespace `pagelove.org/Resource` | Must be `https://pagelove.org/1.0/Resource` (with `/1.0/`) |
| Liquid tags inside `<table>` | HTML foster parenting breaks them — use CSS grid `<div>` instead |
| JS template literals in Liquid body | Wrap `<script>` in `{% raw %}...{% endraw %}` |
| `request.actor` in templates | Use `request.auth.claims.name` for user identity |
| Using Write tool on WebDAV mount | Files may truncate — use bash heredocs instead |
| `doc.OPTIONS()` for user info | OPTIONS doesn't provide auth info — use Liquid templates for auth-aware UI |
