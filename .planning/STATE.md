# STATE — Videoclub STARTUP: Series Tracking

**Project:** Videoclub STARTUP v7.0 — Season/Episode Tracking  
**Milestone:** Series Tracking  
**Last updated:** 2026-04-14

---

## Project Reference

**Core value:** Saber exactamente en qué punto dejaste una serie y poder marcar episodios vistos de forma rápida sin salir de la app.

**Current focus:** Phase 1 — DB + API Foundation

---

## Current Position

**Phase:** 1 — DB + API Foundation  
**Plan:** None started  
**Status:** Not started  

```
[          ] Phase 1: DB + API Foundation
[          ] Phase 2: SeasonViewWindow
[          ] Phase 3: Polish + Retrocompat
```

**Overall progress:** 0 / 3 phases complete

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Phases defined | 3 |
| Requirements mapped | 23/23 |
| Plans complete | 0 |
| Plans in progress | 0 |

---

## Accumulated Context

### Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| 3 coarse phases | Requirements cluster naturally: DB foundation, UI window, polish/retrocompat |
| Phase 1 includes all API functions | DB and API are tightly coupled — `tmdb_id` must exist in DB before any API call makes sense to wire |
| Phase 2 builds the full SeasonViewWindow in one phase | The 10 UI requirements form a single coherent deliverable; splitting would leave a non-functional half |
| Phase 3 is deliberately last | Grid badge (GRID-01) requires episode_progress data from Phase 1+2; retrocompat (BACK-01/02) requires the button to exist from Phase 2 |

### Critical Implementation Notes

- **SCHEMA_VERSION must be bumped to 7 as the very first edit** (Pitfall m3 — if forgotten, migration runs on every launch)
- **`episode_progress` primary key uses `(user_id, item_id, season_number, episode_number)`** — `item_id` FK to `items.id`, NOT `tmdb_series_id` TEXT (per ARCHITECTURE.md final schema)
- **`tmdb_id` wiring path:** `fetch_tmdb_multi` (line 844) → `SmartSearchDialog._finish_manual()` (line 1940) → `add_item()` — needs new params `tmdb_id=""` and `num_seasons=0`
- **`num_seasons` source:** already in `fetch_tmdb_detail` response as `number_of_seasons`; extract and pass through at add time — no extra API call needed
- **Season 0:** iterate `detail["seasons"]` array, use `s["season_number"]` directly, display TMDb `name` field (fall back to "Especiales" for season_number=0 with empty name)
- **Threading pattern:** match existing `get_poster_async` pattern — submit to `_POOL` (or dedicated `_season_pool`), UI updates only via `app_root.after(0, callback)`, guard with `winfo_exists()`
- **Canvas scroll:** bind `<MouseWheel>` to both `canvas` AND `inner` Frame — inner Frame catches events first on Windows

### Blockers

None — ready to start Phase 1.

### Todos

- [ ] Start Phase 1 planning (`/gsd-plan-phase 1`)

---

## Session Continuity

**To resume:** Read ROADMAP.md, then read the current phase's plan file (none yet — start with Phase 1).

**Context files in priority order:**
1. `.planning/STATE.md` (this file — current position)
2. `.planning/ROADMAP.md` (phase structure and success criteria)
3. `.planning/REQUIREMENTS.md` (full requirement list)
4. `.planning/research/ARCHITECTURE.md` (implementation patterns and data flow)
5. `.planning/research/PITFALLS.md` (critical mistakes to avoid)

---
*State initialized: 2026-04-14*
