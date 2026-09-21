# Board Management Workflow — Calendar Scheduling, Monthly Milestones, Focus, Weekly Rolling Files

Practical recipes for maintaining a planning board on top of `.todo` files: how to put tasks on the calendar view, how to organize milestones by month with `timeline` tags, how to use the focus flag, and how to roll the file over week by week. Field-level definitions live in `todo-format.md`; this file is about *what to set and when*.

## Calendar Scheduling with `date` Ranges

A todo appears on the calendar view once it has a date range:

- `todo.date` = `"startSeconds,endSeconds"` — Unix **seconds**, comma-separated string
- `todo.end` = end timestamp in **milliseconds** (precise-time record)

Unit mismatch is the #1 mistake: `date` is in seconds, `start`/`end` in milliseconds.

```python
from datetime import datetime

def day(y, m, d):  # local midnight, Unix seconds
    return int(datetime(y, m, d).timestamp())

todo['date'] = f"{day(2026, 9, 21)},{day(2026, 9, 24)}"  # multi-day span
todo['end']  = day(2026, 9, 24) * 1000                   # precise end, ms
```

- Single-day task: use the same day for both ends.
- A range renders as an occupied block spanning those days on the calendar view.
- Only put `date` on items that genuinely have a schedule — milestone-level or this-week work, not every leaf task.

## Monthly Milestones via `timeline` Tags

The milestone/timeline page aggregates every todo whose tag has `timeline: true`. The convention for *monthly* milestones:

1. Define one tag per month in `todotree.tags` — new key is a fresh millisecond timestamp:

```json
"1787551930048": { "label": "Sep", "color": "#a59eff", "fontColor": "#000000", "level": "9", "timeline": true }
```

- `level` carries the month number. It is written as a string (`"9"`) in real files; numeric also works.
- Pick any `color`; `fontColor` usually black-on-pastel.

2. Tag a todo with that month's key (`todo.tags.push(monthKey)`) and it shows up under that month on the milestone page. Remove the tag to take it out.

### Keeping months meaningful (anti-pileup rules)

Without curation everything ends up in the current month. Split deliberately:

- **Current month = key items only** — the ones tied to your actual goals (OKR or equivalent). Everything else stays untagged.
- **Work planned for a later month** → that month's tag. Create next month's tag ahead of time and move items there instead of letting them sit in the current month.
- **Already-completed deliverables** → the month they were *finished* in. Backfilling done items (`"done": true`) with a past month's tag is fine and makes the milestone page read as a history of shipped work.
- **Routine / support tasks get no month tag** — tagging them is how the current month piles up.
- One theme, one month tag — don't duplicate the same item across months.

Batch retagging is a small recursive walk (match by content keyword, then swap the tag):

```python
def retag(nodes, keyword, old_tag, new_tag=None):
    for n in nodes:
        if keyword in (n.get('todo') or {}).get('content', ''):
            tags = n['todo'].setdefault('tags', [])
            if old_tag in tags:
                tags.remove(old_tag)
                if new_tag: tags.append(new_tag)
        retag(n.get('children') or [], keyword, old_tag, new_tag)
```

## Focus View

`todo.focus: true` puts an item in the focus view. Treat it as *this period's active set*:

- At each period rollover, set `focus: true` on what you'll actually touch (including items completed this period), `false` on everything else.
- Focus is per-item, not inherited — a focused parent does not focus its children.

## Weekly Rolling File

A common pattern is one `.todo` file per week (`MMDD-MMDD.todo`). Rollover recipe:

1. Copy last week's file to the new name (mind holiday weeks — the range may be shorter than 7 days).
2. **Five things must all be updated** (missing any one leaves the board stale or mislabeled — the `title` is the easiest to forget because it's not near the content you're editing):
   - `title` — the new week's name
   - `desc` — new week's summary + notes (holidays, risks)
   - `expandKeys` — rebuild as all numbers (see `todo-format.md`)
   - `version` — fresh timestamp
   - section content (see below)
3. Clear last week's completed items: recursively remove `done` subtrees; drop the whole "done archive" section if you keep one (history stays in the old file, which is what monthly reviews read).
4. Keep long-lived sections (e.g. a milestone backlog) across weeks **as-is**, done items included — they are the long-range view.
5. Rebuild the "this week" section by **moving the node objects themselves** in (not copying — copies drift apart and duplicate). Update wording: "next week" → "this week"; carry-overs get a reason noted.
6. New nodes: `id`, `key`, `start` all the same fresh millisecond timestamp, unique within the file.
7. Push the store to the web board (see `webhook-sync.md`). On the board, the first `Store` write of a *new* file auto-switches `currentKey` to it.
8. Remind anyone else syncing the repo (other machines' app instances) to pull — a stale client saving over the tree will clobber the pushed state.

## Simple Mode for Read-Only Boards

`simpleMode` is a store-level boolean (`todotree.simpleMode`) the renderer reads on load: when `true`, node rows show the date instead of edit controls, and extra tool areas/progress bars are hidden — a clean look for a board people only *look* at.

No code change needed: since the agent controls the store, just set `simpleMode: true` in the store you push (and keep it set on subsequent pushes). This is the recommended default for a Docker-deployed read-only board; the editor clients (vscode / desktop / remote) read the same field, so toggle it there via the view options if you want it off while editing.


## Multi-Writer Sync (Human + Agent on One Git-Backed Board)

When a person (via the app) and an agent (via scripts editing the file / pushing the board) both write the same `.todo` repo, treat it like any shared repo with an *additional* twist — the app saves whole-tree snapshots, not patches:

- **Pull before every scripted edit.** The other writer may have pushed since your last read; editing a stale tree and pushing clobbers their changes.
- **Dedupe by content, not by id.** Concurrent writes produce duplicate nodes with the *same content but different ids* — an id-based check will happily pass and add a second copy. Before adding an item, recursively search the tree for the content keyword first.
- **Push promptly, then tell the other end to pull.** A stale app instance that saves after your push will overwrite the board with its old tree. Don't leave long windows of "edited locally, not yet pushed".
- Prefer one writer at a time per file when you can; when you can't, keep edit sessions short and always end them with pull → edit → push to the board → notify.

## Done-as-Archive

Don't physically delete completed items — check `done: true` and leave them in place:

- Old weekly files *are* the archive: completed items stay behind in the file of the week they were finished, and monthly/quarterly reviews read those files as history. Nothing extra to maintain.
- "Decided not to do (for now)" items also stay, marked in the wording (e.g. a "(deferred)" note with the reason) rather than deleted — the decision itself is worth remembering.
- Deleting done items loses your review material and breaks the "board = what happened" property; the cost of keeping them is one cleared checkbox.

## Tree Hygiene

Periodically tidy the structure, not just the items:

- **Collapse scattered siblings** into their semantic parent (five loose "fix login UI" notes under different sections → children of one "login UI" item).
- **Merge duplicate themes** into a single item (note the merge in the wording once merged).
- **When dissolving a section**: move its done items to the done-archive section, return unfinished items to their original/home sections — don't drop either.
- Before adding anything, search by content keyword (see Multi-Writer Sync above) — most board clutter is near-duplicates that a keyword check would have caught.

## `desc` as the Summary Slot

The file-level `desc` is the first thing shown when the board opens — use it as a structured summary, not scratch notes:

- Keep a fixed shape: a short **period summary** (a few bullet points: what this week/month is about), then a separate headed **risk list** (bullets, one risk each). A fixed shape keeps it scannable; free-form prose tends to collapse into a wall of text.
- Rewrite it at every rollover — a stale desc describing last period is worse than none.
- It pairs well with the milestone tags: desc answers "what is this period about", milestones answer "what lands when".

## Pre-push Checklist

- `expandKeys` are all numbers
- every id in `todo.tags` exists in `todotree.tags`
- ids unique within the file; `date` seconds vs `start`/`end` ms not mixed
- no near-duplicate content introduced (search by keyword before adding)
- verify key nodes by content (recursive search — top-level-only search silently misses nested nodes) **before** the line that pushes
- keep any assertion/verification code out of the push path — an exception between "saved file" and "pushed board" leaves the two out of sync
