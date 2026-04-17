---
gsd_state_version: 1.0
milestone: v7.0
milestone_name: milestone
status: executing
last_updated: "2026-04-17T10:25:16.512Z"
progress:
  total_phases: 3
  completed_phases: 1
  total_plans: 3
  completed_plans: 3
  percent: 100
---

# STATE — Videoclub STARTUP: Series Tracking

**Project:** Videoclub STARTUP v7.0 — Season/Episode Tracking  
**Milestone:** Series Tracking  
**Last updated:** 2026-04-17

---

## Project Reference

**Core value:** Saber exactamente en qué punto dejaste una serie y poder marcar episodios vistos de forma rápida sin salir de la app.

**Current focus:** Phase 2 — SeasonViewWindow

---

## Current Position

Phase: 1 (DB + API Foundation) — COMPLETE
Plan: 3 of 3
**Phase:** 1 — DB + API Foundation  
**Plan:** 01-03 COMPLETE — tmdb_id + num_seasons Wiring  
**Status:** Phase 1 complete — ready for Phase 2

```
[██████████] Phase 1: DB + API Foundation (3/3 plans done) COMPLETE
[          ] Phase 2: SeasonViewWindow
[          ] Phase 3: Polish + Retrocompat
```

**Overall progress:** 1 / 3 phases complete

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| Phases defined | 3 |
| Requirements mapped | 23/23 |
| Plans complete | 1 |
| Plans in progress | 0 |

---
| Phase 01-db-api-foundation P01 | 79s | 2 tasks | 1 files |
| Phase 01-db-api-foundation P02 | 62 | 2 tasks | 1 files |
| Phase 01-db-api-foundation P03 | 60 | 2 tasks | 1 files |

## Accumulated Context

### Key Decisions Made

| Decision | Rationale |
|----------|-----------|
| 3 coarse phases | Requirements cluster naturally: DB foundation, UI window, polish/retrocompat |
| Phase 1 includes all API functions | DB and API are tightly coupled — `tmdb_id` must exist in DB before any API call makes sense to wire |
| Phase 2 builds the full SeasonViewWindow in one phase | The 10 UI requirements form a single coherent deliverable; splitting would leave a non-functional half |
| Phase 3 is deliberately last | Grid badge (GRID-01) requires episode_progress data from Phase 1+2; retrocompat (BACK-01/02) requires the button to exist from Phase 2 |
| Two-function lru_cache split for season fetcher | Prevents caching transient empty-dict results — only real 200 responses and permanent 404 sentinels are cached; public wrapper calls cache_clear() on transient errors |
| media_type != 'movie' guard in fetch_tmdb_detail | Backward compatible — movies receive no _num_seasons/_seasons keys; TV series get both from the already-fetched /tv/{id} response at no extra API cost |
| tmdb_id/num_seasons before user_id in add_item | Item-level fields logically precede user-level; safe defaults (""/0) preserve backward compat for BatchImportDialog without changes |
| _worker injects _tmdb_id via fetch_tmdb_multi | smart_search/fetch_tmdb single-result path does not return _tmdb_id; fetch_tmdb_multi does — injection fills the gap for series manual add path |

### Critical Implementation Notes

- **SCHEMA_VERSION must be bumped to 7 as the very first edit** (Pitfall m3 — if forgotten, migration runs on every launch)
- **`episode_progress` primary key uses `(user_id, item_id, season_number, episode_number)`** — `item_id` FK to `items.id`, NOT `tmdb_series_id` TEXT (per ARCHITECTURE.md final schema)
- **`tmdb_id` wiring path:** `fetch_tmdb_multi` (line 844) → `SmartSearchDialog._finish_manual()` (line 1940) → `add_item()` — needs new params `tmdb_id=""` and `num_seasons=0`
- **`num_seasons` source:** already in `fetch_tmdb_detail` response as `number_of_seasons`; extract and pass through at add time — no extra API call needed
- **Season 0:** iterate `detail["seasons"]` array, use `s["season_number"]` directly, display TMDb `name` field (fall back to "Especiales" for season_number=0 with empty name)
- **Threading pattern:** match existing `get_poster_async` pattern — submit to `_POOL` (or dedicated `_season_pool`), UI updates only via `app_root.after(0, callback)`, guard with `winfo_exists()`
- **Canvas scroll:** bind `<MouseWheel>` to both `canvas` AND `inner` Frame — inner Frame catches events first on Windows

### Blockers

None.

### Todos

- [x] Execute Phase 1 Plan 02 (API functions: fetch_tmdb_detail extension, fetch_tmdb_season_episodes) — DONE
- [x] Execute Phase 1 Plan 03 (SmartSearchDialog call sites: add_item extension, _add_result, _finish_manual, _add_manual wiring) — DONE
- [ ] Execute Phase 2 (SeasonViewWindow UI)

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
