# ROADMAP — Videoclub STARTUP: Series Tracking (Milestone v7.0)

**Project:** Videoclub STARTUP — Season/Episode tracking  
**Granularity:** Coarse (3 phases)  
**Coverage:** 23/23 v1 requirements mapped  
**Created:** 2026-04-14

---

## Phases

- [x] **Phase 1: DB + API Foundation** — Schema migration v6→v7, tmdb_id persistence, season fetch functions (completed 2026-04-17)
- [ ] **Phase 2: SeasonViewWindow** — Full season/episode UI with lazy loading, checkboxes, and progress counters
- [ ] **Phase 3: Polish + Retrocompat** — Progress badge on grid cards, disabled-state handling for legacy series, dark theme styling

---

## Phase Details

### Phase 1: DB + API Foundation
**Goal**: The database has all columns and tables needed for episode tracking, tmdb_id is persisted on add/enrich, and season data can be fetched from TMDb and cached for the session.
**Depends on**: Nothing (first phase)
**Requirements**: DB-01, DB-02, DB-03, DB-04, DB-05, API-01, API-02, API-03, API-04, API-05

**Success Criteria** (what must be TRUE):
  1. Running the app against an existing v6 database completes startup without errors; `PRAGMA table_info(items)` confirms `tmdb_id` and `num_seasons` columns exist; `sqlite_master` confirms `episode_progress` table and its indexes exist
  2. Adding a series via TMDb search persists a non-null `tmdb_id` and `num_seasons` value in the `items` row (verifiable in a SQLite browser after adding)
  3. Calling `fetch_tmdb_season_episodes(tmdb_id, season_number)` from the Python REPL returns a dict with an `episodes` list; a second call to the same arguments returns the cached value without a network request (lru_cache)
  4. The season list for a series is built from the TMDb `seasons[]` array (not `range(1, N+1)`), so Season 0 ("Especiales") appears as a tab when it exists in TMDb data
  5. HTTP 404, 429, and network errors from the TMDb season endpoint each produce a distinct log entry and a user-visible message rather than a silent failure or crash

**Plans:** 3/3 plans complete

Plans:
- [x] 01-01-PLAN.md — Schema migration: SCHEMA_VERSION=7, ALTER TABLE tmdb_id/num_seasons, episode_progress table and DB methods
- [x] 01-02-PLAN.md — API layer: extend fetch_tmdb_detail return, add fetch_tmdb_season_episodes with lru_cache and error handling
- [x] 01-03-PLAN.md — Wire tmdb_id through add_item signature and SmartSearchDialog call sites

---

### Phase 2: SeasonViewWindow
**Goal**: Users can open a dedicated window from a series detail view, navigate season tabs, see all episodes with their watched state, and toggle individual episodes — with the state persisted to SQLite immediately.
**Depends on**: Phase 1
**Requirements**: UI-01, UI-02, UI-03, UI-04, UI-05, UI-06, UI-07, UI-08, UI-09, UI-10

**Success Criteria** (what must be TRUE):
  1. Opening DetailWindow for a series with a persisted `tmdb_id` shows a "Ver temporadas" button; clicking it opens SeasonViewWindow as a modal (grabbing focus), and closing SeasonViewWindow returns focus to DetailWindow
  2. Each season is a tab in ttk.Notebook; switching to a tab that has not been loaded yet triggers a background fetch (spinner visible during load); switching back to an already-loaded tab does not re-fetch
  3. Each episode row shows episode number, title, air date, and a checkbox; checking or unchecking a checkbox writes the new state to `episode_progress` in SQLite within the same user action (no explicit save button)
  4. The season header displays an accurate "X / Y episodios vistos" counter that updates immediately when any checkbox is toggled; "Marcar toda la temporada como vista" sets all checkboxes and updates the counter in one action
  5. Episodes with an `air_date` in the future are rendered in a muted/grey color and their checkboxes are non-interactive
  6. Closing SeasonViewWindow calls the `on_refresh` callback so the parent grid and DetailWindow reflect updated progress without requiring a manual F5 refresh

**Plans:** 3 plans

Plans:
- [ ] 02-01-PLAN.md — _season_pool global + DetailWindow "Ver temporadas" button + _open_seasons method
- [ ] 02-02-PLAN.md — SeasonViewWindow class skeleton: modal, async seasons fetch, dark Notebook, lazy tab registration
- [ ] 02-03-PLAN.md — Episode list: _load_season, _render_episode_list, canvas+frame, checkboxes, progress counter, mark-all

---

### Phase 3: Polish + Retrocompat
**Goal**: Series cards in the main grid show episode progress at a glance, and series added before v7 (without tmdb_id) have a gracefully degraded "Ver temporadas" button with a helpful tooltip rather than a broken state.
**Depends on**: Phase 2
**Requirements**: GRID-01, BACK-01, BACK-02

**Success Criteria** (what must be TRUE):
  1. A series card in the main grid that has at least one watched episode shows a progress badge (e.g. "T2 E5" or "4/10") without opening any window
  2. A series card for a series with `tmdb_id = NULL` shows the "Ver temporadas" button in a disabled state; hovering over it shows a tooltip explaining why it is disabled and suggesting the user re-add the series via search
  3. The ttk.Notebook tabs in SeasonViewWindow use the dark theme palette (`C["bg"]`, `C["surface"]`, `C["muted"]`, `C["text"]`) — no light-grey default ttk appearance leaks through

**Plans**: TBD
**UI hint**: yes

---

## Progress Table

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. DB + API Foundation | 3/3 | Complete   | 2026-04-17 |
| 2. SeasonViewWindow | 0/3 | Not started | - |
| 3. Polish + Retrocompat | 0/? | Not started | - |

---

## Coverage Map

| Requirement | Phase |
|-------------|-------|
| DB-01 | Phase 1 |
| DB-02 | Phase 1 |
| DB-03 | Phase 1 |
| DB-04 | Phase 1 |
| DB-05 | Phase 1 |
| API-01 | Phase 1 |
| API-02 | Phase 1 |
| API-03 | Phase 1 |
| API-04 | Phase 1 |
| API-05 | Phase 1 |
| UI-01 | Phase 2 |
| UI-02 | Phase 2 |
| UI-03 | Phase 2 |
| UI-04 | Phase 2 |
| UI-05 | Phase 2 |
| UI-06 | Phase 2 |
| UI-07 | Phase 2 |
| UI-08 | Phase 2 |
| UI-09 | Phase 2 |
| UI-10 | Phase 2 |
| GRID-01 | Phase 3 |
| BACK-01 | Phase 3 |
| BACK-02 | Phase 3 |

**Mapped: 23/23 — no orphans**

---

## Key Constraints (from research)

These constraints are enforced across all phases:

| Constraint | Where It Applies |
|------------|-----------------|
| `SCHEMA_VERSION` bumped to 7 before any migration code | Phase 1, first edit |
| `ALTER TABLE` uses nullable columns only (no NOT NULL without DEFAULT on existing tables) | Phase 1 |
| Season list built from TMDb `seasons[]` array, never `range(1, N+1)` | Phase 1 API layer |
| All UI updates via `self.after(0, ...)` — no widget calls from worker threads | Phase 2 |
| `SeasonViewWindow.grab_set()` + `<Destroy>` re-grab on DetailWindow | Phase 2 |
| Lazy tab loading via `<<NotebookTabChanged>>` only | Phase 2 |
| Episode list uses Canvas + inner Frame (not ttk.Treeview, not Listbox) | Phase 2 |
| Always use TMDb `episode_number` field as PK component — never list index | Phase 1 DB + Phase 2 |
| `winfo_exists()` guard in all `after()` callbacks (window may close mid-fetch) | Phase 2 |
| Single `Videoclub` file — no new modules | All phases |

---
*Roadmap created: 2026-04-14*
*Updated: 2026-04-14 — Phase 1 plans added (3 plans, 3 waves)*
*Updated: 2026-04-17 — Phase 2 plans added (3 plans, 3 waves sequential)*
