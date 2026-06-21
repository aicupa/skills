---
name: todolist-knowledge
description: Todolist desktop app knowledge — .todo file format, plugin system, plugin API, view protocol, communication architecture patterns, and plugin contributes. Use this skill whenever working with .todo files, building or debugging Todolist plugins, implementing plugin views or head views, choosing a plugin architecture (view+service, inject-only, view+inject, etc.), implementing plugin communication (postMessage RPC, localStorage bridge, inject.js patterns, tree-update push model), or working with the Todolist plugin API and contributes system (context menus, events, views). Also use when users mention todolist format, todo tree structure, plugin service methods, plugin iframe communication, inject.js, callPlugin, double-unwrap, or plugin architecture selection.
---

# Todolist Knowledge

Reference for working with the Todolist desktop app (Electron). This skill covers three domains — read the relevant reference file when you need full details.

## Reference Files

| File | When to Read |
|------|-------------|
| `references/todo-format.md` | Working with `.todo` files — full type definitions, field details, tag system, examples, common operations |
| `references/plugin-api.md` | Building plugin services — full API method table, pluginContributes spec (contextMenus, events, views.head, views.topbar, views.topfix), installation |
| `references/plugin-view.md` | Building plugin views — message protocol, complete HTML template, client-side tree analysis pattern, i18n/lang detection, dark mode detection, pitfalls (timeout, double-unwrap, auto-height, theme) |
| `references/plugin-communication.md` | Plugin communication architecture patterns — 7 architecture types (view+service, service-only, inject-only, view+inject, triple-layer, client-side tree analysis, remote view), communication patterns (callPlugin RPC, double-unwrap, view-to-inject messaging, localStorage bridge, push-based tree updates, graceful degradation), inject.js patterns (singleton guard, DOM injection, MutationObserver, draggable widgets), common pitfalls, decision flowchart |

## Quick Reference

### .todo File Format

A `.todo` file is JSON with one root key. Everything lives under `todotree`:

```javascript
// Reading
const store = JSON.parse(fileText).todotree
const tree = store?.tree ?? []

// Writing
JSON.stringify({ todotree: store })
```

Do NOT put `tree`, `tags`, `title` at the outermost JSON level.

Core structure:
- `todotree.tree`: Array of `{ key, children, todo }` nodes (unlimited nesting)
- `todotree.tags`: Tag definitions `{ label, color, level, fontColor, timeline? }`
- `todotree.timelines`: Ordered todo id array for the horizontal strip
- `todotree.expandKeys`, `todotree.add_mode`: Required (use `[]` and `'bottom'` as defaults)

Each `todo` item has: `id` (number, unique), `content` (string), `done` (boolean), `level` (`'default'|'secondary'|'success'|'warning'|'danger'`). Optional: `date` (Unix seconds comma-separated), `start`/`end` (milliseconds), `tags` (string[] referencing `todotree.tags` keys), `link`, `fileLink`, `focus`, `keep`, `tip`.

Prefer child nodes over `tip` for supplemental content — `tip` is hidden in UI until enabled.

For full type definitions, examples, and helper functions, read `references/todo-format.md`.

### App Config (`~/.todoListNative.json`)

The desktop app stores global configuration in `~/.todoListNative.json`. Read via `getConfig()`, write via `saveConfig(partialConfig)` (shallow merge).

```json
{
  "currentDir": "/Users/xxx/my-todos",
  "currentKey": "/Users/xxx/my-todos/work.todo",
  "currentName": "work",
  "recent": [{ "path": "/path/to/file.todo", "name": "file" }],
  "expandedKeys": ["/Users/xxx/my-todos/2026"],
  "hideAside": false,
  "asidePos": 261,
  "winSize": [1251, 810],
  "lang": "zh-cn",
  "theme": "dark",
  "debug": true,
  "token": "cloud-sync-token",
  "backgroundImage": "https://example.com/bg.jpg",
  "backgroundOp": "8",
  "backgroundSize": "cover",
  "backupSize": "9.5",
  "workPendTimeMax": "9.5",
  "noticeKey": "20260617",
  "dailyInfo": { "date": "06-18", "start": 1781748208423 },
  "frameUrl": "https://...",
  "aiCommand": "clix chat \"$cmd\"",
  "bottomHtmlTool": "...",
  "enableFloatingBall": true,
  "chatVisible": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `currentDir` | `string` | Active workspace folder path. Defaults to `~/.todoListNative` if empty |
| `currentKey` | `string` | Full path of the currently open `.todo` file |
| `currentName` | `string` | Display name of the current file (filename without extension) |
| `recent` | `Array<{path, name}>` | Recently opened files/folders, max 20, newest first |
| `expandedKeys` | `string[]` | Expanded folder paths in the sidebar tree |
| `hideAside` | `boolean` | Whether the sidebar is collapsed |
| `asidePos` | `number` | Sidebar width in pixels |
| `winSize` | `[number, number]` | Window `[width, height]` — persisted on resize, restored on launch |
| `lang` | `string` | UI language (`'zh-cn'`, `'en'`, etc.) |
| `theme` | `string` | `'dark'` or `'light'` — defaults to dark if absent |
| `debug` | `boolean` | Enables the DevTools menu item in the tray context menu |
| `token` | `string\|null` | Cloud sync auth token (set via `SetToken`, cleared via `ClearToken`) |
| `backgroundImage` | `string` | URL of the custom background image |
| `backgroundOp` | `string` | Background opacity (`"0"`–`"10"`, as string) |
| `backgroundSize` | `string` | CSS `background-size` for the background image (e.g. `"cover"`, `"contain"`, `"100% auto"`) — defaults to `cover` in CSS |
| `backupSize` | `string` | Max backup size in MB (as string, e.g. `"9.5"`) |
| `workPendTimeMax` | `string` | Max work duration in hours before overtime alert (as string, e.g. `"9.5"`) |
| `dailyInfo` | `{date, start}` | Today's work session — `date` is `"MM-DD"`, `start` is Unix ms timestamp |
| `noticeKey` | `string` | Last-read changelog date key (e.g. `"20260617"`) — suppresses re-display |
| `frameUrl` | `string` | Custom iframe URL for the GPT/AI panel |
| `aiCommand` | `string` | Shell command template for AI chat — `$cmd` is replaced with user input |
| `bottomHtmlTool` | `string` | Custom HTML tool injected at the bottom area |
| `enableFloatingBall` | `boolean` | Show floating action ball (native only) |
| `chatVisible` | `boolean` | Whether the AI chat panel is open |

Related paths:
- `~/.todoListNative/` — default workspace dir, plugin storage root
- `~/.todoListNative.bak` — backup file
- `~/.todoListNative_daily.json` — daily work-time log (separate from config)

### Plugin System

Plugins extend the Todolist desktop app with a frontend view (HTML iframe) and/or a backend service (Node.js). Installed to `~/.todoListNative/plugins/`.

```
my-plugin/
  package.json        # name, version, view, main, viewSize, pluginContributes
  view/index.html     # Frontend (optional)
  service/index.js    # Backend (optional)
```

Key `package.json` fields:

```json
{
  "name": "@aicupa/plugin-my-plugin",
  "version": "1.0.0",
  "view": "./view",
  "main": "./service",
  "viewjs": "./inject.js",
  "viewSize": { "width": 800, "height": 600 },
  "pluginContributes": {
    "contextMenus": [{ "title": "Action", "title_zh": "操作", "command": "methodName" }],
    "events": { "paste": { "check": "checkMethod", "handler": "handleMethod" } },
    "views": { "head": true }
  }
}
```

- `viewjs` (optional): Path to a JS file that will be injected into the app's `document.head` as a `<script>` tag when plugins load. Runs in the main app context (not iframe).

Service pattern (use JSDoc for type hints):

```javascript
/**
 * @param {import('@aicupa/api').PluginApi} api
 */
module.exports = (api) => ({
  async myMethod(params) {
    const tree = await api.getTree(params.filePath)
    return { ok: true, result: tree }
  }
})
```

Key APIs: `api.getTree()`, `api.reload()`, `api.store()`, `api.readFile()`, `api.writeFile()`, `api.fetch()`, `api.fs`, `api.os`, `api.path`, `api.crypto`, `api.clipboard.writeText()/.readText()`, `api.base64.encode()/.decode()`, `api.mapTree()`, `api.setBackground()`.

Return format: `{ ok: true, result: ... }` or `{ ok: false, error: "..." }`.

For full API table, pluginContributes spec, and installation details, read `references/plugin-api.md`.

### Plugin Architecture Types

Plugins fall into one of 7 archetypes based on which layers they use:

| Architecture | view | inject.js | service | Use Case |
|-------------|------|-----------|---------|----------|
| View + Service | iframe UI | — | Node.js backend | Settings panel + file I/O, API calls, config |
| Service-Only | stub | — | Node.js backend | Context menu / paste event handlers, tree transforms |
| Inject-Only | stub | host DOM | stub | Visual effects, floating widgets, audio players |
| View + Inject | iframe UI | host DOM | — | Settings panel + visual effects (localStorage bridge) |
| View + Inject + Service | iframe UI | host DOM | Node.js backend | Full-stack: settings + effects + file/API access |
| Client-Side Tree Analysis | head/topbar/topfix | — | stub | Display-only indicators from tree data |
| Remote View | external URL | — | — | Embed external web app in a BrowserWindow |

Choose architecture using this decision flow:
- Need host DOM effects? → inject.js required
- Need Node.js capabilities (file I/O, `api.fetch()`, clipboard)? → service required
- Need settings UI panel? → view required
- Only reading tree data for display? → Client-Side Tree Analysis (no service needed)

For full details, decision flowchart, and code patterns, read `references/plugin-communication.md`.

### Plugin Communication Patterns

**callPlugin RPC** — The standard view→service call pattern. Always include a 2s timeout:

```javascript
function callPlugin(method, params) {
  return new Promise((resolve, reject) => {
    const id = ++callId
    const timer = setTimeout(() => { delete pending[id]; reject(new Error('timeout')) }, 2000)
    pending[id] = {
      resolve: (v) => { clearTimeout(timer); resolve(v) },
      reject: (e) => { clearTimeout(timer); reject(e) }
    }
    window.parent.postMessage({ type: 'plugin-call', id, method, params }, '*')
  })
}
```

**Double-unwrap results** — Service returns `{ ok, result }`, host wraps again:

```javascript
function unwrap(res) {
  const inner = res?.result || res
  return inner?.result !== undefined ? inner.result : inner
}
```

**View→Inject messaging** — inject.js listens on host window for `postMessage` from view iframe. Use `localStorage` + `postMessage` dual write for persistence + real-time sync.

**localStorage bridge** — Shared state between view iframe and inject.js. Use `storage` event for cross-tab reactivity. Namespace keys with plugin prefix.

**inject.js essentials** — Singleton guard (`window.__MY_PLUGIN__`), DOM dedup (`getElementById` before create), `MutationObserver` for SPA resilience, `position: fixed` + `pointer-events: none` for overlays.

For complete patterns, inject.js templates, and pitfall details, read `references/plugin-communication.md`.

### Plugin View Communication

Local views (`"view": "./view"`) communicate via `postMessage` in a sandboxed iframe. URL views (`"view": "https://..."`) open in a separate BrowserWindow with no bridge.

Message types:
- `plugin-call` (view→host): Call service method `{ type, id, method, params }`
- `plugin-result` (host→view): Response `{ type, id, result }`
- `plugin-init` (host→view): Init with `{ filePath, theme: { color, backgroundColor } }`
- `plugin-tree-update` (host→view): Tree data push (head/topbar/topfix views)
- `plugin-request-tree` (view→host): Request current tree (head/topbar/topfix views)

View types:
- `views.head` — full-width block iframe above the progress bar
- `views.topbar` — inline iframe in the PageTitle area (top-right, alongside navigator). Non-mobile/non-simple-mode only.
- `views.topfix` — full-width block iframe between filters and tree content

All types share the same protocol. Choose `topbar` for compact indicators, `head` for above-progress content, `topfix` for content between filters and tree.

Head/topbar view tips:
- Views receive `plugin-tree-update` automatically on every tree change — use this for client-side analysis instead of calling the service when only read access is needed
- `plugin-init` may include `lang` — use it (with `navigator.language` fallback) for i18n
- Detect dark mode from `theme.backgroundColor` luminance to apply `.dark` CSS class

Critical pitfalls:
1. **Timeout**: Always add a 2s timeout to `callPlugin` — without it, promises hang forever if host isn't ready
2. **Double-unwrap**: Service returns `{ ok, result }`, `pluginCallService` wraps again. Use: `const inner = res?.result || res; const data = inner?.result || inner`
3. **Head view height**: Set `html, body { height: auto; overflow: hidden }` for auto-height to work
4. **Theme**: Apply `theme.color` / `theme.backgroundColor` from `plugin-init` for dark/light mode

For the complete HTML template, client-side analysis pattern, and detailed pitfall explanations, read `references/plugin-view.md`.
