# Plugin View Protocol — Full Reference

## View Modes

- **Local directory** (`"view": "./view"`) — loads `view/index.html` in a sandboxed `srcDoc` iframe with `postMessage` bridge
- **URL** (`"view": "https://example.com/app/"`) — loads URL in a separate BrowserWindow. No sandbox, no `postMessage` bridge — useful for embedding external web apps

## Message Protocol

### View → Host: Call Service Method

```json
{ "type": "plugin-call", "id": 1, "method": "myMethod", "params": {} }
```

### Host → View: Init (on load)

```json
{ "type": "plugin-init", "filePath": "/path/to/file", "theme": { "color": "#333", "backgroundColor": "#fff" } }
```

### Host → View: Service Result

```json
{ "type": "plugin-result", "id": 1, "result": {} }
```

### Host → View: Tree Update (head views only)

Sent automatically whenever the todolist tree changes:
```json
{ "type": "plugin-tree-update", "tree": [...] }
```

### View → Host: Request Tree (head views only)

```json
{ "type": "plugin-request-tree" }
```

Host responds with a `plugin-tree-update` message containing the current tree.

## Complete View Template

```html
<script>
  let callId = 0
  const pending = {}
  let currentFilePath = ''

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
    if (msg?.type === 'plugin-init') {
      currentFilePath = msg.filePath || ''
      if (msg.theme) document.body.style.color = msg.theme.color
      return
    }
    if (msg?.type === 'plugin-result' && pending[msg.id]) {
      const { resolve, reject } = pending[msg.id]
      delete pending[msg.id]
      msg.error ? reject(new Error(msg.error)) : resolve(msg.result)
    }
    if (msg?.type === 'plugin-tree-update') {
      // Handle tree data update (head views)
    }
  })
</script>
```

## Pitfalls

### Timeout on callPlugin

The `callPlugin` function uses `postMessage`. If the host isn't ready when the message is sent (e.g., `plugin-init` hasn't fired, or service hasn't initialized), the `plugin-result` response never arrives — the Promise hangs forever and silently blocks all downstream logic.

Always add a timeout (2 seconds recommended).

### localStorage as Fallback

Because `callPlugin` can fail silently, don't rely solely on the service for view-side data. Write to `localStorage` immediately as a synchronous backup:

```javascript
function saveData(val) {
  localStorage.setItem('my_key', val)               // instant, reliable
  callPlugin('save', { value: val }).catch(() => {}) // async, best-effort
}

async function loadData() {
  try {
    const res = await callPlugin('load', {})
    if (res?.result?.value) return res.result.value
  } catch {}
  return localStorage.getItem('my_key')
}
```

### Double-Unwrap Results

The service returns `{ ok, result }`, and `pluginCallService` wraps it again. View code needs double unwrap:

```javascript
const inner = res?.result || res
const data = inner?.result || inner
```

### Head View Auto-Height

Set `html, body { height: auto; overflow: hidden }` so the host iframe can measure `body.offsetHeight` correctly via `MutationObserver`.

### Theme Adaptation

Read `theme.color` and `theme.backgroundColor` from the `plugin-init` message and apply to your view elements for dark/light mode support.
