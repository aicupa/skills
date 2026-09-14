# Webhook Sync — Push Local Todos to a Self-Hosted Web Board

How to sync todos from the local Todolist app to a self-hosted web board (todolist-app docker image) using the Settings **Webhook**. The board image ships a node server that accepts the app's webhook messages directly — no extra bridge or receiver needed, just point the webhook at the board's API.

## Setup

1. Deploy the todolist-app docker image and note the board address, e.g. `http://board.local:3000`.
2. In the Todolist app, open **Settings → Webhook** and fill in the board's API endpoint:

   ```
   http://<board-host>:<port>/api?uid=<uid>&file=<board-side-todo-file>
   ```

   - `uid` — storage namespace on the board, letters/digits/`_`/`-` only; defaults to `data`.
   - `file` — the todo file path **on the board side** (e.g. `todo.work`). Synced data lands in this file; open the same file in the board to view the kanban.
3. Edit a todo and save. The board-side file updates shortly after (auto-save debounce, ~1s); refresh the board page to see it.

## What Gets Sent

The app POSTs every app→server call to the webhook URL as JSON:

```json
{ "service": "Store", "params": { "key": "todotree", "value": { "...": "..." } }, "eventId": 123 }
```

The board server dispatches by `service`. Only write-type events have effects:

| Event | Params (essentials) | Effect on the board |
|-------|---------------------|---------------------|
| `Store` | `{ key: 'todotree', value: { tree, expandKeys, version, ... }, path }` | Writes the full todo snapshot (todo tree + node expansion state) to the board-side file |
| `saveConfig` | `{ expandedKeys: [...], currentDir, ... }` | Merges explorer-sidebar expansion and global config |
| `writeFile` / `mkdir` / `rename` / `remove` | `{ path, content? }` | Applies the same file operation on the board side |
| `GetStore` / `getConfig` / `readdir` / `readFile` / plugin & terminal APIs | — | Read-only or no-op; harmless, ignore them |

Two different expansion states exist — don't mix them up:

| Field | Event | Controls |
|-------|-------|----------|
| `value.expandKeys` | `Store` | todo **tree node** expansion (inside the board) |
| `expandedKeys` | `saveConfig` | **file explorer sidebar** expansion (absolute paths) |

`value.version` is a `Date.now()` string stamped on every save — use it to order/dedup snapshots if you process events yourself.

## Verify

From the machine running the app:

```bash
# push a snapshot
curl -X POST 'http://<board-host>:<port>/api?uid=data&file=todo.work' \
  -H 'Content-Type: application/json' \
  -d '{"service":"Store","params":{"key":"todotree","value":{"tree":[],"expandKeys":[],"version":"1"}},"eventId":1}'

# read it back
curl -X POST 'http://<board-host>:<port>/api?uid=data&file=todo.work' \
  -H 'Content-Type: application/json' \
  -d '{"service":"GetStore","params":{"key":"todotree","path":"todo.work"}}'
```

If both curls return and the second echoes the snapshot, the webhook URL is correct — paste the same URL into Settings → Webhook.

## Notes

- The Webhook setting is available in the VSCode extension and Desktop app, and is stored **per todo file** — switching the working file switches (or drops) the sync target. Configure it once per file you want synced.
- The board server allows cross-origin requests (`*`), so posts from the app work out of the box.
- Events also fire on reads (opening files, plugin/terminal activity) — the board handles them as no-ops; nothing to filter on your side.
- Keep `uid` and `file` identical wherever you reference the board file (webhook URL, board page URL, manual curls); the board keys stored data by `uid` + `file`.
