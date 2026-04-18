---
phase: 03-polish-retrocompat
reviewed: 2026-04-18T00:00:00Z
depth: standard
files_reviewed: 1
files_reviewed_list:
  - Videoclub
findings:
  critical: 0
  warning: 3
  info: 2
  total: 5
status: issues_found
---

# Phase 03: Code Review Report

**Reviewed:** 2026-04-18
**Depth:** standard
**Files Reviewed:** 1
**Status:** issues_found

## Summary

Three focused areas reviewed per phase scope: `Database.get_total_watched_count` (line 429), the progress badge block in `ContentCard._build` (lines 1417–1427), and the `DetailWindow._build` "Ver temporadas" split (lines 1596–1616).

The SQL method is correct and safe. The badge block introduces one DB call per card render with no error guard — a crash risk. The DetailWindow split has a subtle lambda-closure bug and a tooltip concern on a disabled button.

---

## Warnings

### WR-01: DB call in `_build` has no exception guard — crashes entire card on DB error

**File:** `Videoclub:1419`
**Issue:** `get_total_watched_count` is called synchronously inside `_build`, which runs on the main thread during grid rendering. If the `episode_progress` table does not yet exist (e.g. fresh install before migration runs, or migration failed silently at line 350), the call raises `sqlite3.OperationalError` and the entire `ContentCard` constructor propagates the exception, blowing up the whole refresh cycle. The migration failure path already swallows the error with a log (`log.error`) and continues, so a partially-migrated DB is a real scenario.

**Fix:**
```python
# Replace lines 1419-1427 with:
if self.item["type"] == "series":
    try:
        watched_count = self.db.get_total_watched_count(self.user_id, self.item["id"])
    except Exception:
        watched_count = 0
    if watched_count >= 1:
        tk.Label(
            poster_zone,
            text=f"{watched_count} ep",
            bg=C["seen"], fg="white",
            font=(FONT_FAMILY, 7, "bold"),
            padx=3
        ).place(x=2, y=POSTER_H - 18)
```

---

### WR-02: Lambda in enabled "Ver temporadas" branch captures `self.item["tmdb_id"]` by reference redundantly — and ignores `num_seasons` from `_open_seasons` signature

**File:** `Videoclub:1603–1606`
**Issue:** `_open_seasons` is defined as `def _open_seasons(self, tmdb_id: str, num_seasons: int)` (line 1782) but its body never uses `tmdb_id` or `num_seasons` — it builds `SeasonViewWindow` directly from `self.item`. The lambda passes `self.item["tmdb_id"]` and `self.item.get("num_seasons", 0)` as arguments, which are silently ignored inside `_open_seasons`. This is dead parameter passing that creates a false expectation that the arguments matter. More importantly, if the `tmdb_id` guard at line 1598 passes (`if tmdb_id:`) but `self.item["tmdb_id"]` is later `None` at call time (e.g. item mutated), the lambda passes `None` — still silently ignored now, but fragile if `_open_seasons` is ever fixed to use `tmdb_id`.

**Fix:** Either remove the parameters from `_open_seasons` entirely (cleaner), or make the body actually use them:
```python
# Option A — remove dead params (preferred)
command=lambda: self._open_seasons()

def _open_seasons(self):
    win = SeasonViewWindow(self, item=self.item, ...)

# Option B — use the params in the body
def _open_seasons(self, tmdb_id: str, num_seasons: int):
    win = SeasonViewWindow(self, item=self.item,
                           tmdb_id=tmdb_id, num_seasons=num_seasons, ...)
```

---

### WR-03: `Tooltip` binds `<Enter>` on a `state="disabled"` button — tooltip may never fire on some platforms

**File:** `Videoclub:1616`
**Issue:** `Tooltip(btn, ...)` binds `<Enter>` to the disabled `tk.Button` at line 1609–1616. On Windows with Tk 8.6+, `<Enter>` events are NOT delivered to widgets with `state="disabled"` in some themes (especially with ttk-influenced behavior). The tooltip is specifically meant to explain WHY the button is disabled, so silent non-delivery defeats its purpose.

**Fix:** Wrap the disabled button in a `tk.Frame` and attach the tooltip to the frame instead, which always receives `<Enter>`:
```python
wrapper = tk.Frame(left, bg=C["bg"])
wrapper.pack(fill="x", pady=(8, 0))
btn = tk.Button(
    wrapper, text="📺 Ver temporadas",
    bg=C["surface"], fg=C["muted"],
    font=(FONT_FAMILY, 9), relief="flat",
    state="disabled"
)
btn.pack(fill="x")
Tooltip(wrapper, "Re-añade la serie via búsqueda para activar")
```

---

## Info

### IN-01: `get_total_watched_count` uses `COUNT(*)` with a `WHERE watched=1` filter — correct but could use `SUM(watched)` for symmetry with sibling method

**File:** `Videoclub:431–435`
**Issue:** `get_season_progress` (line 419) uses `SUM(CASE WHEN watched=1 THEN 1 ELSE 0 END)` while `get_total_watched_count` uses `COUNT(*) WHERE watched=1`. Both are semantically correct since `watched` is stored as `0`/`1`. The inconsistency is minor but could confuse future maintainers about whether `watched` can hold non-boolean integers.

**Fix:** No functional change needed. If consistency is desired:
```python
"SELECT SUM(watched) as cnt FROM episode_progress"
" WHERE user_id=? AND item_id=?",
```
And handle the `NULL` return (when no rows match) already covered by `row["cnt"] if row else 0` — but `SUM` returns `NULL` on empty set, so change to `(row["cnt"] or 0) if row else 0`.

---

### IN-02: Progress badge at `(x=2, y=POSTER_H-18)` has no `pady` — text may clip at the bottom edge

**File:** `Videoclub:1427`
**Issue:** The badge label is placed at `y = POSTER_H - 18 = 182` inside a frame of exact height `POSTER_H = 200`. With `font=(FONT_FAMILY, 7, "bold")` the label height is typically 14–16px, so the bottom sits at ~198px — right at the frame boundary. On high-DPI or different system font scaling, the label may clip. Other badges at `y=2` have the same pattern but clip upward into negative space (clipped by parent, invisible), not downward where it would be obviously broken.

**Fix:** Use `y=POSTER_H - 20` to add 2px breathing room, consistent with the top badges' 2px margin:
```python
).place(x=2, y=POSTER_H - 20)
```

---

_Reviewed: 2026-04-18_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
