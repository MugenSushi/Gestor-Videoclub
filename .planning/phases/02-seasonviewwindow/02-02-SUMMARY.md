---
phase: 02-seasonviewwindow
plan: 02
subsystem: SeasonViewWindow skeleton
tags: [ui, modal, notebook, threading, tkinter]
dependency_graph:
  requires: [02-01]
  provides: [SeasonViewWindow, _show_seasons_spinner, _build_notebook, _on_tab_changed]
  affects: [Videoclub]
tech_stack:
  added: []
  patterns: [ttk.Notebook dark style, after(0) main-thread dispatch, _destroyed guard, lambda d=data default-capture]
key_files:
  created: []
  modified:
    - Videoclub
decisions:
  - _on_close sets _destroyed=True as first statement (T-02-07 race condition mitigation)
  - _on_tab_changed wraps _nb.index("current") in try/except to guard against notebook not yet rendered
  - style.configure/map applied before ttk.Notebook instantiation (ordering required by Tkinter)
  - _load_season is a pass stub — episode rendering deferred to plan 03
metrics:
  duration: 10m
  completed: 2026-04-18
  tasks_completed: 2
  files_modified: 1
requirements: [UI-02, UI-03, UI-09]
---

# Phase 2 Plan 02: SeasonViewWindow Skeleton Summary

**One-liner:** SeasonViewWindow modal Toplevel with async TMDb fetch, dark ttk.Notebook tab-per-season, _destroyed guard, and lazy _on_tab_changed dispatch via after(0).

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Insert SeasonViewWindow class with modal setup | a5120fb | Videoclub |
| 2 | Add _show_seasons_spinner, _build_notebook, _on_tab_changed, _load_season stub | 8b0f08c | Videoclub |

## What Was Built

### Task 1 — SeasonViewWindow constructor and modal setup
- `class SeasonViewWindow(tk.Toplevel)` at line ~1780 under section header `10.5. SEASON VIEW WINDOW`
- `__init__` initializes all instance state: `item`, `user_id`, `db`, `on_refresh`, `app_root`, `_tmdb_id`, `_item_id`, `_destroyed=False`, `_loaded_seasons`, `_season_frames`, `_season_cache`, `_season_bvars`, `_prog_labels`
- Window geometry: `700x520`, `minsize(560, 400)`, `grab_set()` before any widget calls
- `protocol("WM_DELETE_WINDOW", self._on_close)` + `<Escape>` binding
- `_center()` copied from DetailWindow pattern
- `_on_close()` sets `_destroyed=True` first, then calls `on_refresh()`, then `destroy()`

### Task 2 — Async spinner and notebook
- `_show_seasons_spinner`: creates spinner frame, submits `_worker` to `_season_pool`, dispatches via `self.app_root.after(0, lambda d=data: _on_seasons_done(d))` (default-capture)
- `_on_seasons_done` (closure): guards with `_destroyed`/`winfo_exists`, destroys spinner, shows error state (label + Cerrar button) if no seasons, otherwise calls `_build_notebook`
- `_build_notebook`: applies `Dark.TNotebook` style BEFORE creating `ttk.Notebook`; builds one `tk.Frame` tab per season; tab labels use TMDb `name` field with fallbacks (`Especiales` for season_number=0, `T{n}` for numbered); binds `<<NotebookTabChanged>>`; calls `self.after(0, self._on_tab_changed)` to trigger first tab immediately
- `_on_tab_changed`: guarded by `_destroyed` + `winfo_exists`; wraps `_nb.index("current")` in try/except; checks `_loaded_seasons` set before loading; calls `_load_season`
- `_load_season`: stub with `pass` — episode rendering implemented in plan 03

## Deviations from Plan

**1. [Rule 2 - Missing critical guard] Added try/except around `_nb.index("current")`**
- Found during: Task 2 implementation review
- Issue: Plan spec called bare `idx = self._nb.index("current")` — can raise TclError if notebook has no tabs or is not yet mapped
- Fix: Wrapped in `try/except Exception: return`
- Files modified: Videoclub
- Commit: 8b0f08c

## Threat Model Coverage

| Threat | Status |
|--------|--------|
| T-02-04 Information Disclosure (season names in labels) | accepted — public TMDb metadata, no PII |
| T-02-05 DoS (_build_notebook unlimited tabs) | accepted — seasons[] bounded by TMDb |
| T-02-06 EoP (grab_set blocks parent) | accepted — intended modal behavior |
| T-02-07 Spoofing (_destroyed race condition) | mitigated — _destroyed=True set as FIRST statement in _on_close |

## Known Stubs

| Stub | File | Line | Reason |
|------|------|------|--------|
| `_load_season` returns `pass` | Videoclub | ~1903 | Episode list rendering deferred to plan 03 |

## Self-Check: PASSED

- `class SeasonViewWindow(tk.Toplevel)` at line 1780: CONFIRMED
- `10.5. SEASON VIEW WINDOW` section header: CONFIRMED (line 1777)
- `self._destroyed = False` in __init__: CONFIRMED (line 1793)
- `self.grab_set()` in SeasonViewWindow: CONFIRMED (line 1806)
- `self.minsize(560, 400)`: CONFIRMED (line 1804)
- `def _center` in SeasonViewWindow: CONFIRMED (line 1812)
- `def _on_close`: CONFIRMED (line 1818), `_destroyed=True` is first statement
- `def _show_seasons_spinner`: CONFIRMED (line 1824)
- `lambda d=data:` default-capture: CONFIRMED (line 1836)
- `_season_pool.submit(_worker)`: CONFIRMED (line 1858)
- `def _build_notebook`: CONFIRMED (line 1860)
- `Dark.TNotebook` style (configure x2, map x1, Notebook instantiation): CONFIRMED
- `<<NotebookTabChanged>>` binding: CONFIRMED (line 1885)
- `self.after(0, self._on_tab_changed)`: CONFIRMED (line 1886)
- `def _on_tab_changed`: CONFIRMED (line 1888)
- `def _load_season` stub: CONFIRMED (line 1903)
- Commit a5120fb exists: CONFIRMED
- Commit 8b0f08c exists: CONFIRMED
- Syntax check: PASSED
