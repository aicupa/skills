# Plugin Communication Patterns — Full Reference

Patterns and techniques extracted from 14 real-world plugins. Use this reference when building plugin communication logic, choosing an architecture, or debugging message flow.

## Architecture Types

Every plugin falls into one of these archetypes. Choose based on what the plugin needs to do.

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

## Communication Patterns

### Pattern 1: callPlugin RPC

The standard pattern for view→service communication. Implements promise-based RPC over postMessage with correlation IDs.

```javascript
let callId = 0
const pending = {}

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

window.addEventListener('message', (event) => {
  const msg = event.data
  if (msg?.type === 'plugin-result' && pending[msg.id]) {
    const { resolve, reject } = pending[msg.id]
    delete pending[msg.id]
    msg.error ? reject(new Error(msg.error)) : resolve(msg.result)
  }
})
```

**Critical: always include a timeout.** Without it, promises hang forever if the host or service isn't ready. 2 seconds is the recommended default; use 5 seconds for calls that proxy external API requests.

**Observed timeout usage across plugins:**
- 2s: plugin-auth-msc, plugin-todo-warning, todolist-plugin-demo
- 5s: plugin-background-images
- No timeout (bug): plugin-balance — its promises can hang forever

### Pattern 2: Double-Unwrap Results

The service returns `{ ok, result }`, and the host's `pluginCallService` wraps it again. View code must double-unwrap:

```javascript
function unwrap(res) {
  const inner = res?.result || res
  return inner?.result !== undefined ? inner.result : inner
}

// Usage
const data = unwrap(await callPlugin('myMethod', params))
```

**This is the #1 pitfall.** Every plugin that uses `callPlugin` must handle this. The `plugin-background-images` plugin abstracts it cleanly into an `unwrap()` helper — follow that pattern.

### Pattern 3: View-to-Inject Direct Messaging

When both view and inject.js exist, the view can send commands to inject.js via `postMessage` without going through the service. inject.js listens on the host window's `message` event.

```javascript
// In view/index.html (iframe)
window.parent.postMessage({ type: 'plugin-call', method: 'switchEffect', fx: 'sakura' }, '*')

// In inject.js (host context)
window.addEventListener('message', (e) => {
  const msg = e.data
  if (msg?.type === 'plugin-call' && msg?.method === 'switchEffect') {
    applyEffect(msg.fx)
  }
})
```

**Note:** This reuses the `plugin-call` message type but skips the correlation ID and `params` fields. The host's bridge may also intercept these messages, so if the service has a method with the same name, both will be called. Use distinct method names.

### Pattern 4: localStorage as Cross-Context Bridge

`localStorage` provides a synchronous, persistent shared state channel between the view iframe and inject.js. Both run under the same origin in the Electron app.

```javascript
// In view/index.html — write state
function applyFont(preset) {
  localStorage.setItem('active_font_preset', preset)
}

// In inject.js — read on init + listen for changes
let activePreset = localStorage.getItem('active_font_preset') || 'system'
applyPreset(activePreset)

window.addEventListener('storage', (e) => {
  if (e.key === 'active_font_preset') {
    applyPreset(e.newValue)
  }
})
```

**Best practice — dual write:** Write to localStorage AND send postMessage simultaneously. localStorage provides persistence across reloads; postMessage provides real-time update to inject.js in the same tab. The `storage` event only fires in OTHER tabs/windows, not the same one.

```javascript
// In view/index.html — dual write pattern
function switchEffect(type) {
  localStorage.setItem('my_effect', type)  // persist for reload
  window.parent.postMessage({ type: 'plugin-call', method: 'syncEffect', fx: type }, '*')  // real-time
}
```

**Namespace your keys** to avoid collisions with the host app or other plugins. Use a `pluginName_` prefix (e.g., `aicupa_weather_pos`, `active_font_preset`).

### Pattern 5: Push-Based Tree Updates (Head/Topbar/Topfix Views)

Head, topbar, and topfix views receive tree data automatically on every change. Use this for client-side analysis instead of calling the service.

```javascript
window.addEventListener('message', (event) => {
  const msg = event.data
  if (msg?.type === 'plugin-init') {
    applyTheme(msg.theme)
    // Request initial tree data
    window.parent.postMessage({ type: 'plugin-request-tree' }, '*')
  }
  if (msg?.type === 'plugin-tree-update') {
    const stats = analyzeTree(msg.tree || [])
    render(stats)
  }
})
```

**Lifecycle:**
1. Host sends `plugin-init` with `filePath`, `theme`, optionally `lang`
2. View sends `plugin-request-tree` to get initial data
3. Host sends `plugin-tree-update` with current tree
4. On every subsequent tree change, host sends another `plugin-tree-update`

**When to use client-side vs service-side analysis:**
- Client-side: counting, filtering, sorting, display-only stats — avoids round-trip latency
- Service-side: cross-file queries, file I/O, clipboard, heavy computation

**Caveat (duplicated logic):** plugin-todo-warning implements analysis in BOTH view and service. This creates a maintenance risk — threshold changes must be updated in both places. Prefer one location: client-side for display-only views, service-side if other consumers need the analysis.

### Pattern 6: Service State Patterns

**In-memory state (closure variable):**
```javascript
module.exports = (api) => {
  let cutBuffer = null  // lives in closure, survives across calls, lost on restart
  return {
    cutNode({ node }) { cutBuffer = { node }; return { ok: true } },
    getCutBuffer() { return { ok: true, result: { node: cutBuffer?.node || null } } }
  }
}
```

**File-based persistence:**
```javascript
module.exports = (api) => {
  const configPath = api.path.join(api.os.homedir(), '.todoListNative', 'plugins', 'my-config.json')
  return {
    async getConfig() {
      try {
        const data = JSON.parse(await api.readFile(configPath))
        return { ok: true, result: data }
      } catch { return { ok: true, result: {} } }
    },
    async saveConfig(params) {
      await api.writeFile(configPath, JSON.stringify(params))
      return { ok: true }
    }
  }
}
```

**Using `api.store()`:**
```javascript
await api.store('todotree', modifiedTree, filePath)  // persist to .todo file
await api.reload(filePath)  // refresh host UI
```

### Pattern 7: Graceful Degradation with localStorage Fallback

When the service might not be ready, use localStorage as a synchronous fallback:

```javascript
async function loadData() {
  try {
    const res = await callPlugin('getData', {})
    const data = unwrap(res)
    if (data) {
      localStorage.setItem('my_cache', JSON.stringify(data))  // cache for fallback
      return data
    }
  } catch {}
  // Fallback to localStorage cache
  try { return JSON.parse(localStorage.getItem('my_cache')) } catch {}
  return null
}
```

**Example:** plugin-auth-msc uses this pattern — tries the service first, falls back to encrypted localStorage if the service isn't ready.

## inject.js Patterns

### Singleton Guard

Prevent double-initialization when the script is re-injected:

```javascript
;(function() {
  if (window.__MY_PLUGIN_LOADED__) return
  window.__MY_PLUGIN_LOADED__ = true
  // ... plugin code
})()
```

**All inject.js plugins use this pattern.** The host may re-inject scripts on navigation or plugin reload.

### DOM Injection with Deduplication

```javascript
function renderWidget() {
  if (document.getElementById('my-widget')) return  // already exists
  const el = document.createElement('div')
  el.id = 'my-widget'
  el.style.cssText = 'position:fixed; z-index:2147483647; pointer-events:none;'
  document.body.appendChild(el)
}
```

**Key techniques:**
- Use `position: fixed` with high `z-index` for overlays
- Use `pointer-events: none` on the container, re-enable on interactive children
- Check `getElementById` before creating to prevent duplicates

### MutationObserver for SPA Resilience

Re-create DOM elements if the host SPA navigation destroys them:

```javascript
if (!window.__MY_OBSERVER_ATTACHED__) {
  window.__MY_OBSERVER_ATTACHED__ = true
  new MutationObserver(() => {
    if (!document.getElementById('my-widget')) renderWidget()
  }).observe(document.body, { childList: true })
}
```

**Used by:** plugin-weather, plugin-cursor-glow

### Style Injection

Inject CSS into the host page:

```javascript
function injectStyles() {
  if (document.getElementById('my-plugin-styles')) return
  const style = document.createElement('style')
  style.id = 'my-plugin-styles'
  style.textContent = `
    #my-widget { /* ... */ }
    @keyframes myAnimation { /* ... */ }
  `
  document.head.appendChild(style)
}
```

### Draggable Floating Widget

A recurring pattern for floating UI elements (weather, music player):

```javascript
let isDragging = false, startX, startY, origX, origY

el.addEventListener('mousedown', (e) => {
  isDragging = true
  startX = e.clientX; startY = e.clientY
  origX = el.offsetLeft; origY = el.offsetTop
})

document.addEventListener('mousemove', (e) => {
  if (!isDragging) return
  el.style.left = (origX + e.clientX - startX) + 'px'
  el.style.top = (origY + e.clientY - startY) + 'px'
})

document.addEventListener('mouseup', () => {
  if (!isDragging) return
  isDragging = false
  localStorage.setItem('my_widget_pos', JSON.stringify({ left: el.style.left, top: el.style.top }))
})
```

**Persist position to localStorage** so the widget stays where the user placed it across reloads.

## Common Pitfalls

### 1. Missing callPlugin Timeout

Without a timeout, promises hang forever if the host isn't ready. **Always add a timeout.**

### 2. Double-Unwrap Forgotten

Service returns `{ ok, result }`, host wraps again. Access `res.result.result` or use an `unwrap()` helper.

### 3. setInterval/setTimeout Leaks in inject.js

`setInterval` timers are never cleared when the plugin is disabled. There is no teardown lifecycle hook. Minimize long-running timers; prefer `requestAnimationFrame` for animations.

### 4. postMessage with '*' Origin

All plugins use `'*'` as the target origin. This is acceptable in the controlled Electron environment but would be a security concern in a web context. No plugins validate `event.origin` on incoming messages.

### 5. Inconsistent Message Shape for View→Inject

Some plugins send non-standard `plugin-call` messages to inject.js (missing `id`, using `effect` instead of `params`). This can conflict with the host's bridge. Use distinct method names and consider using a different `type` field (e.g., `plugin-inject-call`) if the message is only for inject.js.

### 6. No Cleanup/Teardown

No plugin implements a destroy or cleanup function. DOM elements, event listeners, timers, and MutationObservers persist indefinitely. This is a platform limitation — the host does not provide a teardown lifecycle hook.

### 7. Duplicated Logic Between View and Service

Some plugins (plugin-todo-warning) implement the same analysis in both view and service. Choose one location based on whether the analysis needs Node.js capabilities or is display-only.

### 8. CDN Dependencies in inject.js

Loading scripts from CDNs at runtime (Tone.js, Live2D) means the plugin breaks offline. Bundle critical dependencies or use `api.fetch()` through the service with caching.

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
        ├─ Yes → Architecture 5: View + Inject + Service
        └─ No → Architecture 4: View + Inject
```
