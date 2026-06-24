# Plugin Architecture — Full Reference

Every plugin falls into one of these archetypes. Choose based on what the plugin needs to do.

## Architecture Types

### 1. View + Service (Standard Panel)

The view iframe communicates with the service via the host's postMessage bridge. Use when the plugin needs a settings/interaction panel AND Node.js capabilities (file I/O, HTTP proxy, clipboard, config access).

```
view/index.html  ──postMessage──>  Host App  ──pluginCallService──>  service.js
     (iframe)    <──plugin-result──           <──{ ok, result }──     (Node.js)
```

**Examples:** plugin-background-images, plugin-auth-msc, plugin-balance, todolist-plugin-demo (plugin-overdue-tasks)

**When to use:**
- The view needs data from the filesystem, external APIs (via `api.fetch()`), or app config
- The plugin needs to persist state across restarts (via `api.fs` or `api.store()`)
- The plugin needs to modify app state (`api.setBackground()`, `api.reload()`, `api.store()`)

### 2. Service-Only (No View)

All logic lives in the service. The host invokes methods through `pluginContributes` (context menus, paste events). The view is a stub or absent.

```
Host App  ──contextMenu/event──>  service.js
          <──{ ok, result }──     (Node.js)
```

**Examples:** plugin-cut

**When to use:**
- The plugin operates on tree data (cut/paste, sort, transform)
- Triggered by context menu or paste events, not by a UI panel
- No visual output needed beyond what the host provides

### 3. Inject-Only (Host DOM Manipulation)

All logic lives in `inject.js`, which runs in the host page context. The service and view are empty stubs. Use for visual effects, floating widgets, or audio that must live in the host DOM.

```
inject.js  ──directly manipulates──>  Host DOM
(runs in host context, not sandboxed)
```

**Examples:** plugin-bg-music

**When to use:**
- The plugin adds visual/audio effects to the host page (particles, overlays, music players)
- No settings panel is needed (or settings are minimal, handled in inject.js itself)
- The effect must persist across SPA navigation

### 4. View + Inject (No Service)

The view iframe provides a settings UI; inject.js provides the visual effect in the host DOM. They communicate via `postMessage` (view→parent→inject.js listener) and/or `localStorage` as shared state. The service is empty or absent.

```
view/index.html  ──postMessage──>  window.parent  ──message event──>  inject.js
     (iframe)    ──localStorage──>                 <──localStorage──   (host DOM)
```

**Examples:** plugin-anime-border, plugin-cursor-glow, plugin-font-switcher

**When to use:**
- The plugin has a visual effect in the host DOM AND a settings panel
- No Node.js capabilities are needed (no file I/O, no HTTP proxy)
- State can be persisted in localStorage

### 5. View + Inject + Service (Triple Layer)

All three layers are active. The view provides settings UI, inject.js provides host DOM effects, and the service provides Node.js capabilities (file persistence, API calls).

```
view/index.html  ──postMessage──>  Host App  ──pluginCallService──>  service.js
     (iframe)    ──postMessage──>  inject.js (host DOM)               (Node.js)
                 ──localStorage──> inject.js
```

**Examples:** plugin-weather, plugin-live2d

**When to use:**
- The plugin needs all three: settings UI, host DOM manipulation, AND Node.js capabilities
- Example: a weather widget with a config panel (view), a floating display (inject.js), and API/file access (service)

### 5b. Triple Layer with inject.js → service via CustomEvent

A variant of the Triple Layer where inject.js needs to call the service. inject.js cannot use `plugin-call` postMessage (host source verification rejects non-iframe sources — see `plugin-communication.md` Pitfall #9). Instead, the main framework provides a `plugin-call-service` CustomEvent listener that routes calls to `pluginCallService`.

```
inject.js ──plugin-call-service CustomEvent──> Main Framework ──pluginCallService──> service.js
  (host)   <──callback CustomEvent──                          <──{ ok, result }──    (Node.js)
  
Host App ──plugin-command-done──> inject.js (CustomEvent, context menu results)
         ──plugin-tree-updated──> inject.js + plugin-dropdown ──plugin-tree-update──> panel view
```

**Examples:** plugin-todo-dep

**Requires main framework support:** A `plugin-call-service` CustomEvent listener in `todo-tree/index.tsx` (see `plugin-communication.md` Pattern 11).

**When to use:**
- inject.js needs to read/write files, call service methods, or access Node.js APIs
- The host framework has the `plugin-call-service` listener (or you can add it)
- No head view needed — simpler than the hidden head view bridge approach

**Alternative (without main framework change):** Use a hidden head view iframe as bridge — see `plugin-communication.md` Pattern 8.

### 6. Client-Side Tree Analysis (Head/Topbar/Topfix View)

The view receives tree data via `plugin-tree-update` push messages and does all analysis client-side. The service is an empty stub. No `callPlugin` RPC needed.

```
Host App  ──plugin-init──>      view/index.html (iframe)
          ──plugin-tree-update──>   (client-side analysis)
          <──plugin-request-tree──
```

**Examples:** plugin-task-line (topfix), plugin-todo-warning (topbar)

**When to use:**
- The plugin only needs to READ and DISPLAY tree data
- Analysis is lightweight (counting, filtering, sorting)
- Avoiding `callPlugin` round-trip latency is important for responsive head/topbar indicators

### 7. Remote View (External URL)

The view points to an external URL, which opens in a separate BrowserWindow. No postMessage bridge, no sandbox.

```
package.json: "view": "https://example.com/app/"
──> Opens in separate BrowserWindow (no plugin bridge)
```

**Examples:** plugin-editor

**When to use:**
- Embedding an existing external web app
- No host integration needed beyond launching the window

## Quick Reference Table

| Architecture | view | inject.js | service | Use Case |
|-------------|------|-----------|---------|----------|
| View + Service | iframe UI | — | Node.js backend | Settings panel + file I/O, API calls, config |
| Service-Only | stub | — | Node.js backend | Context menu / paste event handlers, tree transforms |
| Inject-Only | stub | host DOM | stub | Visual effects, floating widgets, audio players |
| View + Inject | iframe UI | host DOM | — | Settings panel + visual effects (localStorage bridge) |
| View + Inject + Service | iframe UI | host DOM | Node.js backend | Full-stack: settings + effects + file/API access |
| Triple Layer + inject→service | panel (optional) | host DOM | Node.js backend | inject.js calls service via `plugin-call-service` CustomEvent or head view bridge |
| Client-Side Tree Analysis | head/topbar/topfix | — | stub | Display-only indicators from tree data |
| Remote View | external URL | — | — | Embed external web app in a BrowserWindow |

## Decision Flowchart

```
Does the plugin need a visible panel/settings UI?
├─ No: Does it need to modify tree data or respond to context menus?
│   ├─ Yes → Architecture 2: Service-Only
│   └─ No: Does it add visual effects to the host page?
│       ├─ Yes → Architecture 3: Inject-Only
│       └─ No → Probably not a plugin
└─ Yes: Does it add visual effects to the host page?
    ├─ No: Is it a head/topbar/topfix indicator that only reads tree data?
    │   ├─ Yes → Architecture 6: Client-Side Tree Analysis
    │   └─ No: Does it need Node.js capabilities?
    │       ├─ Yes → Architecture 1: View + Service
    │       └─ No → Simple view with localStorage persistence
    └─ Yes: Does it need Node.js capabilities?
        ├─ Yes: Does inject.js need to call service methods?
        │   ├─ Yes: Can you modify the main framework?
        │   │   ├─ Yes → Architecture 5b: Triple Layer + plugin-call-service CustomEvent
        │   │   └─ No → Architecture 5b: Triple Layer + Hidden Head View Bridge
        │   └─ No → Architecture 5: View + Inject + Service
        └─ No → Architecture 4: View + Inject
```
