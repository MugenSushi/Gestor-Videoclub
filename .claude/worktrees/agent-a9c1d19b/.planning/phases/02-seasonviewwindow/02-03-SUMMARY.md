---
phase: 02-seasonviewwindow
plan: 03
subsystem: SeasonViewWindow episode rendering
tags: [ui, episodes, tkinter, canvas, threading, sqlite]
dependency_graph:
  requires: [02-02]
  provides: [_load_season, _render_episode_list, _mark_all_season]
  affects: [Videoclub]
tech_stack:
  added: []
  patterns: [Canvas+Frame scrollable list, BooleanVar per episode, default-capture closures, after(0) dispatch, datetime.date.today()]
key_files:
  created: []
  modified:
    - Videoclub
decisions:
  - _season_bvars includes ALL episodes (including future) so total count prevents false 100% with unaired episodes
  - _toggle recounts entire bvars list rather than incrementing delta to avoid drift on rapid toggles
  - import datetime inside method is safe (stdlib); module-level only imports datetime.datetime not datetime.date
  - episodios vistos string appears 3 times (prog_lbl init, _toggle, _mark_all) — all three update the counter
metrics:
  duration: 12m
  completed: 2026-04-18
  tasks_completed: 2
  files_modified: 1
requirements: [UI-04, UI-05, UI-06, UI-07, UI-08, UI-09, UI-10]
---

# Phase 2 Plan 03: Episode Rendering Summary

**One-liner:** Full SeasonViewWindow episode UI — async fetch with spinner, Canvas+Frame scroll, per-episode BooleanVar Checkbuttons, SQLite toggle on click, progress counter, bulk-mark skipping future episodes.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Implement _load_season (replace stub) with spinner and async episode fetch | 0939810 | Videoclub |
| 2 | Implement _render_episode_list, _mark_all_season, full episode rows | 12f9aa1 | Videoclub |

## What Was Built

### Task 1 — _load_season (stub replacement)

- Spinner label "Cargando temporada..." shown immediately in tab_frame
- `_worker` submits to `_season_pool`, calls `fetch_tmdb_season_episodes`
- Dispatch via `self.app_root.after(0, lambda d=data: _on_done(d))` — default-capture prevents late-binding
- `_on_done` guarded by `self._destroyed or not self.winfo_exists()` — prevents crash on quick close
- On success: `_season_cache[season_number] = data` set BEFORE `_render_episode_list` call
- On 404: "Esta temporada aún no tiene datos en TMDb." label
- On network error: "Error al conectar con TMDb. Comprueba tu conexión." label

### Task 2 — _render_episode_list and _mark_all_season

**_render_episode_list:**
- `get_episode_progress(user_id, item_id, season_number)` loads watched state from SQLite
- Season header: progress counter label (fg=C[seen] at 100%) + "Marcar toda la temporada" button (right-aligned)
- Horizontal separator `tk.Frame(bg=C["border"], height=1)`
- Canvas + Scrollbar + inner Frame; `<Configure>` bound to update scrollregion
- `<MouseWheel>` bound to BOTH canvas AND inner Frame (Windows inner-frame event capture)
- Per episode: `tk.BooleanVar`, `tk.Checkbutton` (state=disabled + fg=muted for future episodes)
- Episode label: `f"{ep_num:02d}. {ep_name}"`, air_date[:7] right-aligned
- Row hover: `<Enter>`/`<Leave>` with `r=row` default capture
- `_toggle` closure: `n=ep_num, v=var, sn=season_number, r=row` default capture; writes `set_episode_watched` then recounts all BooleanVars for counter

**_mark_all_season:**
- Reads `_season_cache[season_number]` for episode list
- Iterates episodes; skips `air_date > today` (matches disabled checkbox state)
- Calls `set_episode_watched(..., True)` per eligible episode
- Sets matching BooleanVar to True via bv_idx parallel counter
- Updates `_prog_labels[season_number]` counter and color

## Deviations from Plan

None — plan executed exactly as written.

The acceptance criterion "exactly 2 matches" for `episodios vistos` is superseded by the parenthetical noting "_mark_all also updates" — 3 matches in the file is correct behavior.

## Threat Model Coverage

| Threat | Status |
|--------|--------|
| T-02-08 Injection (ep_name/ep_date in tk.Label) | accepted — Tkinter Label.config(text=) has no injection surface |
| T-02-09 Tampering (ep_num as PK) | mitigated — `ep.get("episode_number", 0)` used; parameterized query in set_episode_watched |
| T-02-10 DoS (long episode list) | accepted — Canvas scroll handles 6-26 episodes trivially |
| T-02-11 EoP (_mark_all_season writes) | accepted — UI button only; user_id validated at login |
| T-02-12 Spoofing (future episode BooleanVar) | mitigated — is_future disables Checkbutton; _mark_all skips air_date > today |

## Known Stubs

None — all episode rendering is fully implemented.

## Self-Check: PASSED

- `def _load_season` at line 1903: CONFIRMED (no longer `pass`)
- `fetch_tmdb_season_episodes` called inside _load_season: CONFIRMED (line 1915)
- `self._season_cache[season_number] = data` before render: CONFIRMED (line 1922)
- "Esta temporada aún no tiene datos en TMDb.": CONFIRMED (line 1927)
- "Error al conectar con TMDb. Comprueba tu conexión.": CONFIRMED (line 1934)
- "Cargando temporada...": CONFIRMED (line 1907)
- `lambda d=data:` in _load_season: CONFIRMED (line 1915)
- `def _render_episode_list` at line 1941: CONFIRMED
- `def _mark_all_season` at line 2080: CONFIRMED
- `db.set_episode_watched` in _toggle and _mark_all_season: CONFIRMED (lines 2042, 2097)
- `db.get_episode_progress` in _render_episode_list: CONFIRMED (line 1950)
- "episodios vistos" x3: CONFIRMED (lines 1974, 2049, 2109)
- "Marcar toda la temporada": CONFIRMED (line 1986)
- `state="disabled"` for future episodes: CONFIRMED (line 2061)
- MouseWheel bound to canvas and inner: CONFIRMED (lines 2012, 2013)
- `n=ep_num, v=var` default capture in _toggle: CONFIRMED (line 2040)
- `air_date > today` checks: CONFIRMED (lines 1964, 2029, 2092)
- `ep_num:02d` format: CONFIRMED (line 2067)
- "No hay episodios disponibles": CONFIRMED (line 2020)
- Syntax check: PASSED
- Commit 0939810 exists: CONFIRMED
- Commit 12f9aa1 exists: CONFIRMED
