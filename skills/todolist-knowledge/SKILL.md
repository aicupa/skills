---
name: todolist-knowledge
description: Todolist desktop app knowledge — .todo file format, plugin system, plugin API, view protocol, and plugin contributes. Use this skill whenever working with .todo files, building or debugging Todolist plugins, implementing plugin views or head views, or working with the Todolist plugin API and contributes system (context menus, events, views). Also use when users mention todolist format, todo tree structure, plugin service methods, or plugin iframe communication.
---

# Todolist Knowledge

Reference for working with the Todolist desktop app (Electron). This skill covers three domains — read the relevant reference file when you need full details.

## Reference Files

| File | When to Read |
|------|-------------|
| `references/todo-format.md` | Working with `.todo` files — full type definitions, field details, tag system, examples, common operations |
| `references/plugin-api.md` | Building plugin services — full API method table, pluginContributes spec (contextMenus, events, views.head), installation |
| `references/plugin-view.md` | Building plugin views — message protocol, complete HTML template, pitfalls (timeout, double-unwrap, auto-height, theme) |

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
  "viewSize": { "width": 800, "height": 600 },
  "pluginContributes": {
    "contextMenus": [{ "title": "Action", "title_zh": "操作", "command": "methodName" }],
    "events": { "paste": { "check": "checkMethod", "handler": "handleMethod" } },
    "views": { "head": true }
  },
  "dependencies": { "@aicupa/api": "^1.0.1" }
}
```

Service pattern:

```javascript
const { createPlugin } = require('@aicupa/api')
module.exports = createPlugin((api) => ({
  async myMethod(params) {
    const tree = await api.getTree(params.filePath)
    return { ok: true, result: tree }
  }
}))
```

Key APIs: `api.getTree()`, `api.reload()`, `api.store()`, `api.readFile()`, `api.writeFile()`, `api.clipboard.writeText()/.readText()`, `api.base64.encode()/.decode()`, `api.mapTree()`.

Return format: `{ ok: true, result: ... }` or `{ ok: false, error: "..." }`.

For full API table, pluginContributes spec, and installation details, read `references/plugin-api.md`.

### Plugin View Communication

Local views (`"view": "./view"`) communicate via `postMessage` in a sandboxed iframe. URL views (`"view": "https://..."`) open in a separate BrowserWindow with no bridge.

Message types:
- `plugin-call` (view→host): Call service method `{ type, id, method, params }`
- `plugin-result` (host→view): Response `{ type, id, result }`
- `plugin-init` (host→view): Init with `{ filePath, theme: { color, backgroundColor } }`
- `plugin-tree-update` (host→view): Tree data push (head views)
- `plugin-request-tree` (view→host): Request current tree (head views)

Critical pitfalls:
1. **Timeout**: Always add a 2s timeout to `callPlugin` — without it, promises hang forever if host isn't ready
2. **Double-unwrap**: Service returns `{ ok, result }`, `pluginCallService` wraps again. Use: `const inner = res?.result || res; const data = inner?.result || inner`
3. **Head view height**: Set `html, body { height: auto; overflow: hidden }` for auto-height to work
4. **Theme**: Apply `theme.color` / `theme.backgroundColor` from `plugin-init` for dark/light mode

For the complete HTML template and detailed pitfall explanations, read `references/plugin-view.md`.
