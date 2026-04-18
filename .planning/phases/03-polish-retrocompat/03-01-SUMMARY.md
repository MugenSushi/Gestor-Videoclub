---
phase: 03-polish-retrocompat
plan: "01"
subsystem: grid-badge
tags: [database, ui, badge, episode-progress]
dependency_graph:
  requires: [episode_progress table (Phase 1), ContentCard._build (existing)]
  provides: [Database.get_total_watched_count, ContentCard progress badge]
  affects: [ContentCard rendering for series items]
tech_stack:
  added: []
  patterns: [SELECT COUNT(*) aggregate, tk.Label.place() z-order badge]
key_files:
  created: []
  modified:
    - Videoclub
decisions:
  - "Badge uses bg=C['seen'] (green) to match watched status color — visual consistency over a new color"
  - "Guard watched_count >= 1 hides badge when 0 — no empty label placeholder rendered"
  - "x=2 matches existing badge left margin per UI-SPEC exception table"
metrics:
  duration: "~5 minutes"
  completed: "2026-04-18T21:17:23Z"
  tasks_completed: 2
  files_modified: 1
---

# Phase 03 Plan 01: Episode Progress Badge Summary

**One-liner:** Green `"N ep"` badge on series cards via new `Database.get_total_watched_count` SELECT COUNT query rendered at bottom-left of poster_zone with double guard (series type + count >= 1).

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add Database.get_total_watched_count method | 6384ead | Videoclub (line 429) |
| 2 | Add episode progress badge to ContentCard._build | 6384ead | Videoclub (lines 1417-1427) |

## What Was Built

### Task 1: Database.get_total_watched_count

Inserted after `get_season_progress` (line 429):

- Queries `episode_progress` without `season_number` filter — aggregates all seasons
- Returns `int` count of rows where `watched=1` for the given `(user_id, item_id)` pair
- Returns `0` if no rows found (null-safe)

### Task 2: ContentCard progress badge

Inserted between type badge `.place()` call and `self.poster_lbl.bind(...)` in `ContentCard._build`:

- Guard 1: `self.item["type"] == "series"` — movies never render badge
- Guard 2: `watched_count >= 1` — no empty placeholder when nothing watched
- Badge: `tk.Label` with `text=f"{watched_count} ep"`, `bg=C["seen"]`, `fg="white"`, `font=(FONT_FAMILY, 7, "bold")`, `padx=3`
- Placement: `.place(x=2, y=POSTER_H - 18)` = `(2, 182)` — bottom-left, clear of status badges at y=2
- Z-order: placed after all other poster widgets so renders on top

## Verification Results

```
grep def get_total_watched_count  → line 429 (1 match)
grep POSTER_H - 18               → line 1427 (1 match)
grep watched_count >= 1          → line 1420 (1 match)
grep f"{watched_count} ep"       → line 1423 (1 match)
python ast.parse                 → syntax OK
```

## Deviations from Plan

None — plan executed exactly as written. Both tasks staged together in one `git add Videoclub` since edits were made sequentially to the same file before the first commit; all content verified present in commit 6384ead.

## Known Stubs

None. Badge reads live data from `episode_progress` table via `get_total_watched_count`.

## Self-Check: PASSED

- Videoclub modified: confirmed (git show --stat HEAD shows 21 insertions)
- Commit 6384ead exists: confirmed
- Syntax: clean (ast.parse exits 0)
- All 4 grep patterns: matched
