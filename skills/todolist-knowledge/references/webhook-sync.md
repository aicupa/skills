# Webhook & Sync-to-Web — Full Reference

How the Todolist webhook (Settings → Webhook) works: which events are emitted, with which params, when you edit todos or change expansion — and how to bridge them into a local mirror file.

## Webhook Mechanism

### Where it is configured

Settings modal (`web/src/components/settings-modal/index.tsx`) → **Webhook** field (`<Input type="url">`). The field only renders when `isInVscode || isNative`. The value is persisted **per todo file** in the store (`IStoreTodoTree.webhook`, see `src/api/type.ts`), not in the global app config.

### When it activates

`web/src/pages/todo-tree/index.tsx` (effect keyed on `webhook`):

```tsx
if (isInVscode || isNative) {
  if (webhook) {
    mockVscodeApi.hook = (message) => axios.post(webhook, message)
  } else {
    mockVscodeApi.hook = null
  }
}
```

Constraints:

- Only fires in the **VSCode extension and Desktop (native)** builds. The pure-browser web app never sends webhook events.
- The POST goes out from the webview/native renderer — the receiver must allow cross-origin (`Access-Control-Allow-Origin: *`), or the request dies silently in the console.

### What triggers a POST

The hook lives inside `@saber2pr/vscode-webview` (`lib/core/callService.js`). **Every** `callService(service, params)` call emits one POST — including reads like `GetStore`/`getConfig`. Event shape:

```json
{
  "service": "Store",
  "params": { "key": "todotree", "value": { "...store snapshot...": true }, "path": "/home/xxx/.todo" },
  "eventId": 123
}
```

The local-server host (`web/server/app.js` `POST /api` → `servicesMock`) receives the **same message body** through the main path; it never sends webhooks itself. The webhook is a parallel fan-out added in the front-end callService layer.

## Event Catalog (what actually fires)

Full `callService` inventory is 60+ services, but only these matter in practice. Grouped by user action:

### Editing todo content / expanding tree nodes (debounced save)

Every debounced `save()` emits **two** events in order:

| # | service | params |
|---|---------|--------|
| 1 | `Store` | `{ key: 'todotree', value: <full store snapshot>, path, currentDir }` |
| 2 | `pushBackupVersion` | `{ file, version, content: { todotree: {...} }, maxSize }` |

Expanding/collapsing a **tree node** mutates `value.expandKeys` and goes through the **same** debounced save → same `Store` event. There is no separate "expand" event.

### Sidebar explorer operations

| service | params | when |
|---------|--------|------|
| `saveConfig` | `{ expandedKeys: [abs paths], currentDir, ...config }` | explorer folder expand/collapse, theme/lang change, … |
| `readdir` | `{ path }` | directory refresh / navigation |
| `writeFile` | `{ path, content }` / `{ dir, filename, content }` | create/save file from sidebar |
| `mkdir` / `rename` / `remove` | `{ path }`-ish | file management |
| `readFile` | `{ path }` | open a file |

### Initialization noise (ignore these)

`GetStore`, `getConfig`, `GetToken`, `parsePath`, `watchFile`, `getBackupVersionList`, `plugin*`, `pty*`, terminal/shortcut APIs — all fire on load or on plugin/terminal activity, all read-only. A sync receiver should filter them out.

### Store event payload

`params.value` is the full `IStoreTodoTree` store — the same object written into the `.todo` file:

| Field | Meaning |
|-------|---------|
| `tree` | Full todo tree (`ITodoTree[]`) — the todo data |
| `expandKeys` | Expanded **tree node** keys — board expansion state |
| `title`, `tags`, `add_mode`, `autoSort`, … | View settings (see `todo-format.md`) |
| `version` | `String(Date.now())` — use for ordering/dedup |
| `webhook` | The webhook URL itself (echoed back) |

### saveConfig event payload (explorer state)

The sidebar **file explorer** expansion goes through a separate `saveConfig` event (the local server merges `expandedKeys` — resolved to absolute paths — into `todolist.config.json`, see `web/server/app.js` `saveConfig`):

```json
{
  "service": "saveConfig",
  "params": {
    "expandedKeys": ["/home/xxx/2026", "/home/xxx/work"],
    "currentDir": "/home/xxx"
  },
  "eventId": 456
}
```

Do NOT confuse the two expandKeys:

| Field | Event | Meaning |
|-------|-------|---------|
| `params.value.expandKeys` | `Store` | Todo **tree node** expansion (inside the board) |
| `params.expandedKeys` | `saveConfig` | **File explorer sidebar** expansion (absolute paths, global config) |

## Bridge: consume webhook events locally

Run a local HTTP receiver that consumes webhook events and mirrors the snapshot to a `.todo` file (zero-dependency node):

```js
// webhook-sync.js — node webhook-sync.js
const http = require('http')
const fs = require('fs')

const OUT = '/home/xxx/synced/todo.todo' // open with the Desktop app / local-server mode

http
  .createServer((req, res) => {
    res.setHeader('Access-Control-Allow-Origin', '*') // webview sends cross-origin POST
    res.setHeader('Access-Control-Allow-Headers', '*')
    if (req.method === 'OPTIONS') return res.end() // preflight

    let body = ''
    req.on('data', (c) => (body += c))
    req.on('end', () => {
      try {
        const { service, params } = JSON.parse(body)
        if (service === 'Store' && params.key === 'todotree') {
          // write in .todo file format: single root key `todotree`
          fs.writeFileSync(OUT, JSON.stringify({ todotree: params.value }, null, 2))
        }
        if (service === 'saveConfig') {
          // explorer state — merge into the same mirror dir's config if needed
          fs.writeFileSync(OUT + '.config.json', JSON.stringify({ config: params }, null, 2))
        }
      } catch {}
      res.end()
    })
  })
  .listen(9750, () => console.log('listening :9750'))
```

Point the Webhook setting at `http://127.0.0.1:9750`. Keep the single-root-key format `{ "todotree": {...} }` so the Desktop app / local-server mode (`web/server/server.js` serves `/api`) can open the mirror directly.

## Pitfalls

- Webhook fires for **every** `callService`, not only saves — always filter on `service` (see Event Catalog).
- Each `.todo` file carries its own `webhook` setting; switching display files switches (or drops) the receiver.
- `params.value.version` is `Date.now()` as a string — compare numerically after `Number()`.
- The browser build never emits webhook events; do not build a sync flow that assumes it does.
- Remember `expandKeys` reference tree node keys; after re-keying nodes (copy/move), stale keys are filtered on load (`filterBindingKeysFromTree`), so explorer state may reset — that is expected.
