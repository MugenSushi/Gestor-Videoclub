---
phase: 02-seasonviewwindow
reviewed: 2026-04-18T00:00:00Z
depth: standard
files_reviewed: 1
files_reviewed_list:
  - Videoclub
findings:
  critical: 1
  warning: 4
  info: 2
  total: 7
status: issues_found
---

# Phase 02: Code Review Report

**Reviewed:** 2026-04-18
**Depth:** standard
**Files Reviewed:** 1
**Status:** issues_found

## Summary

Reviewed `SeasonViewWindow` (lines 1780–2112), `_open_seasons` (lines 1751–1763), `_show_seasons_spinner` (1824–1858), `_build_notebook` (1860–1886), `_on_tab_changed` (1888–1901), `_load_season` (1903–1939), `_render_episode_list` (1941–2078), `_mark_all_season` (2080–2112), `_season_pool` (line 162), and surrounding infrastructure.

One critical bug (`FONT_FAMILY` referenced before assignment — crashes on non-Windows at import). Four logic warnings (counter mismatch between future/all episodes, dead `_404` branch, global `lru_cache` cleared on any transient error, `_toggle` counts include future episodes). Two info items (repeated `import datetime`, mouse-wheel binding only covers Windows delta units).

---

## Critical Issues

### CR-01: `FONT_FAMILY` referenced before assignment — `NameError` on non-Windows

**File:** `Videoclub:115`
**Issue:** The ternary `"Segoe UI" if … else FONT_FAMILY` references `FONT_FAMILY` in its `else` branch before the name has ever been defined. On Windows the condition is always `True` so it goes unnoticed; on Linux/macOS the `else` branch executes and raises `NameError: name 'FONT_FAMILY' is not defined`, crashing the entire module at import time.

**Fix:**
```python
# Replace line 115 with a fallback that does not self-reference:
FONT_FAMILY = "Segoe UI" if platform.system() == "Windows" else "DejaVu Sans"
```

---

## Warnings

### WR-01: `_toggle` counter includes future-episode `BooleanVar`s — totals are wrong

**File:** `Videoclub:2046-2051`
**Issue:** `_render_episode_list` appends a `BooleanVar` into `_season_bvars[season_number]` for **every** episode, including disabled future ones (line 2038). But `_toggle` computes `total_now = len(self._season_bvars[sn])` (all vars) and `watched_now` by summing **all** vars. Future checkboxes start `False` and can never be set, so `total_now` is inflated by the number of future episodes and the "all watched" colour condition is never reachable while any future episode exists.

The header counter calculated at render time (`total_countable`, `watched_init`) correctly excludes future episodes, creating an inconsistency: the header shows `3 / 5` on load but `_toggle` immediately flips it to `3 / 8` when the user ticks the first episode.

**Fix:**
Only append to `_season_bvars` for non-future episodes, and keep the disabled checkbox outside the tracked list:
```python
# Before creating var/row, branch:
if not is_future:
    var = tk.BooleanVar(value=is_watched)
    self._season_bvars[season_number].append(var)
else:
    var = tk.BooleanVar(value=False)   # untracked throwaway
```
`_toggle` then naturally counts only the trackable episodes, matching `total_countable`.

---

### WR-02: `_mark_all_season` bvar index assumes episodes are sorted and contiguous — silently marks wrong episodes

**File:** `Videoclub:2087-2102`
**Issue:** `_mark_all_season` iterates `episodes` in API order and advances `bv_idx` to index into `bvars`. This works only if `_render_episode_list` appended bvars in the exact same episode order. But (a) the TMDb API may occasionally return episodes out of order, (b) if WR-01 is fixed and future episodes are skipped during append, the index-alignment still holds only because both loops skip future episodes in the same order — but `_mark_all_season` also iterates all episodes to decide whether to skip, meaning any future episode that appears in the middle resets the `bv_idx` alignment correctly only by coincidence of sequential iteration. The fragile index coupling is a maintenance hazard; if episode rows are ever reordered or filtered, `bv_idx` will point to the wrong var.

**Fix:** Align bvars by `episode_number` key instead of positional index. Store them in a dict:
```python
# In _render_episode_list, change the list to a dict:
self._season_bvars[season_number] = {}   # ep_num -> BooleanVar
...
self._season_bvars[season_number][ep_num] = var

# In _mark_all_season:
for ep in episodes:
    ep_num = ep.get("episode_number", 0)
    ...
    if is_future:
        continue
    self.db.set_episode_watched(...)
    if ep_num in bvars:
        bvars[ep_num].set(True)

# In _toggle:
watched_now = sum(1 for bv in self._season_bvars[sn].values() if bv.get())
total_now   = len(self._season_bvars[sn])
```

---

### WR-03: `fetch_tmdb_season_episodes` clears the **entire** lru_cache on any transient error

**File:** `Videoclub:1035`
**Issue:** When any transient network error occurs on **any** season request, `_fetch_tmdb_season_cached.cache_clear()` wipes cached results for **all** seasons across all series currently open. A user browsing season 1 while season 3 has a transient 5xx will lose season 1's cached data. For a modal window this is low-severity in practice, but it is architecturally wrong and will cause unnecessary re-fetches.

**Fix:** Use a dict-based cache instead of `lru_cache`, keyed by `(tmdb_id, season_number)`, so individual entries can be invalidated without clearing everything:
```python
_season_data_cache: dict = {}  # (tmdb_id, season_number) -> dict | None

def fetch_tmdb_season_episodes(tmdb_id: str, season_number: int) -> dict:
    key = (tmdb_id, season_number)
    if key in _season_data_cache:
        cached = _season_data_cache[key]
        return {} if cached is None else cached   # None = permanent miss
    result = _do_fetch_season(tmdb_id, season_number)
    if result.get("_404"):
        _season_data_cache[key] = None   # permanent miss
        return {}
    if result:
        _season_data_cache[key] = result
    # transient error: do NOT store; will retry on next call
    return result
```

---

### WR-04: Dead branch `data.get("_404")` in `_load_season` is unreachable

**File:** `Videoclub:1924`
**Issue:** `_load_season._on_done` checks `elif data.get("_404")` (line 1924), but `fetch_tmdb_season_episodes` (line 1037–1038) already strips the `_404` sentinel and returns `{}` to the caller. The `_404` key will therefore never appear in `data`, so this branch is dead. The user will see the generic "Error al conectar" message for a 404 instead of the more helpful "Esta temporada aún no tiene datos en TMDb."

**Fix:** `fetch_tmdb_season_episodes` should distinguish permanent absence from transient error by returning a sentinel the caller can inspect, e.g. a dedicated return value or by re-exposing the `_404` flag:
```python
# Option A — return sentinel dict for 404:
if result.get("_404"):
    _season_data_cache[key] = None
    return {"_no_data": True}   # permanent, no episodes, not a network error

# In _load_season._on_done:
elif data.get("_no_data"):
    tk.Label(..., text="Esta temporada aún no tiene datos en TMDb.").pack(...)
```

---

## Info

### IN-01: `import datetime` repeated inside two methods

**File:** `Videoclub:1942, 2081`
**Issue:** `datetime` is imported inside `_render_episode_list` and again inside `_mark_all_season`. Both are standard-library imports that belong at module top level. Inlined imports are evaluated on every call and are harder to track.

**Fix:** Move `import datetime` to the top-level imports block (already present: `from datetime import datetime` — extend it to `from datetime import datetime, date`), then replace `datetime.date.today()` with `date.today()`.

---

### IN-02: Mouse-wheel scroll binding uses Windows-only `e.delta // 120`

**File:** `Videoclub:2012-2014`
**Issue:** The `<MouseWheel>` binding divides `e.delta` by 120, which is the Windows scroll unit. On Linux the delta is 1 per scroll step (event type `<Button-4>`/`<Button-5>` instead of `<MouseWheel>`), and on macOS `e.delta` is already in lines. The episode list will not scroll on Linux.

**Fix:**
```python
def _on_mousewheel(e):
    if platform.system() == "Windows":
        canvas.yview_scroll(-1 * (e.delta // 120), "units")
    elif platform.system() == "Darwin":
        canvas.yview_scroll(-1 * e.delta, "units")
    # Linux handled via Button-4/5 below

for w in (canvas, inner):
    w.bind("<MouseWheel>", _on_mousewheel)
    w.bind("<Button-4>", lambda e: canvas.yview_scroll(-1, "units"))
    w.bind("<Button-5>", lambda e: canvas.yview_scroll( 1, "units"))
```

---

_Reviewed: 2026-04-18_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
