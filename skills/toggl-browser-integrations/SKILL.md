---
name: toggl-browser-integrations
description: Author, post, and verify a Toggl 2.0 custom website integration (the in-page Toggl 2.0 "Track" button) — building the integration JSON, finding anchor/resolver CSS selectors, injecting it on a live page, and checking the button landed in the right spot. Use whenever the user wants the Toggl 2.0 button to show on a new website, asks to build/edit/debug an integration definition, mentions anchors/resolvers/selectors for the Toggl 2.0 button, or says a tracked title or metadata reads wrong — even if they never use the word "integration". Drives the extension's side-panel JSON editor via a shared-DOM bridge (write the definition to a `data-toggl-focus-post` attribute on the page), which loads the definition and applies it locally so the button renders for inspection.
---

# Toggl 2.0 — Custom Website Integrations

A Toggl 2.0 *integration* injects the in-page Toggl 2.0 "Track" button into a
third-party website (GitHub, Jira, Asana, …) and teaches it how to read a task
description + metadata from that page's DOM. An integration is a single JSON
document. This skill is how
you author one against a live page, push it into the extension's side panel without
typing, see the button render, and iterate on selector/placement until it's right —
then the user **Saves** it to their organization.

You need only the **Toggl 2.0 browser extension installed and signed in** — no source
code or repo access. Everything about the JSON format is documented in §3 below; this
skill is self-contained.

---

## 1. Setup & how we work together

This is a **collaboration against a live extension**, not source editing. Some steps
only you (Claude) can do, some only the user can — get the handshake right before
building anything. You drive the DOM and the post/verify loop; the user drives the
browser state and the side panel.

**One-time environment** (the user sets this up):

- The **Toggl 2.0 extension** is installed in their browser (Chrome).
- The user is **signed in to Toggl 2.0**, with a workspace/organization available. (The
  button and its resolvers talk to Toggl 2.0, so an unauthenticated session renders a
  broken/empty button.)
- **Recommended: run with only Claude-in-Chrome and Toggl 2.0 enabled — disable other
  extensions for the session.** Other extensions can silently intercept the actions
  Claude drives through the MCP. A concrete case seen in the wild: **1Password** hooks
  page inputs/clicks and can block Claude from completing some actions (e.g. swallowing
  a click or grabbing focus). If `javascript_tool` actions, clicks, or the post bridge
  behave erratically and the §6 tab checks are all clean, suspect a third-party
  extension and ask the user to disable everything except Claude + Toggl, then retry.

**Per session — split the work:**

| Step | Who | How |
|---|---|---|
| Open a tab on the integration's **domain** | **You** | Chrome MCP `navigate` / `tabs_create_mcp` (e.g. `https://app.acme.com`). **Then confirm it's the tab the user sees** — inject a banner + check `document.visibilityState` (§6); the MCP may have opened a background window. |
| Navigate to the **exact page/item** to integrate (open the issue, the board card, the list view) | **User** | They know the app, the auth, and the precise state — ask them to land on it. |
| Open the Toggl 2.0 **side panel on that tab** | **User** | Extension action icon → context menu → **"Open side panel"**. This is a user gesture on browser chrome — **you can't do it**. The panel must be on the *same* tab you post to (the bridge is per-tab). |
| Pick a **workspace/organization** in the panel header ("Saving to") | **User** | Local apply needs an org, else a *"Pick an organization"* toast and nothing injects. |
| Confirm the page, probe the DOM, post + verify, iterate | **You** | §2, §4, §5, via Chrome MCP `javascript_tool` (inspect + post) and a screenshot to look — prefer the **Chrome MCP** `screenshot`/`zoom` for the page (no extra grant); use the desktop `computer` screenshot only to see the side panel (§5-B). |

> No Claude-in-Chrome MCP — or it's wedged (§6)? The flow still works: do the DOM/selector
> reasoning from the page HTML the user shares, hand them the `setAttribute(...)` post
> snippet (§2) plus a one-line verify snippet (§5) to paste into the page's DevTools
> console, and ask for a screenshot. The post behaves the same in the console as from
> `javascript_tool` (it's a plain DOM write), so when `javascript_tool` starts erroring,
> switch the user to the console rather than looping on the tool.

---

## 2. The post bridge (how a definition reaches the panel)

The side panel is a separate extension document, so the page can't talk to it directly.
The extension ships a content script (on every page) that relays a definition you write
to a **shared-DOM attribute** into the panel:

```
data-toggl-focus-post on <html>  →  extension content script  →  side panel
```

> **The whole bridge only works in authoring mode** — i.e. while the **custom-integration
> side-panel builder is open** on the tab. That open builder *is* authoring mode; it's the
> user's deliberate opt-in to this flow. With it closed, a posted definition is ignored
> (nothing applies or injects), the current-integration read (below) answers
> `gated: true`, and no `data-toggl-debug-*` attributes are emitted. So if "nothing
> happens," the first thing to confirm is that the user has the builder open on the exact
> tab you're driving.

**To post a definition, write the JSON to `data-toggl-focus-post` on `<html>`:**

```js
document.documentElement.setAttribute(
  'data-toggl-focus-post',
  JSON.stringify(integrationDefinition),
);
```

A DOM write is visible to **every execution world**, so this works **identically** from
the Chrome MCP `javascript_tool` (which runs in an isolated world) and from the DevTools
console — no main-world tricks, no `<script>` injection, no event dispatch. The content
script reads the attribute, relays it to the panel, then **removes** it — and that removal
doubles as a reachability check:

```js
// after posting, check whether the content script consumed it:
document.documentElement.hasAttribute('data-toggl-focus-post')
// false → consumed: the content script IS present on this tab and read the post
// true  → not consumed: wrong/hidden tab, or no content script on this page (§6)
```

> The attribute value must be a JSON **string** (`setAttribute` coerces to string, so
> always `JSON.stringify(...)` the definition). And note: *consumed ≠ applied* — removal
> proves the content script read it, but the **button only injects if the side panel is
> open on this tab with an org selected** (§5-A walks the failure ladder).

What happens next, automatically:
1. The side panel switches to its **Edit as JSON** view and shows the document.
2. It parses the JSON into form state (invalid JSON → inline error, *no* apply).
3. **It applies the definition locally** — this injects the button on the active tab
   so you can see it immediately, without persisting anything yet. A *"Applied
   locally"* toast confirms it. (You do **not** click "Apply locally" yourself; the
   post does it.)

Local apply is a draft: it's stored locally and injected, but **not** synced to the
backend. When the definition is correct, the user clicks **Save** in the side panel to
sync it to their organization. Do not click Save for them unless asked — local apply
is enough for authoring and verification.

**Reading / editing an existing integration.** To *update* a site that already has an
integration (rather than author from scratch), read the current definition first. The
content script answers a request written to `data-toggl-focus-get-current` by writing the
matched integration to `data-toggl-focus-current`:

```js
// 1. request — the value just needs to change each time (a timestamp works):
document.documentElement.setAttribute('data-toggl-focus-get-current', String(Date.now()));
// 2. read the answer in a *follow-up* call (the content script replies asynchronously):
JSON.parse(document.documentElement.getAttribute('data-toggl-focus-current') || 'null');
// → { exists, kind: 'custom' | 'native' | 'none', definition, gated? }
```

- `kind: 'custom'` → an editable org integration; its id is preserved, so tweak the
  `definition` and **re-post it (above) to update the same record** in place.
- `kind: 'native'` → a built-in; editing forks a fresh custom override (mirror its shape).
- `kind: 'none'` → nothing matched — author from scratch (§3).
- `gated: true` → the side panel isn't open (this read is only answered while it is, i.e.
  *authoring mode*). Ask the user to open the panel on this tab, then retry.

The response is left in `data-toggl-focus-current` rather than auto-removed — that's
intentional: while the builder is open you can read it back in a follow-up call (and re-read
it), and the next request simply overwrites it.

---

## 3. Anatomy of the JSON — every attribute

### 3.1 Top level (the integration object)

| Field | Req | Purpose | Example |
|---|---|---|---|
| `id` | ✅ | Stable identifier. Use a fresh **UUID** (`crypto.randomUUID()`). When posting, the panel **keeps the form's own id** and ignores the one in your JSON, so the value is cosmetic during authoring — but use a UUID anyway. | a UUID |
| `name` | ✅ | Human label shown in the Toggl 2.0 UI. | `"GitHub"` |
| `domain` | ✅ | Host the integration activates on. Supports a `*.` wildcard subdomain. | `"github.com"`, `"*.atlassian.net"`, `"*.notion.so"` |
| `selectors` | — | Named CSS aliases reused across resolvers/anchors. Reference an alias elsewhere with `@aliasName`. Keeps long selectors DRY. | `{ "issue-viewer": "div[data-testid=\"issue-viewer-container\"]" }` |
| `resolvers` | — | Named recipes for **reading text** out of the page (the task description, project, tags, …). Referenced by key from a button's `description`/`metadata`. See §3.3. | see §3.3 |
| `buttons` | ✅ | One or more button placements. Most sites need several (detail view vs. list row vs. board card). See §3.2. | array of buttons |
| `cssContent` | — | Inline CSS string injected with the button. Use for small per-site layout tweaks. | `".toggl-wrapper{margin:0 4px}"` |
| `themeDetector` | — | Makes the button follow the host page's light/dark theme. `colorElement`: element whose background determines theme; `watchElement` (opt): element to observe for theme flips. | `{ "colorElement": "body" }` |
| `taskResolver` | — | Links page items to existing Toggl 2.0 tasks. `source` (an integration name, e.g. `"jira"`) + `keyPattern` (regex matched against the first word of the description to extract a task key). | `{ "source": "jira", "keyPattern": "^[A-Za-z][A-Za-z0-9]+-\\d+$" }` |

### 3.2 A button

Placement — **where** the button mounts:

| Field | Req | Purpose |
|---|---|---|
| `name` | ✅ | Unique-per-integration id for this placement. Also emitted as `data-toggl-button-wrapper="<name>"` on the injected wrapper — that's how you find it in §5. |
| `anchor` | ✅ | CSS selector for the element the button mounts relative to. |
| `append` | — | Position relative to `anchor`: `'first'` = prepend *inside* anchor · `'last'` = append *inside* anchor · `'before'` = previous sibling · `'after'` = next sibling. Omitted ≈ inside. |
| `container` | — | CSS selector for the root the button's resolvers read within, and the portal root for its popovers/menus. Use for modals/dialogs so description resolution is scoped (e.g. Jira's modal blanket). |
| `target` | — | CSS selector that scopes **where anchors are searched**. Anchors are queried *inside* `target` only. |
| `multiple` | — | `true` → mount on **every** matching anchor (list rows, board cards). `false` → only the first (a single detail view). |
| `wrapper` | — | HTML string wrapped around the button — your hook for layout (margins, inline-flex). Defaults to `<div/>`. Gets `data-toggl-button-wrapper` set on it. |
| `className` / `initialClassName` | — | Extra classes added to the wrapper for styling. |

Appearance:

| Field | Purpose |
|---|---|
| `variant` | `'minimal'` (compact icon) or `'default'` (fuller button). |
| `iconSize` | Icon size in px (e.g. `12`–`20`). Match the host UI's control size. |

Behaviour:

| Field | Purpose |
|---|---|
| `autoTrack` | Start tracking automatically when the page/item is detected (vs. requiring a click). |
| `quickLog` | Enable the quick-log affordance on the button. |
| `subscribe` | An element-selector the button **watches for mutations** to re-resolve its description — needed on SPAs where the title node updates in place (e.g. Jira navigating issue→issue). |
| `description` | What the button tracks as the task title. A **locator** (§3.3). |
| `metadata` | Extra fields (project, tags, …) read off the page. A map of name → locator (§3.3). |

### 3.3 Reading text: resolvers, locators, metadata

A **resolver** (entry in top-level `resolvers`) locates one piece of text. Kinds:

- **Element** (default) — walk the DOM and read text:
  - Ancestor (pick ≤1): `root: true` (search from document root) · `closest: '<sel|@alias>'` (nearest matching ancestor) · `parent: <n>` (n levels up).
  - Traverse (pick ≤1): `query: '<sel>'` (descendant) · `previousSibling`/`nextSibling: <n>`.
  - `textAttribute: '<attr>'` — read an attribute instead of `.textContent` (e.g. `aria-label`, `data-text`, `style`).
  - `regex` + `replace` — post-process the captured string (first capture group is used).
  - `unmarkdown: true` — strip markdown.
- **URL** — `{ "url": "path" | "query" | "hash", "regex": "...", "replace": "..." }` — read from the location (e.g. Jira's `urlKey` from `/browse/ABC-1`).
- **Literal** — `{ "literal": "fixed text" }`.

A **locator** (a button's `description` / a `metadata` entry) assembles resolvers into
the final string. Forms, simplest → richest:

```jsonc
"description": "issueTitle"                          // one resolver by key
"description": ["#", "number", ": ", "description"]  // array = concatenated; bare strings are literals
"description": { "resolvers": ["issueKey", " ", "issueTitle"] }   // object form
"description": [ { "resolvers": [...] }, { "resolvers": [...] } ] // ordered fallbacks: first non-empty wins
```

`metadata` maps a field name to a locator; add `"multiple": true` to collect several
(e.g. tags):

```jsonc
"metadata": {
  "project": { "locators": { "resolvers": ["detailProject"] } },
  "tags":    { "multiple": true, "locators": { "resolvers": ["detailTags"] } }
}
```

> Selector aliasing: any CSS-selector slot (`anchor`, `closest`, `query`, …) accepts
> `@aliasName` to pull the selector from the top-level `selectors` map.

### 3.4 A complete minimal example

The smallest thing worth posting: one button on a detail view that tracks a title.
Start here, confirm it renders, then layer on metadata and more buttons.

```jsonc
{
  "id": "b3f1c2a4-8d2e-4f6a-9c10-1a2b3c4d5e6f",  // any UUID for a new custom integration
  "name": "Acme Tasks",
  "domain": "app.acme.com",
  "resolvers": {
    "title": { "root": true, "query": "h1[data-task-title]" }  // read the page <h1>
  },
  "buttons": [
    {
      "name": "task-detail",          // → data-toggl-button-wrapper="task-detail"
      "anchor": "[data-testid='task-toolbar']",
      "append": "last",                // mount inside the toolbar, at the end
      "variant": "minimal",
      "iconSize": 16,
      "multiple": false,               // single detail view, not a list
      "description": ["title"]         // track the resolved <h1> text
    }
  ]
}
```

Post it by writing the JSON to the attribute (works the same from `javascript_tool` or
the DevTools console):

```js
document.documentElement.setAttribute(
  'data-toggl-focus-post',
  JSON.stringify(/* the object above */),
);
```

Then verify with the §5 snippet — expect exactly one `task-detail` wrapper inside the
toolbar. Once it's placed, add `metadata` (project/tags), `autoTrack`, or a second
button for the list/board view.

---

## 4. The build loop (interactive)

Don't build silently end-to-end. This is a **turn-taking loop**: the user can see the
page and knows the intent; you read the DOM and iterate fast. Check in at the marked
gates rather than guessing — a wrong selector is cheap to fix, a wrong *assumption*
about what they want wastes a whole round.

### Step 1 — Confirm you're on the right page
After the user says they've navigated + opened the side panel, take a `screenshot` and
read the DOM (`get_page_text` / `read_page`). Tell the user what page/item you see
("I see an issue detail with title X and a toolbar on the right"). If it's a login
wall, empty state, or the wrong view, say so and ask them to fix the state — **don't
proceed on a guess.**

### Step 2 — Interview for intent  ← gate
Before writing any JSON, ask the user (briefly, batched into one message) for what you
can't reliably infer from the DOM:
- **Position** — where should the button sit? Relate it to something visible ("right of
  the Share button", "inside the card, under the title").
- **Description** — which text is the task title? (the issue title, the card name…)
- **Single or multiple** — one button on this detail view, or one per row/card across a
  list/board? → `multiple`. **Don't lead with this as a question — probe the page first
  (Step 3) and infer it from the anchor match count.** A candidate anchor that matches
  once → single (detail view); one that matches many sibling rows/cards → `multiple`.
  Only *ask* when the DOM is genuinely ambiguous (e.g. a single-item view that's
  actually one row of a list that's currently filtered to one). Otherwise state your
  inference and let the user correct it: "I see 10 cards, so I'll put a button on each —
  shout if you only want it on one."
- **Metadata** (optional) — project, tags, anything else worth capturing.

Restate your understanding in one or two sentences and get an explicit **confirm**
before building. This is the gate the whole loop hangs on.

### Step 3 — Probe selectors
With `javascript_tool`, find a stable `anchor` for the position and a resolver `query`
for each piece of text. **Verify each selector's match count before trusting it — and
let that count decide `multiple`:**
```js
document.querySelectorAll('YOUR-ANCHOR-SELECTOR').length   // == 1 → multiple:false · > 1 (repeated rows/cards) → multiple:true
```
This is how you answer the single-vs-multiple question without interrogating the user
(Step 2): the page already knows. A count of 1 is a detail view; a count matching the
visible rows/cards is a list → `multiple: true`. If the count is unexpected (e.g. 1 when
you expected a list, or many when you expected one), the anchor is wrong, not the intent
— fix the selector before deciding.

Prefer `data-testid` / roles / durable prefixes over hashed class names (§6). For a
list, scope resolvers with `closest`/`@alias` so each row reads *its own* text — and
confirm the scoping by reading two different rows and checking their resolved text
differs (don't trust a single-row check; one card can accidentally read another's text).

### Step 4 — Post, verify, adjust  ← loop
Assemble the JSON (§3), **post it** (§2 — it auto-applies and injects), then **verify**
(§5: count the wrappers, then `screenshot`/`zoom`). Show the user the result each round.
If it's mis-placed or reads the wrong text, change `anchor` / `append` / `container` /
resolvers and **re-post** (re-posting re-applies). Repeat until the button is in the
right slot, the right count, and reads correctly — confirming with the user before
calling it done.

### Step 5 — Hand off to Save
Local apply is a draft — injected on this tab, but not persisted across browsers or
sessions. When the user is happy with placement and the tracked text, ask them to click
**Save** in the side panel to persist the integration to their organization. Don't Save
for them unless asked.

---

## 5. Checking the button position (did it work?)

The injected wrapper carries `data-toggl-button-wrapper="<button name>"`. Use it to
confirm the button mounted and landed where intended.

**A. Did it mount, and how many?** (`javascript_tool`)
```js
[...document.querySelectorAll('[data-toggl-button-wrapper]')].map((n) => {
  const r = n.getBoundingClientRect();
  return {
    name: n.getAttribute('data-toggl-button-wrapper'),
    rect: { x: Math.round(r.x), y: Math.round(r.y), w: Math.round(r.width), h: Math.round(r.height) },
    visible: r.width > 0 && r.height > 0,
    // The wrapper's own siblings — tells you which slot it landed in (`append`).
    prevSibling: n.previousElementSibling?.tagName ?? null,
    nextSibling: n.nextElementSibling?.tagName ?? null,
  };
});
```
- **0 nodes** → in likelihood order: (1) the post wasn't consumed — re-read
  `document.documentElement.hasAttribute('data-toggl-focus-post')`; if it's still `true`
  the content script never picked it up → you're on the wrong/hidden MCP tab or the page
  has no content script (§6 — confirm with a banner + `document.visibilityState`); (2) the
  side panel isn't open on the **exact tab you're driving**; (3) no org selected in
  "Saving to"; (4) anchor didn't match; (5) apply errored — check the side-panel toast.
  Work down that list before touching the selectors.

> **Arm a MutationObserver before posting** — it tells you definitively whether Toggl
> ever injected anything (vs. injected-then-stripped by an SPA re-render), and separates
> a *post* failure (observer never fires) from a *placement* failure (it fires but the
> slot is wrong):
> ```js
> window.__seen = [];
> window.__obs?.disconnect();
> window.__obs = new MutationObserver((muts) => {
>   for (const m of muts) for (const n of m.addedNodes)
>     if (n.nodeType === 1 && (n.matches?.('[data-toggl-button-wrapper]') || n.querySelector?.('[data-toggl-button-wrapper]')))
>       window.__seen.push(Date.now());
> });
> window.__obs.observe(document.documentElement, { childList: true, subtree: true });
> ```
> After posting, read `window.__seen.length`. **Never saw a wrapper** → the post didn't
> reach the panel (recheck consumed-ack / tab / panel-open / org). **Saw N then live count
> is 0** → Vue/React stripped the injected sibling; add `subscribe` and/or rethink `append`.
- **Wrong count** → flip `multiple` (1 expected but got many → `false`; per-row but
  got 1 → `true`, and make sure `anchor` matches each row).
- **Mounted but mis-placed** → it's in the DOM but the wrong slot: adjust `append`
  (inside vs. sibling) and/or `wrapper` styling; use `container`/`target` to scope.

**B. Look at it.** Screenshot the tab and `zoom` the anchor region for a close read of
alignment/spacing. Scroll the wrapper into view first if needed:
```js
document.querySelector('[data-toggl-button-wrapper="YOUR-NAME"]')
  ?.scrollIntoView({ block: 'center' });
```
Confirm against the host UI: is it beside the intended control, not overlapping,
correctly sized (`iconSize`), and themed (`themeDetector`)?

> **Which screenshot tool — prefer the Chrome MCP for page-only checks.** Both
> `mcp__Claude_in_Chrome__computer` (`action:'screenshot'` / `'zoom'`) and the desktop
> `computer` MCP can do this, but they're not equivalent:
> - **Chrome MCP screenshot/zoom** captures *only the page viewport*. It needs **no
>   extra access grant**, doesn't hide the user's other apps, and won't capture anything
>   outside the page — the cleanest, most private choice for confirming the **button on
>   the page** (placement, alignment, theme). It has its own `zoom` action too. Use this
>   **by default** for §5-B. (Verified working on a live board: both `screenshot` and
>   `zoom` returned the page with the injected wrappers.)
> - **Desktop `computer` screenshot** captures the whole window, so it's the *only* way to
>   see the **extension side panel** (the "Saving to" org, the loaded JSON, "Applied
>   locally" toast) — that panel is browser chrome, not page DOM, so the Chrome MCP can't
>   see it. But it requires a computer-use **access grant for Chrome** and momentarily
>   hides non-allowlisted apps. Reserve it for when you specifically need to verify
>   *panel state*, not just the button.
>
> Rule of thumb: **button on the page → Chrome MCP screenshot; side-panel state → desktop
> `computer` screenshot.** Don't reach for the heavier desktop grant just to look at a
> button.

**C. Description/metadata sanity — read it straight off the DOM.** While the side panel is
open (*authoring mode*), every wrapper carries `data-toggl-debug-*` attributes with exactly
what that button resolved — read them instead of hovering/screenshotting:
```js
[...document.querySelectorAll('[data-toggl-button-wrapper]')].map((n) => ({
  button:      n.dataset.togglDebugButton,
  anchor:      n.dataset.togglDebugAnchor,
  description: n.dataset.togglDebugDescription,                      // the tracked title
  metadata:    JSON.parse(n.dataset.togglDebugMetadata  || '{}'),
  resolvers:   JSON.parse(n.dataset.togglDebugResolvers || '{}'),    // { alias: { matched, text } }
}));
```
`resolvers` is the fast way to debug a blank/wrong title: find the alias whose `matched`
is `false` (its selector found **no element** → fix the `query`/`closest`) versus
`matched: true` with empty `text` (found the node but read the wrong/empty text → fix
`textAttribute`/`regex` or the `query`). For a `multiple` button this reads **every card's**
resolved values in one call — confirm they differ per card (proves `closest` scoping). If
the `data-toggl-debug-*` attributes are absent, the side panel isn't open. (Hovering/opening
the button still works as a visual cross-check.)

**D. Is it actually *clickable*? (visible ≠ clickable).** A button can render perfectly
in a screenshot yet sit *under* another element at its center point, so clicks never
reach it. This is common on **cards/rows with a stretched click-overlay** (a full-size
`position:absolute` link/`::before` that makes the whole card clickable) — the overlay,
or a sibling stacking context like a status badge, can paint over your button. Don't
trust the screenshot; **hit-test** the button's center, and simulate the overlay being
raised on hover/activate (the failure the user actually hits):
```js
[...document.querySelectorAll('[data-toggl-button-wrapper="YOUR-NAME"]')].map((w) => {
  w.scrollIntoView({ block: 'center' });
  const r = w.getBoundingClientRect();
  if (r.width === 0) return { offscreen: true };          // can't hit-test off-screen
  const cx = Math.round(r.left + r.width/2), cy = Math.round(r.top + r.height/2);
  const card = w.closest('YOUR-CARD-OR-ROW-SELECTOR');    // the host's card/row container
  const topNow = document.elementFromPoint(cx, cy);
  // simulate "activate": some hosts bump the overlay's z-index on hover
  const ov = card?.querySelector('YOUR-OVERLAY-SELECTOR'); // the host's stretched click-overlay
  const pz = ov?.style.zIndex; if (ov) ov.style.zIndex = '1';
  const topActive = document.elementFromPoint(cx, cy);
  if (ov) ov.style.zIndex = pz;
  return {
    clickableAtRest:  !!topNow?.closest('[data-toggl-button-wrapper="YOUR-NAME"]'),
    clickableActive:  !!topActive?.closest('[data-toggl-button-wrapper="YOUR-NAME"]'),
    coveredBy: topNow ? (topNow.getAttribute('data-testid') || topNow.className?.slice?.(0,40)) : null,
  };
});
```
If `coveredBy` names a host element, you have a stacking-context problem — see the §6
"stretched click-overlay" pitfall for the fix.

---

## 6. Pitfalls

- **Nothing injects after a post:** check the consumed-ack first
  (`hasAttribute('data-toggl-focus-post')` — if still `true`, the content script never read
  it, so you're on the wrong/hidden tab or a page with no content script). If it *was*
  consumed: side panel not open on *this* tab, no org selected, or invalid JSON (look for
  the inline error / error toast). The bridge is per-tab by design.
- **Wrong / hidden tab with the MCP:** the Chrome MCP frequently drives a tab in its
  own tab group — often a **background window the user isn't looking at** (`navigate`
  can land you on a tab that reports `document.visibilityState === 'hidden'`,
  `hasFocus() === false`). The user then opens the side panel on *their* visible tab,
  and your posts hit a different, panel-less tab. **Confirm tab identity early:**
  inject a bright fixed banner and ask the user if they see it, and check
  `document.visibilityState`. The panel must be on the **exact tabId you post to**.
- **`javascript_tool` wedged — `Cannot access a chrome-extension:// URL of different
  extension`:** the MCP's debugger-based execution breaks when the focused tab is an
  extension page or the debugger session goes stale (it can stick across tab/group
  churn). Tell: `get_page_text` (content-script read) still works while `javascript_tool`
  fails on every tab. Closing tabs, closing the side panel, new tab groups, and
  re-navigating **do not** clear it. What does: **fully quit & reopen Chrome** (or close
  the whole MCP window, not just tabs), disable conflicting extensions, then reconnect
  the Claude-in-Chrome browser. Don't burn turns re-trying the tool — switch the user to
  the console path (which is unaffected) or have them do a real browser restart.
- **MCP DOM-read redaction:** reading raw `el.outerHTML` often comes back
  `[BLOCKED: Cookie/query string data]`, and values like UUIDs return
  `[BLOCKED: Sensitive key]`. Don't fight it — extract **structured** info instead
  (`{ tag, className parts, data-testid, textContent slice }`) and walk children
  shallowly. Also: `javascript_tool` does **not** reliably support top-level `await`
  (`await is only valid in async functions`) — avoid it; split into separate calls.
- **Attribute value must be a JSON string:** `setAttribute` coerces to string, so always
  `JSON.stringify(...)` the definition. Don't try to pass functions or DOM nodes — it's
  plain JSON only.
- **Brittle selectors:** hashed CSS-module class names (e.g.
  `HeaderMetadata-module__…__Dg2N7`) change between deploys. Prefer `data-testid` /
  roles; if you must use a generated class, expect to revisit it.
- **SPA re-renders drop the button or stale the title:** rely on the built-in anchor
  observer for re-mount, and add `subscribe` so the description re-resolves when the
  watched node mutates in place.
- **List vs. detail:** these are *different buttons* with different `anchor`,
  `multiple`, and scoping (`container`/`closest`). Don't force one button to serve
  both — add a separate entry in `buttons` for each surface.
- **The posted `id` is ignored** in favour of the form's id; don't rely on it to
  target an existing record.
- **Button covered by a stretched click-overlay (stacking-context battle):** cards/rows
  that are "click anywhere to open" use a full-size `position:absolute` overlay (a link or
  `::before` with `inset:0` covering the whole card). Your button can *look* on top but be
  un-clickable, and **`z-index` won't save you** — once your wrapper and the covering
  element live in *different* stacking contexts, bumping z-index to 9999 does nothing
  (verify with §5-D). Two fixes, in order of preference:
  - **Place the button where the host already lifts its own interactive children.** Hosts
    raise their real controls (title link, labels) above the overlay with `position:relative`
    or `isolation:isolate`. Anchoring *next to one of those* (e.g. `append:"after"` the
    title link) inherits a safe slot — this is why an inline-after-title button "just works"
    while a footer one gets covered.
  - **Give your wrapper its own stacking context that beats the overlay**, via `cssContent`
    targeting `[data-toggl-button-wrapper="<name>"]`. The reliable recipe: keep it **in
    normal flow** (`position:relative`, *not* absolute — absolute drops it back under sibling
    contexts) and add **`isolation:isolate`** (the same trick the host uses), plus
    `margin-left:auto`/`align-self` for placement:
    ```jsonc
    "anchor": ".CARD-FOOTER-SELECTOR", "append": "last",   // mount at the card bottom
    "cssContent": "[data-toggl-button-wrapper=\"card-button\"]{position:relative;isolation:isolate;z-index:1;margin-left:auto;align-self:center;}"
    ```
    Diagnose first: walk up from `document.elementFromPoint(cx,cy)` and look for an ancestor
    with `isolation:isolate`, `transform`, `opacity<1`, or a `z-index` — that's the context
    boundary your z-index can't cross. Re-confirm the fix with §5-D (at rest **and** active).
  - Footer/flex placement gotcha: a footer with `justify-content:space-between` + `flex-wrap`
    *reorders* appended/prepended items — a "first child" can render on the right. Use
    `margin-left:auto`, `order`, or accept the side it lands and verify visually.
- **Duplicate buttons after re-posting with a *changed* `anchor`:** re-posting re-applies,
  but switching the `anchor` (or doing live DOM experiments) can leave the *old* buttons in
  place while adding new ones — you'll see 2–3× the expected wrapper count. The clean reset
  is to **reload the page** (clears all injected wrappers; the local draft does *not*
  auto-reinject), then post the final definition once. Always recount wrappers (§5-A) after
  a re-post to catch stacking early.
- **Iterate CSS live, then re-post clean.** To find a working placement/`cssContent` recipe
  fast, mutate **one** wrapper's `.style` directly and hit-test it (§5-D) — far quicker than
  a full re-post per attempt. But single-element edits make that card differ from the rest;
  once the recipe is right, fold it into `cssContent`, reload, and post once so all cards
  are uniform.

---

## 7. Worked example — a `multiple` button across a board/list

A generic board where every card carries a stable test id and a title link. (The
selectors below are **placeholders** — probe the real ones live, §4.) We want one button
per card, after the title, tracking `#<id> <title>`. The key move for lists: **scope each
resolver to its own card with `closest: "@card"`** so card N reads card N's text, not the
first card's.

```jsonc
{
  "id": "<uuid>",
  "name": "Acme Board",
  "domain": "app.example.com",
  "selectors": { "card": "[data-testid=\"card\"]" },              // alias reused below
  "resolvers": {
    "cardId":    { "closest": "@card", "query": "[data-testid=\"card-id\"]" },     // "#1234"
    "cardTitle": { "closest": "@card", "query": "[data-testid=\"card-title\"]" }   // the title
  },
  "buttons": [
    {
      "name": "card-button",
      "anchor": "[data-testid=\"card-title\"]",  // one anchor per card
      "append": "after",                          // inline, just after the title
      "multiple": true,                           // ← mount on every card
      "variant": "minimal",
      "iconSize": 14,
      "description": { "resolvers": ["cardId", " ", "cardTitle"] }  // "#1234 Some task…"
    }
  ],
  "themeDetector": { "colorElement": "body" }
}
```

Verifying a `multiple` button: expect **one wrapper per card** (count == card count), each
with `visible: true` and `prevSibling` == the title link. A handy per-card check is to
read each wrapper's `closest('<your-card-selector>')` and confirm the cards are all
distinct — that proves the `closest`-scoping worked and every button reads its own row.

**Placement variant — bottom of the card (and the overlay caveat).** The `append:"after"`
the title anchor above is the *easy* slot: hosts usually lift the title link into its own
stacking context, so a button next to it sits above any stretched click-overlay and is
clickable for free. If the user instead wants the button at the **card bottom**, anchoring
in a footer element *looks* fine but the button can be covered by the card's click-overlay
or a sibling badge that uses `isolation:isolate` — `z-index` alone won't fix it (different
stacking contexts; see §6). The working bottom recipe keeps the button in flow and gives
it its own context:

```jsonc
"buttons": [{
  "name": "card-button",
  "anchor": ".CARD-FOOTER-SELECTOR",
  "append": "last",
  "multiple": true,
  "variant": "minimal",
  "iconSize": 14,
  "description": { "resolvers": ["cardId", " ", "cardTitle"] }
}],
"cssContent": "[data-toggl-button-wrapper=\"card-button\"]{position:relative;isolation:isolate;z-index:1;margin-left:auto;align-self:center;}"
```

Always confirm with the §5-D hit-test (at rest **and** with the overlay raised), not just
a screenshot — visible ≠ clickable.

---

## 8. Starter notes for common integrations

These are **structural guesses** to seed Steps 2–3 — *not* verified selectors. Every
host changes its DOM over time, so **always probe the real anchors/resolvers live (§4)**
and verify match counts before trusting anything here. Treat them as "where to look and
what shape to expect," not copy-paste config.

- **Google Calendar** (`calendar.google.com`) — the open-event popover/dialog is a single
  detail surface → `multiple: false`. Anchor near the popover's action row (the icons by
  the event title); read the title from the event-title heading. Google uses hashed class
  names, so prefer `[role]` / `aria-label` / `data-*` hooks, and scope resolvers with
  `container`/`closest` to the open dialog (`[role="dialog"]`) so you don't read the
  calendar grid behind it.
- **Trello** (`trello.com`) — two surfaces, two buttons. The **card back** (opened card
  detail) is single (`multiple:false`); anchor in the card-detail header/action list, title
  = card name. **Board cards** are a list (`multiple:true`); one button per card tile, title
  = card name, scoped with `closest` to the card. Watch for the "click anywhere opens the
  card" overlay on board tiles (§6 stacking-context fix).
- **Linear** (`linear.app`) — issue detail is a single surface, and the SPA swaps
  issue→issue **in place**, so add `subscribe` on the title node or the description goes
  stale on navigation. Title = issue title; the issue key (e.g. `ENG-123`) is a good `url`
  resolver from the path, or a header element.
- **Monday / ClickUp / Height** and similar boards — expect the same board-vs-detail split
  as Trello: a `multiple` button per row/card plus a single button on the item detail, with
  per-row `closest` scoping and likely an overlay battle on the cards.

> Already native: **GitHub, Jira, Asana, Notion, Todoist** (and others) ship built-in.
> Building a custom integration for one of these *overrides* the native one — the
> current-integration read (§2) reports `kind: "native"`, and editing it forks a fresh
> custom definition. Mirror their structure rather than starting from scratch.
