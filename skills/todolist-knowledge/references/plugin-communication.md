# Plugin Communication Patterns — Full Reference

Patterns and techniques for plugin communication, extracted from 14 real-world plugins. For architecture type selection (View+Service, Inject-Only, etc.), see `plugin-architecture.md`.

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

### Pattern 8: Hidden Head View Bridge (inject.js → service, fallback approach)

> **Prefer Pattern 11 (`plugin-call-service` CustomEvent)** if you can modify the main framework. It's simpler and doesn't require a head view.

inject.js runs in the main window context, not in an iframe. The host's `plugin-call` handler at `usePluginContributes.ts:161-172` verifies `entries[i][1].contentWindow === source` — since inject.js's message source is `window` (not an iframe `contentWindow`), the host **rejects** all `plugin-call` messages from inject.js. This means inject.js cannot call service methods via the standard `callPlugin` RPC.

**Fallback solution (no main framework changes):** Use a hidden head view iframe (height 0, always loaded) as a relay:

```javascript
// In head view (bridge) — view/index.html in head mode
window.parent.postMessage({ type: 'tdep-bridge-ready' }, '*')

window.addEventListener('message', async (e) => {
  if (e.data?.type === 'tdep-save') {
    const result = await callPlugin('saveDeps', e.data)
    window.parent.postMessage({ type: 'tdep-save-result', result }, '*')
  }
})

// In inject.js — save bridgeWindow reference
let bridgeWindow = null
window.addEventListener('message', e => {
  if (e.data?.type === 'tdep-bridge-ready' && e.source && e.source !== window) {
    bridgeWindow = e.source
  }
})

// Later: send commands through the bridge
bridgeWindow.postMessage({ type: 'tdep-save', todoId: 123, depIds: [1, 2] }, '*')
```

**Key implementation details:**
- Check `e.source !== window` when receiving bridge-ready — inject.js's own messages also hit the listener
- The head view is always loaded (unlike panel views which only exist when opened), so the bridge is always available
- Use custom message types (e.g. `tdep-save`, not `plugin-call`) to avoid conflicts with the host bridge

### Pattern 9: CustomEvent Bridge for inject.js

inject.js cannot receive `postMessage` results from service calls (no iframe to route back to). For certain data flows, the main framework dispatches `CustomEvent` on `window` that inject.js can listen to.

**Context menu command results:**
```javascript
// Main framework dispatches after contextMenu command completes:
window.dispatchEvent(new CustomEvent('plugin-command-done', {
  detail: { pluginName, command, result }
}))

// inject.js listens:
window.addEventListener('plugin-command-done', e => {
  const d = e.detail
  if (d.command !== 'setDependency') return
  // Double-unwrap: pluginCallService wraps result
  const r = d.result?.result || d.result
  if (!r?.target) return
  showModal(r.target)
})
```

**Tree update notifications:**
```javascript
// Main framework dispatches after tree changes:
window.dispatchEvent(new CustomEvent('plugin-tree-updated'))

// plugin-dropdown forwards to panel iframe:
window.addEventListener('plugin-tree-updated', () => {
  iframeRef.current?.contentWindow?.postMessage({ type: 'plugin-tree-update' }, '*')
})
```

These CustomEvents require small additions to the main framework (2 `dispatchEvent` calls + 1 listener in `plugin-dropdown`).

### Pattern 10: Hover-Based Lazy DOM Decorations

Permanently injected DOM elements in todo items get wiped by React re-renders. MutationObserver + re-injection is fragile and creates flicker. Instead, show decorations only on hover using event delegation:

```javascript
let hoverWrap = null

document.addEventListener('mouseover', e => {
  const el = e.target.closest?.('[data-todowrapid]')
  if (el === hoverWrap) return
  if (hoverWrap) { removeTag(hoverWrap); hoverWrap = null }
  if (!el) return
  hoverWrap = el
  showTag(el)
})

document.addEventListener('mouseout', e => {
  if (!hoverWrap) return
  const related = e.relatedTarget
  if (related && hoverWrap.contains(related)) return
  removeTag(hoverWrap)
  hoverWrap = null
})

function showTag(wrapEl) {
  const textEl = wrapEl.querySelector('[data-todoid]')
  if (!textEl || textEl.querySelector('.my-tag')) return
  const tag = document.createElement('span')
  tag.className = 'my-tag'
  tag.textContent = '...'
  textEl.appendChild(tag)
}
```

**Why this works:**
- No permanent DOM mutation → immune to React re-renders
- Event delegation on `document` → works for dynamically added todo items
- `[data-todowrapid]` is the row-level `<Space>` wrapper, `[data-todoid]` is the inner `<Text>` element

**Data source:** Cache dep/decoration data in a JS variable (populated via `plugin-call-service` CustomEvent or `localStorage`) and update on `plugin-tree-updated`. The hover handler reads from this cache — no async calls during hover.

### Pattern 11: `plugin-call-service` CustomEvent (inject.js → service, preferred)

A generic mechanism for inject.js to call any plugin service method via CustomEvent, without needing a head view bridge. Requires a one-time addition to the main framework.

**Main framework side** (`todo-tree/index.tsx`):
```javascript
useEffect(() => {
  const handler = async (e) => {
    const { pluginName, method, params, callbackEvent } = e.detail || {}
    if (!pluginName || !method) return
    try {
      const result = await callService('pluginCallService', { name: pluginName, method, params })
      if (callbackEvent) window.dispatchEvent(new CustomEvent(callbackEvent, { detail: result }))
    } catch (err) {
      if (callbackEvent) window.dispatchEvent(new CustomEvent(callbackEvent, { detail: { ok: false, error: String(err) } }))
    }
  }
  window.addEventListener('plugin-call-service', handler)
  return () => window.removeEventListener('plugin-call-service', handler)
}, [])
```

**inject.js side:**
```javascript
const PLUGIN_NAME = '@aicupa/plugin-my-plugin'
let callSeq = 0

function callPluginService(method, params) {
  return new Promise((resolve, reject) => {
    const cbEvent = 'my-cb-' + (++callSeq)
    const timer = setTimeout(() => { window.removeEventListener(cbEvent, handler); reject(new Error('timeout')) }, 5000)
    const handler = (e) => {
      clearTimeout(timer)
      window.removeEventListener(cbEvent, handler)
      const d = e.detail
      if (d?.ok === false) reject(new Error(d.error || 'failed'))
      else resolve(d)
    }
    window.addEventListener(cbEvent, handler)
    window.dispatchEvent(new CustomEvent('plugin-call-service', {
      detail: { pluginName: PLUGIN_NAME, method, params, callbackEvent: cbEvent },
    }))
  })
}
```

**Key details:**
- Each call gets a unique `callbackEvent` name to correlate request/response
- Result is double-wrapped by `pluginCallService` — use the same `unwrap()` helper
- Generic: works for any plugin, any service method — not coupled to a specific plugin
- No head view, no bridge iframe, no `postMessage` source verification issues

**When to use over Pattern 8 (head view bridge):**
- You can add the `plugin-call-service` listener to the main framework
- Simpler architecture — no hidden iframe, no bridge-ready handshake, no `postMessage` relay

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

This is why **disabling** or **uninstalling** a plugin requires a full app reload (`callService('reload', {})`) — the injected `viewScripts` (`<script>` tags in `document.head`) and their side effects cannot be cleaned up by navigation alone. See `references/plugin-api.md` → "Plugin Lifecycle & Reload Behavior" for the full matrix.

### 7. Duplicated Logic Between View and Service

Some plugins (plugin-todo-warning) implement the same analysis in both view and service. Choose one location based on whether the analysis needs Node.js capabilities or is display-only.

### 8. CDN Dependencies in inject.js

Loading scripts from CDNs at runtime (Tone.js, Live2D) means the plugin breaks offline. Bundle critical dependencies or use `api.fetch()` through the service with caching.

### 9. inject.js Cannot Call Service Directly

The host's `plugin-call` handler (`usePluginContributes.ts:161-172`) checks `entries[i][1].contentWindow === source` to verify the message came from a registered iframe. inject.js runs in the main window context — its `postMessage` source is `window`, which fails this check. All `plugin-call` messages from inject.js are silently dropped.

**Solution (preferred):** Use `plugin-call-service` CustomEvent (Pattern 11) — requires adding a listener to the main framework, but no head view needed.

**Solution (no framework changes):** Use a hidden head view iframe as a bridge (Pattern 8).

### 10. React Re-Renders Wipe Permanent DOM Injections

Any DOM elements injected into todo item rows by inject.js will be destroyed when React re-renders the todo tree (on add, edit, delete, reorder, or filter changes). MutationObserver-based re-injection is fragile — it creates flicker and race conditions with React's reconciliation.

**Solution:** Use hover-based lazy decorations (Pattern 10) instead of permanent injection. Show decorations only when the user hovers, and remove them on mouseout. This is immune to React re-renders because no persistent DOM state is maintained.

### 11. Head View Timing — plugin-tree-update Before plugin-init

Head views may receive `plugin-tree-update` before `plugin-init`. If your head view needs `filePath` from `plugin-init` (e.g., to call `scanAllDeps`), the first `plugin-tree-update` will have an empty `filePath`. The second `plugin-tree-update` (which arrives after `plugin-init` completes) will work correctly.

**Workaround:** Guard service calls on `currentFilePath` being non-empty:
```javascript
if (msg.type === 'plugin-tree-update') {
  if (!currentFilePath) return  // skip until plugin-init provides filePath
  await scanAndSync()
}
```

### 12. Plugin File Sync During Development

Plugin source at the dev path (e.g., `/Users/you/github/plugin-foo/`) is NOT what the app reads. The app loads plugins from `~/.todoListNative/plugins/<plugin-name>/`. After editing source files, you must:
1. Copy files: `cp -r ./service.js ./inject.js ./view ./package.json ~/.todoListNative/plugins/@aicupa/plugin-foo/`
2. Reload or restart the app. Navigating away from the todolist page and back re-fetches contributes and service caches, but injected `viewScripts` require a full reload (`callService('reload', {})`) to pick up changes.

Forgetting either step means your changes won't take effect and debugging will show stale behavior.

### 13. Context Menu `node` May Not Preserve Custom Fields

The `node` parameter passed to contextMenu service methods comes from the host's React state, which may strip custom fields (e.g., `depIds`) that the plugin added to the `.todo` file. The host deserializes tree data and may only keep known fields in memory.

**Problem:** `node.todo.depIds` is `undefined` even though `depIds` exists in the `.todo` file.

**Solution:** Always read custom fields from `api.getTree()` (which reads from the file/store), not from the `node` parameter:
```javascript
async setDependency({ node, filePath }) {
  const data = await api.getTree(filePath)
  const allTodos = []
  flattenTodos(getTreeNodes(data), allTodos)
  const targetInTree = allTodos.find(t => t.id === node.todo.id)
  return {
    ok: true,
    target: {
      id: node.todo.id,
      content: targetInTree?.content || node.todo.content,
      depIds: targetInTree?.depIds || [],  // from tree data, NOT node.todo.depIds
    },
  }
}
```

