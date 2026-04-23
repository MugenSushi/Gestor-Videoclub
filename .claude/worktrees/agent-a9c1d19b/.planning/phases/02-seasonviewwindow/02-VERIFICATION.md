---
phase: 02-seasonviewwindow
verified: 2026-04-18T16:12:51Z
status: human_needed
score: 6/6 must-haves verified
overrides_applied: 0
human_verification:
  - test: "Open DetailWindow for a series with persisted tmdb_id. Verify 'Ver temporadas' button is visible. Verify it is absent for a movie item."
    expected: "Button appears only for series with non-null tmdb_id; absent for movies and series with null tmdb_id."
    why_human: "Conditional rendering depends on runtime item data; cannot be verified without a live DB row."
  - test: "Click 'Ver temporadas'. Verify SeasonViewWindow opens as modal (parent window unresponsive). Verify spinner appears, then season tabs appear. Verify closing the window returns focus to DetailWindow."
    expected: "Modal focus grab works; spinner → tabs transition completes; grab restored on DetailWindow after close."
    why_human: "Tkinter grab_set and modal behaviour requires UI interaction to confirm."
  - test: "Switch to a second season tab. Verify spinner appears and episode list loads. Switch back to first tab. Verify no second spinner (lazy load guard working)."
    expected: "Already-loaded tabs do not re-fetch; _loaded_seasons set prevents duplicate spinner."
    why_human: "Tab-switch lazy-load guard can only be confirmed through live UI interaction."
  - test: "Toggle a checkbox for a non-future episode. Immediately open a SQLite browser on the DB file and check episode_progress table."
    expected: "Row inserted/updated in episode_progress with watched=1 for that user_id/item_id/season/episode combination."
    why_human: "SQLite write confirmation requires inspecting the actual database file post-toggle."
  - test: "Verify progress counter updates on toggle and turns green at 100%. Click 'Marcar toda la temporada'. Verify all non-future checkboxes tick and counter shows 100% in green."
    expected: "Counter text matches X/Y pattern. At 100% fg turns C[seen] (#1db954). Future episodes remain unticked and uncounted."
    why_human: "Visual color and counter accuracy require live UI interaction with real season data."
  - test: "Open a season that has future episodes (air_date in the future). Verify those rows appear grey and checkboxes are non-interactive."
    expected: "Future episode rows have muted fg and state=disabled on checkbutton."
    why_human: "Requires a series with a future air_date in TMDb to verify visually."
  - test: "Close SeasonViewWindow. Verify the parent grid refreshes without manual F5."
    expected: "on_refresh callback fires, grid cards update to reflect any newly-marked episodes."
    why_human: "Grid refresh requires observing UI state change on close."
---

# Phase 2: SeasonViewWindow Verification Report

**Phase Goal:** Users can open a dedicated window from a series detail view, navigate season tabs, see all episodes with their watched state, and toggle individual episodes — with the state persisted to SQLite immediately.
**Verified:** 2026-04-18T16:12:51Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths (Roadmap Success Criteria)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Opening DetailWindow for series with tmdb_id shows "Ver temporadas" button; clicking opens SeasonViewWindow as modal; closing returns focus to DetailWindow | VERIFIED (code) / needs human (runtime) | Line 1576-1585: conditional button. Line 1751-1763: _open_seasons instantiates SeasonViewWindow, binds `<Destroy>` → grab_set restore. Line 1806: grab_set() in SeasonViewWindow.__init__. |
| 2 | Each season is a ttk.Notebook tab; switching to unloaded tab triggers background fetch (spinner); switching back to loaded tab does not re-fetch | VERIFIED (code) / needs human (runtime) | Line 1885-1886: NotebookTabChanged bound + after(0) trigger. Line 1888-1901: _on_tab_changed checks _loaded_seasons set before calling _load_season. Line 1903: _load_season shows spinner then async fetch. |
| 3 | Each episode row shows episode number, title, air date, checkbox; toggle writes to episode_progress in SQLite immediately (no save button) | VERIFIED (code) / needs human (runtime) | Line 2040-2044: _toggle closure calls db.set_episode_watched immediately on checkbox change. Line 2067: f"{ep_num:02d}. {ep_name}" label. Lines 2072-2079: air_date[:7] right-aligned label. |
| 4 | Season header shows accurate "X / Y episodios vistos" counter; updates on toggle; "Marcar toda" sets all and updates counter | VERIFIED (code) / needs human (runtime) | Lines 1974, 2049, 2109: three update sites for counter. Line 1986: "Marcar toda la temporada" button. Line 2080: _mark_all_season iterates non-future episodes and calls set_episode_watched. |
| 5 | Future episodes (air_date in the future) are grey and non-interactive | VERIFIED (code) / needs human (runtime) | Line 2029: is_future = bool(ep_date) and ep_date > today. Line 2061: chk.config(state="disabled", fg=C["muted"]). Line 2094-2095: _mark_all_season skips future episodes. |
| 6 | Closing SeasonViewWindow calls on_refresh so grid and DetailWindow reflect changes without manual refresh | VERIFIED (code) / needs human (runtime) | Lines 1819-1822: _on_close sets _destroyed=True first, then calls on_refresh(), then destroy(). |

**Score:** 6/6 truths verified at code level

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Videoclub` (line 162) | `_season_pool = ThreadPoolExecutor(max_workers=2, thread_name_prefix="season")` | VERIFIED | Exists at line 162, immediately after _poster_pool line 161. |
| `Videoclub` (line ~4146) | `_season_pool.shutdown(wait=False)` in VideoclubApp._on_close | VERIFIED | Lines 4145-4146: _poster_pool.shutdown then _season_pool.shutdown on adjacent lines. |
| `Videoclub` (line ~1578) | "Ver temporadas" button in DetailWindow._build, conditional on series+tmdb_id | VERIFIED | Lines 1575-1585: tmdb_id extracted, conditional block, button with bg=C["accent"]. |
| `Videoclub` (line 1751) | DetailWindow._open_seasons method | VERIFIED | Lines 1751-1763: method exists, instantiates SeasonViewWindow with all correct args. |
| `Videoclub` (line 1780) | `class SeasonViewWindow(tk.Toplevel)` under "10.5. SEASON VIEW WINDOW" | VERIFIED | Line 1777: section header. Line 1780: class definition. |
| `Videoclub` (line 1818) | `_on_close` sets _destroyed=True first, then on_refresh(), then destroy() | VERIFIED | Lines 1819-1822: exact order confirmed. |
| `Videoclub` (line 1824) | `_show_seasons_spinner` with spinner frame, async _season_pool worker, after(0) dispatch | VERIFIED | Lines 1824-1858: spinner frame, _worker function, lambda d=data default-capture at line 1836, _season_pool.submit at line 1858. |
| `Videoclub` (line 1860) | `_build_notebook` with Dark.TNotebook style before Notebook instantiation, tab-per-season, Especiales fallback | VERIFIED | Lines 1860-1886: style.configure x2 + style.map before ttk.Notebook at 1871. "Especiales" fallback at line 1879. |
| `Videoclub` (line 1888) | `_on_tab_changed` with _destroyed guard, try/except on _nb.index, _loaded_seasons check | VERIFIED | Lines 1888-1901: all guards present including try/except added as deviation in plan 02. |
| `Videoclub` (line 1903) | `_load_season` fully implemented (not stub) with spinner, async fetch, cache, render | VERIFIED | Lines 1903-1939: full implementation. fetch_tmdb_season_episodes at line 1914. _season_cache set at line 1922 before _render_episode_list. |
| `Videoclub` (line 1941) | `_render_episode_list` with header, canvas+frame, episode rows, progress counter | VERIFIED | Lines 1941-2078: full implementation. get_episode_progress at 1950. Canvas+Frame+scrollbar. MouseWheel loop at lines 2010-2014 binding both widgets. |
| `Videoclub` (line 2080) | `_mark_all_season` skipping future episodes, calling set_episode_watched, updating counter | VERIFIED | Lines 2080-2115: full implementation. air_date > today skip at line 2092-2095. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| DetailWindow._build() | DetailWindow._open_seasons() | lambda: self._open_seasons(tmdb_id, num_seasons) | WIRED | Line 1581: command=lambda: self._open_seasons(...) |
| DetailWindow._open_seasons() | SeasonViewWindow.__init__ | SeasonViewWindow(self, item=...) | WIRED | Lines 1752-1759: correct args passed |
| SeasonViewWindow.__init__ | _show_seasons_spinner | self._show_seasons_spinner() | WIRED | Line 1811: called at end of __init__ |
| _show_seasons_spinner | _season_pool worker | _season_pool.submit(_worker) | WIRED | Line 1858 |
| _worker (thread) | _on_seasons_done (main thread) | self.app_root.after(0, lambda d=data: _on_seasons_done(d)) | WIRED | Line 1836: default-capture confirmed |
| _build_notebook | _on_tab_changed | NotebookTabChanged + self.after(0, self._on_tab_changed) | WIRED | Lines 1885-1886 |
| _on_tab_changed | _load_season | self._load_season(season_number, frame) | WIRED | Line 1901 |
| _load_season | _render_episode_list | self.app_root.after(0, lambda d=data: _on_done(d)) → _render_episode_list | WIRED | Lines 1915, 1923 |
| Checkbutton command | db.set_episode_watched | _toggle inner function | WIRED | Lines 2040-2044 |
| _on_close | on_refresh | self.on_refresh() in _on_close | WIRED | Line 1820 |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|---------------|--------|--------------------|--------|
| _render_episode_list | progress (watched state) | db.get_episode_progress(user_id, item_id, season_number) | Yes — SQLite query in Database class (Phase 1) | FLOWING |
| _render_episode_list | episodes | fetch_tmdb_season_episodes → _season_cache | Yes — TMDb HTTP API with lru_cache (Phase 1) | FLOWING |
| _toggle | new_state (BooleanVar) | db.set_episode_watched(...) | Yes — SQLite upsert in Database class (Phase 1) | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Videoclub parses without SyntaxError | `python -c "import ast; ast.parse(open('Videoclub', encoding='utf-8').read())"` | SYNTAX OK | PASS |
| _season_pool declared at module level | `grep -c "_season_pool  = ThreadPoolExecutor" Videoclub` | 1 match at line 162 | PASS |
| All 6 plan commits exist in git history | `git log --oneline cc3f78c be5ab73 a5120fb 8b0f08c 0939810 12f9aa1` | All 6 present | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| UI-01 | 02-01-PLAN | "Ver temporadas" button visible only if series + tmdb_id | SATISFIED | Lines 1575-1585: conditional button guard + rendering confirmed |
| UI-02 | 02-02-PLAN | SeasonViewWindow with grab_set; grab restored to DetailWindow on close | SATISFIED (code) | Line 1806: grab_set. Lines 1760-1763: Destroy→grab_set restore. Needs human for runtime confirmation. |
| UI-03 | 02-02-PLAN | ttk.Notebook tabs + NotebookTabChanged; lazy load | SATISFIED (code) | Lines 1885-1901: NotebookTabChanged + _loaded_seasons lazy guard |
| UI-04 | 02-03-PLAN | "X / Y episodios vistos" counter per tab, updates on toggle | SATISFIED (code) | Lines 1974, 2049, 2109: counter init and both update sites |
| UI-05 | 02-03-PLAN | Canvas+Frame scrollable episode list; episode number, title, air date per row | SATISFIED (code) | Lines 1993-2014: Canvas+Frame+Scrollbar. Lines 2025-2079: episode rows with all three fields |
| UI-06 | 02-03-PLAN | Checkbutton per episode; toggle writes to SQLite immediately | SATISFIED (code) | Lines 2037-2044: BooleanVar + _toggle → set_episode_watched |
| UI-07 | 02-03-PLAN | "Marcar toda la temporada como vista" button in header | SATISFIED (code) | Lines 1983-1991: button with command=_mark_all_season. Lines 2080-2115: bulk-mark implementation |
| UI-08 | 02-03-PLAN | Future episodes in grey, non-interactive | SATISFIED (code) | Lines 2029, 2061: is_future detection and disabled+muted state |
| UI-09 | 02-02-PLAN + 02-03-PLAN | Loading spinner visible during TMDb fetch; error visible on failure | SATISFIED (code) | Line 1824: seasons spinner. Line 1905: episode spinner. Lines 1846, 1927, 1934: error labels |
| UI-10 | 02-03-PLAN | Closing SeasonViewWindow calls on_refresh | SATISFIED (code) | Lines 1819-1822: _on_close sequence confirmed |

All 10 requirements from UI-01 through UI-10 are accounted for. No orphaned requirements.

GRID-01, BACK-01, BACK-02 are mapped to Phase 3 — not in scope for this phase.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| Videoclub | 1903 | `_load_season` — stub was `pass` in plan 02 | INFO (resolved) | Replaced with full implementation in plan 03 at same line. Not a live stub. |

No live stubs detected in the final state of the file. `_load_season` at line 1903 contains full implementation.

### Human Verification Required

#### 1. Modal behavior and grab_set chain

**Test:** Open DetailWindow for a series with a persisted tmdb_id. Click "Ver temporadas". Attempt to interact with the parent DetailWindow while SeasonViewWindow is open.
**Expected:** Parent window is unresponsive (grab_set active). Closing SeasonViewWindow returns keyboard/mouse focus to DetailWindow.
**Why human:** Tkinter grab_set and modal focus chaining requires live UI interaction.

#### 2. Lazy tab loading (no re-fetch on revisit)

**Test:** Open SeasonViewWindow. Wait for tabs to appear. Click season tab 2 — spinner then episodes. Click back to tab 1.
**Expected:** Tab 1 shows no spinner (already loaded). No duplicate network call.
**Why human:** Tab-switch sequence requires runtime observation.

#### 3. SQLite write on checkbox toggle

**Test:** Toggle a checkbox for a non-future episode. Open a SQLite browser on the DB file.
**Expected:** Row in `episode_progress` with correct user_id, item_id, season_number, episode_number, watched=1.
**Why human:** Database write confirmation requires external inspection.

#### 4. Progress counter accuracy and color flip

**Test:** Toggle checkboxes on a season. Observe counter. Mark all non-future episodes watched.
**Expected:** Counter text updates instantly to reflect new watched count. At 100% (all non-future episodes watched) the counter text turns green (C["seen"] = #1db954).
**Why human:** Visual color and counter accuracy require live UI interaction.

#### 5. Future episode disabled state

**Test:** Find or construct a series with an episode whose air_date is in the future. Open that season tab.
**Expected:** Future episode rows have grey text. Checkboxes are disabled (cannot be clicked). "Marcar toda" does not mark them.
**Why human:** Requires real TMDb data with future air_date to verify visually.

#### 6. on_refresh grid update on close

**Test:** Mark several episodes watched in SeasonViewWindow. Close the window. Observe the main grid.
**Expected:** Grid cards update (progress badge or watched indicator) without pressing F5.
**Why human:** Grid refresh depends on parent on_refresh callback wiring that exists in VideoclubApp — confirmed in code, but visual result requires runtime observation.

### Gaps Summary

No code-level gaps found. All 6 roadmap success criteria are implemented, all 10 requirement IDs (UI-01 through UI-10) have corresponding code evidence, all commits are present in git history, and the file passes Python syntax check.

The 7 human verification items are standard UI/runtime behaviors that cannot be confirmed programmatically. They represent expected manual smoke-test steps, not implementation deficiencies.

---

_Verified: 2026-04-18T16:12:51Z_
_Verifier: Claude (gsd-verifier)_
