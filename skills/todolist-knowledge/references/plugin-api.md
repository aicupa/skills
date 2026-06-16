# Plugin API — Full Reference

## Service Pattern

The `api` object is injected into the service factory. Use `createPlugin` from `@aicupa/api` for type hints:

```javascript
const { createPlugin } = require('@aicupa/api')

module.exports = createPlugin((api) => {
  return {
    async myMethod(params) {
      const tree = await api.getTree(params.filePath)
      return { ok: true, result: tree }
    }
  }
})
```

`createPlugin` is an identity function at runtime — it only provides TypeScript/IDE inference. Plain export also works:

```javascript
module.exports = function (api) {
  return {
    async myMethod(params) { return { ok: true } }
  }
}
```

## API Methods

| API | Description |
|-----|-------------|
| `api.reload(filePath?)` | Refresh the todolist view |
| `api.getTree(filePath?)` | Get the todolist JSON tree data |
| `api.store(key, value, filePath?)` | Save data to the todolist store |
| `api.relaunch()` | Restart the Electron app |
| `api.setTitle(title, filePath?)` | Set the window title |
| `api.getConfig()` | Get app configuration |
| `api.readFile(path)` | Read a file (returns string) |
| `api.writeFile(path, content)` | Write content to a file |
| `api.readdir(dir)` | List files in a directory |
| `api.mkdir(dir)` | Create a directory (recursive) |
| `api.remove(path)` | Remove a file or directory |
| `api.pathJoin(...args)` | Join path segments |
| `api.mapTree(tree, fn)` | Recursively map over a tree structure |
| `api.getArray(val)` | Safely convert a value to an array |
| `api.base64.encode(str)` | Encode string to base64 |
| `api.base64.decode(str)` | Decode base64 to string |
| `api.clipboard.writeText(text)` | Write text to clipboard |
| `api.clipboard.readText()` | Read text from clipboard |
| `api.isWindows` | `true` if running on Windows |

## Return Format

Service methods should return:
- Success: `{ ok: true, result: ... }`
- Failure: `{ ok: false, error: "message" }`

## pluginContributes — Full Reference

### contextMenus

Register items in the tree node right-click menu:

```json
"contextMenus": [
  { "title": "Cut", "title_zh": "剪切", "command": "cutNode" }
]
```

| Field | Required | Description |
|-------|----------|-------------|
| `title` | Yes | Menu item label (English) |
| `title_zh` | No | Chinese label, falls back to `title` |
| `command` | Yes | Service method name. Receives `{ node, filePath }` |

The `node` parameter contains `{ key, children, todo: { id, content, done, level, ... } }`.

### events.paste

Intercept paste actions on tree nodes:

```json
"events": {
  "paste": { "check": "getCutBuffer", "handler": "pasteNode" }
}
```

| Field | Description |
|-------|-------------|
| `check` | Service method to check if plugin has pending data. Return `{ ok: true, result: { node: ... } }` if active, `{ ok: true, result: { node: null } }` if not |
| `handler` | Service method to perform paste. Receives `{ targetNode, filePath }`. Tree reloads automatically after success |

### views.head

Render plugin view at the top of the todolist, above the progress bar:

```json
"views": { "head": true }
```

The view is loaded as an inline iframe with auto-height. It receives tree data on every update and supports the full `plugin-call` / `plugin-result` protocol.

## Installation & Storage

- **npm**: Plugins with `@aicupa/plugin-` prefix can be searched/installed from the Plugin Marketplace
- **Local**: Plugin icon → Plugin Market → Install from local → select directory
- **Storage**: `~/.todoListNative/plugins/` (plugins), `~/.todoListNative/plugins.json` (registry)
- `@aicupa/api` is auto-provisioned at `~/.todoListNative/plugins/node_modules/@aicupa/api/`
