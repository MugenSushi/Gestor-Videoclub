# Phase 2: SeasonViewWindow - Context

**Gathered:** 2026-04-17
**Status:** Ready for planning

<domain>
## Phase Boundary

Build `SeasonViewWindow` — a modal `tk.Toplevel` that opens from `DetailWindow` for series items
with a valid `tmdb_id`. It displays season tabs (ttk.Notebook), loads episode lists lazily from
TMDb, and lets the user mark individual episodes as watched via checkboxes. Watched state is
written to SQLite (`episode_progress`) immediately on toggle.

Scope: UI-01 through UI-10. No grid badge (Phase 3). No retrocompat disabled-button (Phase 3).

</domain>

<decisions>
## Implementation Decisions

### Seasons List Initialization

- **D-01**: Season list is fetched **asynchronously** — on `SeasonViewWindow` open, show a
  full-window spinner ("Cargando temporadas...") while `fetch_tmdb_detail(tmdb_id, "tv")` runs
  in a background pool worker. Build the `ttk.Notebook` tabs only after the seasons array
  arrives. Never call `fetch_tmdb_detail` synchronously in `__init__`.

- **D-02**: Error fallback when seasons list fetch fails: show `tk.Label` with
  `"No se pudo obtener las temporadas. Comprueba tu conexión."` plus a `"Cerrar"` button
  (`command=self.destroy`). **No fallback to `num_seasons` integer** — do not build generic T1/T2
  tabs from the integer alone.

### Modal Stack & Window Setup

- **D-03** (locked by UI-SPEC): `SeasonViewWindow` calls `grab_set()` in `__init__`.
  `DetailWindow._open_seasons()` binds `win.bind("<Destroy>", lambda e: self.grab_set() if
  self.winfo_exists() else None)` to restore focus. `_on_close()` calls `self.on_refresh()`
  before `self.destroy()`.

- **D-04** (locked by UI-SPEC): `protocol("WM_DELETE_WINDOW", self._on_close)` +
  `bind("<Escape>", lambda e: self._on_close())`. Both paths call `on_refresh` then `destroy`.

### Tab Building & Lazy Loading

- **D-05** (locked by ROADMAP + UI-SPEC): Season list built from TMDb `seasons[]` array
  (field `s["season_number"]`). **Never** `range(1, num_seasons+1)`. Tab labels from TMDb
  `s["name"]`; fallback `"Especiales"` for `season_number == 0` with empty name; fallback
  `f"T{season_number}"` for numbered seasons with empty name.

- **D-06** (locked by UI-SPEC): Each tab is an empty `tk.Frame(bg=C["bg"])` populated lazily
  on `<<NotebookTabChanged>>`. `_loaded_seasons: set` tracks loaded tabs.
  `self.after(0, self._on_tab_changed)` triggers first tab immediately after notebook build.

### Episode Rendering

- **D-07** (locked by UI-SPEC): Canvas + inner Frame (not Treeview, not Listbox).
  MouseWheel bound to BOTH canvas AND inner Frame: `lambda e: canvas.yview_scroll(-1 * (e.delta // 120), "units")`.

- **D-08** (locked by UI-SPEC): Episode row = Checkbutton + `"{ep_num:02d}. {ep_name}"` label
  (fill="x", expand=True) + air date `ep_date[:7]` right-aligned. Future episodes
  (`air_date > today`): `state=DISABLED`, `fg=C["muted"]`.

- **D-09** (locked by UI-SPEC): Checkbox toggle (`_toggle`) writes synchronously to SQLite via
  `db.set_episode_watched(user_id, item_id, season_number, ep_num, new_state)` on main thread.
  Updates progress counter label in-place. No confirmation dialog.

### "Marcar toda la temporada"

- **D-10** (Claude's discretion): Bulk-mark skips future episodes (`air_date > today`).
  Consistent with the disabled-checkbox pattern for unaired episodes. Marks only episodes with
  `air_date <= today` or empty `air_date`. Calls `set_episode_watched` per-episode in a loop
  (no new DB method needed — small episode count).

### Progress Counter

- **D-11** (locked by UI-SPEC): Counter format `"X / Y episodios vistos"`. Updates immediately
  on each toggle by recounting BooleanVars (fast, no extra DB query). When `X == Y`:
  `fg=C["seen"]`. Initialized from `db.get_episode_progress(user_id, item_id, season_number)`
  on tab load.

### Constructor Signature

- **D-12** (Claude's discretion): `SeasonViewWindow(parent, item: dict, user_id: int, db: Database, on_refresh, app_root: tk.Tk)`. Consistent with `DetailWindow` and `RefreshMetaDialog` signatures. `item["id"]` provides `item_id`, `item["tmdb_id"]` provides the TMDb series ID, `item["title"]` provides the window title suffix.

### File Placement

- **D-13** (Claude's discretion): Insert `SeasonViewWindow` class after the `DetailWindow`
  class (after line ~1748 in current file). Section header:
  `# ══════════════... 10.5. SEASON VIEW WINDOW ...══════════════`.
  Add `_open_seasons(self, tmdb_id, num_seasons)` method to `DetailWindow._build()`.

### Threading

- **D-14** (Claude's discretion): Use existing `_poster_pool` for season fetching, OR create
  a dedicated `_season_pool = ThreadPoolExecutor(max_workers=2)` near line 161. The STATE.md
  notes `_season_pool` as the intended pattern. Prefer a dedicated pool to avoid competing with
  poster loading.

### Claude's Discretion

- "Marcar toda" skip-future behavior (D-10)
- Constructor signature alignment (D-12)
- File insertion point (D-13)
- Worker pool (D-14)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### UI Design Contract
- `.planning/phases/02-seasonviewwindow/02-UI-SPEC.md` — Complete visual and interaction contract:
  component inventory, spacing tokens, color roles, interaction protocols (modal stack, lazy
  loading, checkbox toggle, mark-all). Planner and executor treat this as ground truth for all
  widget code.

### Requirements
- `.planning/REQUIREMENTS.md` — Requirements UI-01..UI-10 (all Phase 2 requirements)

### Architecture & Pitfalls
- `.planning/research/ARCHITECTURE.md` — Canvas+Frame scroll pattern, threading pattern
  (`_poster_pool`, `after(0, ...)`), ttk.Style override code, `winfo_exists()` guard examples
- `.planning/research/PITFALLS.md` — `grab_set` re-entry, Canvas scroll on Windows,
  lru_cache pitfalls, Season 0 naming

### Phase Goal & Constraints
- `.planning/ROADMAP.md` — Phase 2 success criteria (6 criteria), Key Constraints table
  (rows 4–10 apply directly to Phase 2)

### Prior Phase Context
- `.planning/STATE.md` §Key Decisions — threading decisions, episode_progress PK, DB methods
  added in Phase 1

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `DetailWindow._center()` (line 1507) — center-on-screen helper, copy verbatim for SeasonViewWindow
- `DetailWindow.__init__` structure (lines 1488–1504) — grab_set, configure, bind Escape pattern
- Canvas+Frame scroll block (lines 3492–3501) — exact template for episode list
- `_poster_pool` (line 161) — existing ThreadPoolExecutor pattern; `_season_pool` to follow same shape
- `get_poster_async` (lines 1238–1273) — complete threading pattern: submit → `after(0, _finalize)` → `winfo_exists()` guard

### Established Patterns
- `after(0, callback)` — all UI updates from worker threads go through this
- `winfo_exists()` guard — checked in every `after()` callback before touching widgets
- `tk.Toplevel` modal stack: `grab_set()` in `__init__`, `on_refresh()` + `destroy()` on close
- Dark theme: `bg=C["bg"]`, `fg=C["text"]`, `bg=C["surface"]` for cards/rows
- `BooleanVar` per checkbox — existing pattern in app (SetupWizard uses Checkbuttons)

### Integration Points
- `DetailWindow._build()` left column (around line 1570) — add "📺 Ver temporadas" button conditional on `itype == "series" and tmdb_id`
- `db.get_episode_progress(user_id, item_id, season_number)` → `{ep_num: bool}` (Phase 1)
- `db.set_episode_watched(user_id, item_id, season_number, ep_num, watched)` → bool (Phase 1)
- `db.get_season_progress(user_id, item_id, season_number)` → `(watched, total)` (Phase 1)
- `fetch_tmdb_season_episodes(tmdb_id, season_number)` → `{"episodes": [...]}` (Phase 1, lru_cached)
- `fetch_tmdb_detail(tmdb_id, "tv")` → includes `"_seasons": [{"season_number": N, "name": "..."}]` (Phase 1)

</code_context>

<specifics>
## Specific Ideas

- Season list initialization must be fully async — no synchronous `fetch_tmdb_detail` call in `__init__`
- Error state for failed seasons fetch: label + "Cerrar" button (not silent failure, not fallback tabs)
- "Marcar toda la temporada" skips episodes with `air_date > today`

</specifics>

<deferred>
## Deferred Ideas

- Window geometry memory (size between opens) — not needed for v1; always open at minimum 600×480
- "Marcar toda" → "Desmarcar" toggle text — out of scope v1 (per UI-SPEC line 157)
- Disabled "Ver temporadas" button for `tmdb_id = NULL` series — Phase 3 (BACK-01/02)

</deferred>

---

*Phase: 02-seasonviewwindow*
*Context gathered: 2026-04-17*
