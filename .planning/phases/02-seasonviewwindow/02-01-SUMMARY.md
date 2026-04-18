---
phase: 02-seasonviewwindow
plan: 01
subsystem: DetailWindow / entry-point wiring
tags: [threadpool, ui, entry-point, detail-window]
dependency_graph:
  requires: [01-03]
  provides: [_season_pool, DetailWindow._open_seasons, Ver-temporadas-button]
  affects: [Videoclub]
tech_stack:
  added: []
  patterns: [ThreadPoolExecutor dedicated pool, conditional button, Toplevel modal chain]
key_files:
  created: []
  modified:
    - Videoclub
decisions:
  - Dedicated _season_pool (max_workers=2) separate from _poster_pool to avoid contention
  - tmdb_id extracted as local var in _build() after Eliminar button (itype already existed)
  - _open_seasons inserted before _delete for logical grouping inside DetailWindow
metrics:
  duration: 8m
  completed: 2026-04-18
  tasks_completed: 2
  files_modified: 1
requirements: [UI-01]
---

# Phase 2 Plan 01: Entry-Point Wiring Summary

**One-liner:** Added `_season_pool` ThreadPoolExecutor, conditional "Ver temporadas" button in DetailWindow, and `_open_seasons()` method that instantiates SeasonViewWindow with grab_set restore.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Add _season_pool global and shutdown hook | cc3f78c | Videoclub |
| 2 | Add Ver-temporadas button and _open_seasons method | be5ab73 | Videoclub |

## What Was Built

### Task 1 — _season_pool global
- `_season_pool = ThreadPoolExecutor(max_workers=2, thread_name_prefix="season")` at line 162, immediately after `_poster_pool`
- `_season_pool.shutdown(wait=False)` in `VideoclubApp._on_close()` immediately after `_poster_pool.shutdown`
- T-02-02 (DoS via unconstrained pool) mitigated by `max_workers=2` cap

### Task 2 — Ver temporadas entry point
- "📺 Ver temporadas" button added to `DetailWindow._build()` left panel after Eliminar button
- Conditional: only rendered when `itype == "series" and tmdb_id`
- Uses `bg=C["accent"]`, `fg="white"`, `font=(FONT_FAMILY, 9)`, `relief="flat"`, `cursor="hand2"`
- `DetailWindow._open_seasons(tmdb_id, num_seasons)` method added before `_delete`
- Instantiates `SeasonViewWindow(self, item=..., user_id=..., db=..., on_refresh=..., app_root=...)`
- Binds `<Destroy>` on SeasonViewWindow window to restore `DetailWindow.grab_set()` when it closes

## Deviations from Plan

None — plan executed exactly as written.

`itype` was already declared at line 1516 in `_build()` as confirmed by reading the file before editing; only `tmdb_id` was declared as a new local variable in the button block.

## Threat Model Coverage

| Threat | Status |
|--------|--------|
| T-02-01 Information Disclosure (tmdb_id) | accepted — public TMDb ID, no PII |
| T-02-02 DoS (_season_pool unconstrained) | mitigated — max_workers=2 in Task 1 |
| T-02-03 Tampering (on_refresh callback) | accepted — internal method reference |

## Known Stubs

`SeasonViewWindow` is referenced in `_open_seasons()` but does not yet exist — it will be created in Plan 02-02. Launching the app and clicking "Ver temporadas" will raise `NameError: name 'SeasonViewWindow' is not defined` until Plan 02-02 is complete. This is expected and intentional — this plan is the entry-point wiring only.

## Self-Check: PASSED

- Videoclub exists and is modified: CONFIRMED
- Commit cc3f78c exists: CONFIRMED
- Commit be5ab73 exists: CONFIRMED
- Syntax check: PASSED (ast.parse)
- `_season_pool  = ThreadPoolExecutor(max_workers=2, thread_name_prefix="season")` at line 162: CONFIRMED
- `_season_pool.shutdown(wait=False)` present: CONFIRMED (line 3807)
- `📺 Ver temporadas` button present: CONFIRMED (line 1578)
- `def _open_seasons` present: CONFIRMED (line 1751)
- `SeasonViewWindow(` present in _open_seasons: CONFIRMED (line 1752)
- `itype == "series" and tmdb_id` guard: CONFIRMED (line 1576)
- `win.bind("<Destroy>", ... grab_set ...)` present: CONFIRMED (lines 1760-1762)
