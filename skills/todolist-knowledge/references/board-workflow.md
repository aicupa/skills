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

## Pre-push Checklist

- `expandKeys` are all numbers
- every id in `todo.tags` exists in `todotree.tags`
- ids unique within the file; `date` seconds vs `start`/`end` ms not mixed
- verify key nodes by content (recursive search — top-level-only search silently misses nested nodes) **before** the line that pushes
- keep any assertion/verification code out of the push path — an exception between "saved file" and "pushed board" leaves the two out of sync
