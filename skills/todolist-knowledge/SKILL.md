---
name: todolist-knowledge
description: Todolist desktop app knowledge — .todo file format, plugin system, plugin API, view protocol, communication architecture patterns, and plugin contributes. Use this skill whenever working with .todo files, building or debugging Todolist plugins, implementing plugin views or head views, choosing a plugin architecture (view+service, inject-only, view+inject, head-view-bridge, plugin-call-service, etc.), implementing plugin communication (postMessage RPC, localStorage bridge, inject.js patterns, tree-update push model, hidden head view bridge, plugin-call-service CustomEvent, hover-based lazy decorations), or working with the Todolist plugin API and contributes system (context menus, events, views). Also use when users mention todolist format, todo tree structure, plugin service methods, plugin iframe communication, inject.js, callPlugin, double-unwrap, plugin architecture selection, inject.js service access, head view bridge, plugin-call-service, context menu node custom fields, or plugin file sync.
---

# Todolist Knowledge

Reference for the Todolist desktop app (Electron). Read the relevant reference file when you need details — do NOT rely on this summary alone for implementation.

## Reference Files

| File | When to Read |
|------|-------------|
| `references/todo-format.md` | Working with `.todo` files — type definitions, field details, tag system, examples, common operations |
| `references/app-config.md` | App configuration (`~/.todoListNative.json`) — all config fields, related paths, explorer filtering |
| `references/plugin-api.md` | Building plugin services — API method table, pluginContributes spec (contextMenus, events, views.head/topbar/topfix), installation, lifecycle & reload behavior |
| `references/plugin-architecture.md` | Choosing a plugin architecture — 7 architecture types (view+service, service-only, inject-only, view+inject, triple-layer, client-side tree analysis, remote view), quick reference table, decision flowchart |
| `references/plugin-view.md` | Building plugin views — message protocol, complete HTML template, client-side tree analysis pattern, i18n/lang detection, dark mode, panel auto-refresh, head view bridge, dual-mode view pattern, pitfalls |
| `references/plugin-communication.md` | Plugin communication patterns — callPlugin RPC, double-unwrap, view-to-inject messaging, localStorage bridge, push-based tree updates, CustomEvent bridge, `plugin-call-service` CustomEvent, hover-based lazy decorations, inject.js patterns (singleton, MutationObserver, draggable), common pitfalls |

## Essential Rules

### .todo File Format

A `.todo` file is JSON with one root key `todotree`. Everything lives under it:
```javascript
const store = JSON.parse(fileText).todotree
const tree = store?.tree ?? []
```
Do NOT put `tree`, `tags`, `title` at the outermost JSON level. Required fields: `tree`, `expandKeys` (`[]`), `add_mode` (`'bottom'`), `timelines` (`[]`).

### Plugin Structure

Plugins are installed to `~/.todoListNative/plugins/`. A plugin has up to three layers:
```
my-plugin/
  package.json        # name, version, view, main, viewjs, viewSize, pluginContributes
  view/index.html     # Frontend iframe (optional)
  service/index.js    # Node.js backend (optional)
  inject.js           # Host DOM script (optional)
```

Service return format: `{ ok: true, result: ... }` or `{ ok: false, error: "..." }`.

### Critical Pitfalls (memorize these)

1. **Double-unwrap**: Service returns `{ ok, result }`, host wraps again → `const inner = res?.result || res; const data = inner?.result || inner`
2. **callPlugin timeout**: Always add 2s timeout — without it, promises hang forever
3. **inject.js cannot call service directly**: Host source verification rejects non-iframe sources. Use `plugin-call-service` CustomEvent (preferred) or hidden head view bridge
4. **Context menu `node`**: May not preserve custom fields — always read from `api.getTree()`, not from `node.todo`
5. **Plugin file sync**: Edits must be copied to `~/.todoListNative/plugins/<name>/` and app reloaded
6. **React re-renders wipe DOM injections**: Use hover-based lazy decorations instead of permanent injection
