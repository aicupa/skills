# App Config — Full Reference

## Config File (`~/.todoListNative.json`)

The desktop app stores global configuration in `~/.todoListNative.json`. Read via `getConfig()`, write via `saveConfig(partialConfig)` (shallow merge).

```json
{
  "currentDir": "/Users/xxx/my-todos",
  "currentKey": "/Users/xxx/my-todos/work.todo",
  "currentName": "work",
  "recent": [{ "path": "/path/to/file.todo", "name": "file" }],
  "expandedKeys": ["/Users/xxx/my-todos/2026"],
  "hideAside": false,
  "asidePos": 261,
  "winSize": [1251, 810],
  "lang": "zh-cn",
  "theme": "dark",
  "debug": true,
  "token": "cloud-sync-token",
  "backgroundImage": "https://example.com/bg.jpg",
  "backgroundOp": "8",
  "backgroundSize": "cover",
  "backupSize": "9.5",
  "workPendTimeMax": "9.5",
  "noticeKey": "20260617",
  "dailyInfo": { "date": "06-18", "start": 1781748208423 },
  "frameUrl": "https://...",
  "aiCommand": "clix chat \"$cmd\"",
  "bottomHtmlTool": "...",
  "enableFloatingBall": true,
  "chatVisible": false
}
```

| Field | Type | Description |
|-------|------|-------------|
| `currentDir` | `string` | Active workspace folder path. Defaults to `~/.todoListNative` if empty |
| `currentKey` | `string` | Full path of the currently open `.todo` file |
| `currentName` | `string` | Display name of the current file (filename without extension) |
| `recent` | `Array<{path, name}>` | Recently opened files/folders, max 20, newest first |
| `expandedKeys` | `string[]` | Expanded folder paths in the sidebar tree |
| `hideAside` | `boolean` | Whether the sidebar is collapsed |
| `asidePos` | `number` | Sidebar width in pixels |
| `winSize` | `[number, number]` | Window `[width, height]` — persisted on resize, restored on launch |
| `lang` | `string` | UI language (`'zh-cn'`, `'en'`, etc.) |
| `theme` | `string` | `'dark'` or `'light'` — defaults to dark if absent |
| `debug` | `boolean` | Enables the DevTools menu item in the tray context menu |
| `token` | `string\|null` | Cloud sync auth token (set via `SetToken`, cleared via `ClearToken`) |
| `backgroundImage` | `string` | URL of the custom background image |
| `backgroundOp` | `string` | Background opacity (`"0"`–`"10"`, as string) |
| `backgroundSize` | `string` | CSS `background-size` for the background image (e.g. `"cover"`, `"contain"`, `"100% auto"`) — defaults to `cover` in CSS |
| `backupSize` | `string` | Max backup size in MB (as string, e.g. `"9.5"`) |
| `workPendTimeMax` | `string` | Max work duration in hours before overtime alert (as string, e.g. `"9.5"`) |
| `dailyInfo` | `{date, start}` | Today's work session — `date` is `"MM-DD"`, `start` is Unix ms timestamp |
| `noticeKey` | `string` | Last-read changelog date key (e.g. `"20260617"`) — suppresses re-display |
| `frameUrl` | `string` | Custom iframe URL for the GPT/AI panel |
| `aiCommand` | `string` | Shell command template for AI chat — `$cmd` is replaced with user input |
| `bottomHtmlTool` | `string` | Custom HTML tool injected at the bottom area |
| `enableFloatingBall` | `boolean` | Show floating action ball (native only) |
| `chatVisible` | `boolean` | Whether the AI chat panel is open |

## Related Paths

- `~/.todoListNative/` — default workspace dir, plugin storage root
- `~/.todoListNative.bak` — backup file (supports diff comparison with current data via `Diff.diffLines`)
- `~/.todoListNative_daily.json` — daily work-time log (separate from config)

## Explorer Filtering

The file explorer (`web/src/utils/expore.ts`) filters out:
- Dot-prefixed files/directories (e.g. `.git`, `.DS_Store`)
- `plugins`, `node_modules`, `.git` directories
