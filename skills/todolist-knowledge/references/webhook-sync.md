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

## Agent Direct Read/Write (bypassing the app)

The board's `/api` is plain HTTP JSON — a script or agent can maintain the board without the app in the loop at all. The Verify curls above are the basic shape; these are the details that matter once you do it for real:

**1. `eventId` should mirror the file's `version`.** Pass `int(value.version)` (the store's timestamp string as a number), matching what the app sends. The example above uses `1` for a one-off test; for ongoing writes, a stale/regressed `eventId` can make the board treat your push as an older event than the file it already holds. Bump `version` to a fresh `Date.now()` timestamp on every edit and carry it into `eventId`.

**2. Send both `file` in the URL and `path` in params.** They name the same board-side file; keep them identical (`uid` too) wherever the file is referenced.

**3. `Store` is a full-snapshot overwrite, not a merge.** `params.value` must be the *entire* store object — `tree`, `expandKeys`, `tags`, `title`, `desc`, `version`, all of it. Pushing a partial object (say, just `tree`) writes a file missing the other fields and the board renders it broken or resets it. Read (`GetStore`) → mutate in memory → write back the whole store.

Round-trip recipe:

```bash
BOARD='http://<board-host>:<port>'; UID='data'; FILE='2026/W38.todo'

# read current store
curl -s -X POST "$BOARD/api?uid=$UID&file=$FILE" -H 'Content-Type: application/json' \
  -d '{"service":"GetStore","params":{"key":"todotree","path":"'"$FILE"'"}}'

# ...edit the JSON (keep it a complete store), set version = now-ms, then:
curl -s -X POST "$BOARD/api?uid=$UID&file=$FILE" -H 'Content-Type: application/json' \
  -d '{"service":"Store","params":{"key":"todotree","path":"'"$FILE"'","value":<entire-store>},"eventId":<version-as-number>}'
```

A `Store` whose `file`/`path` doesn't exist yet **creates** the board-side file and auto-points `currentKey` at it (see Notes) — that's the usual way a new period file goes live.

### Token economy: push with a local script, never inline the store

A full-snapshot `Store` push looks heavy but costs the agent **zero tokens** when done right: the payload travels script → board over HTTP and never enters the model's context. The anti-pattern is *inlining* the store — reading the whole JSON into the conversation, editing it there, and emitting the full payload as generated text. That pays for the same kilobytes twice (once reading, once writing), invites transcription errors, and still needs a tool call to send anyway.

The correct loop for an LLM agent:

- The model produces only the **edit intent** — a short local script (Python/Node/bash+curl) that reads the authoritative `.todo` file, applies the change, and POSTs the whole store.
- Verify by printing a **small summary** (a few nodes' `content`/`done`/`tags`), never `print(store)`.
- Don't `GetStore` the full tree into context to "check" — have the script extract just the fields you need (title, node count, a node by id).
- Keep the `.todo` files in a git repo as the source of truth. Then a wiped board (e.g. a redeployed container without a volume) is just "re-run the push script for every file" — push all files, then `saveConfig` to restore `currentKey` and explorer `expandedKeys`.

## Redeploying the Docker Board Wipes It — Prevention & Restore

**Why it wipes:** the board keeps everything *inside the container* — todo files under `/app/web/server/tree`, plus its `currentKey` / explorer state in the container's config. `docker run` a fresh container without a volume and the board comes up empty. (This is by design for a read-only board backed by a git repo: the repo is the data, the board is a projection.)

**Prevention:** mount a volume over the tree dir if you want the board itself to survive redeploys:

```bash
docker run -d -p 3000:3000 \
  -v /path/on/host/todolist-data:/app/web/server/tree \
  saber2pr/todolist-app:master
```

**Restore (board already wiped):** with the `.todo` files in a git repo as the source of truth, recovery is pure API calls — no backups needed:

1. **Re-push every `.todo` file** from the repo: for each file POST a `Store` with its full `todotree` and `eventId = version` (script it — see Token economy above; skip any file that shouldn't be pushed to this board, e.g. ones holding secrets).
2. Ordering only matters for the *last* push: the last newly-created file becomes `currentKey`. Push the file you want opened by default (this week's / the current one) last — or fix it afterwards in step 3.
3. **Restore board config**: POST `saveConfig` with `currentKey` (board-side full path, e.g. `/app/web/server/tree/2026/W38.todo`), `currentName`, `currentDir`, and `expandedKeys` — the absolute paths of every directory to auto-expand in the explorer sidebar.
4. **Verify**: `getConfig` shows the right `currentKey`/`expandedKeys`; a `GetStore` on the default file returns the expected `title`, root-node count and `version`.


- The Webhook setting is available in the VSCode extension and Desktop app, and is stored **per todo file** — switching the working file switches (or drops) the sync target. Configure it once per file you want synced.
- The board server allows cross-origin requests (`*`), so posts from the app work out of the box.
- Events also fire on reads (opening files, plugin/terminal activity) — the board handles them as no-ops; nothing to filter on your side.
- Keep `uid` and `file` identical wherever you reference the board file (webhook URL, board page URL, manual curls); the board keys stored data by `uid` + `file`.
- **currentKey auto-switch**: when a `Store` event writes a **newly created** board-side file, the server automatically points `currentKey`/`currentName` at it, so the web board opens the new file by default. To switch to an *existing* file later, POST a `saveConfig` with `params: { currentKey: "<board-side full path>", currentName: "<name>" }` (board-side paths look like `/app/web/server/tree/<file>`).
