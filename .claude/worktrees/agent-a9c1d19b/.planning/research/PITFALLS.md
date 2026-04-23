# Domain Pitfalls — Episode/Season Tracking (Videoclub STARTUP v7.0)

**Domain:** Desktop Python/Tkinter app adding TMDb-sourced episode tracking to existing SQLite schema
**Researched:** 2026-04-14
**Based on:** Direct code inspection of `D:/VideoClub/Videoclub` (~3650 lines, schema v6)

---

## Critical Pitfalls

Mistakes that cause rewrites or corrupt data.

---

### Pitfall C1: Season 0 is "Specials" — iterating `range(1, num_seasons+1)` misses it and crashes on others

**What goes wrong:**
TMDb uses season_number=0 for "Specials" (bonus episodes, recap episodes, unaired pilots). The `number_of_seasons` field in the TV detail response counts *regular* seasons and does NOT include season 0. If you build season tabs by iterating `range(1, series["number_of_seasons"] + 1)`, you silently skip specials. Worse: some series have irregular season numbering (e.g., a series with seasons 1, 2, 4 where 3 was renumbered) — the count is still right but individual numbers are wrong.

**Why it happens:**
Developers assume seasons are a contiguous 1-indexed sequence. TMDb actually returns the authoritative list in the `seasons` array of the TV detail endpoint (`/3/tv/{id}`), each item having `season_number`, `episode_count`, and `name`. The `number_of_seasons` integer is a shortcut that lies about the real structure.

**Consequences:**
- Season 0 never shown, user cannot mark special episodes watched
- If a series has gaps (seasons 1,2,4), hitting `/tv/{id}/season/3` returns HTTP 404, crashing silently or showing empty UI
- `episode_progress` rows stored with wrong `season_number` when the iteration number and the actual TMDb season_number diverge

**Prevention:**
Always fetch `/3/tv/{tmdb_id}` first to get the `seasons` array. Iterate over that array, not over a numeric range. Each element gives `season_number` directly.

```python
# WRONG — do not do this
for n in range(1, detail["number_of_seasons"] + 1):
    fetch_season(tmdb_id, n)

# CORRECT
for s in detail.get("seasons", []):
    fetch_season(tmdb_id, s["season_number"])
```

**Detection:** Query TMDb for a series known to have specials (e.g., Breaking Bad, Doctor Who) during development and verify the season list.

**Phase:** Phase that builds SeasonViewWindow and the season tab/list UI.

---

### Pitfall C2: `tmdb_id` is not persisted today — the DB has no column for it

**What goes wrong:**
The current schema (v6) has no `tmdb_id` column in `items`. `fetch_tmdb_multi` returns `_tmdb_id` in its result dict and it is used transiently during search (line 844, 1940–1943), but it is never written to the DB. The `episode_progress` table needs a stable `tmdb_id` to call `/tv/{id}/season/{n}`. Without it, every time SeasonViewWindow opens you must re-search TMDb by title — which fails for series with ambiguous titles and wastes API calls.

**Why it happens:**
The original design never needed a persistent TMDb ID (search was stateless). The v7 KEY DECISION already identifies this gap but it is easy to implement the migration incorrectly.

**Consequences:**
- SeasonViewWindow cannot load data for series added before v7 without a live TMDb re-search
- Re-search by title is unreliable: "The Office" returns US and UK versions; user gets the wrong season data silently
- Without tmdb_id, `episode_progress` rows cannot have a stable foreign reference; they could silently map to the wrong series if the title is renamed in the DB

**Prevention:**
The migration must `ALTER TABLE items ADD COLUMN tmdb_id TEXT`. Then populate it for existing rows via a background job at first launch. The job should call `fetch_tmdb_multi(title, type_hint=row["type"])` and take the first result's `_tmdb_id` only when the match confidence is high (title match exact or near-exact). For ambiguous titles, leave `tmdb_id` NULL and show a one-time prompt in SeasonViewWindow asking the user to confirm which series.

**Warning signs:**
- `SeasonViewWindow` receives `item` dict without `tmdb_id` key and falls through to a title-based search silently — add an explicit `assert` or early-return with a user-visible message during development.

**Phase:** DB migration phase (v6→v7). Must land before SeasonViewWindow is built.

---

### Pitfall C3: SQLite `ALTER TABLE` cannot add NOT NULL columns without a default — migration silently passes then crashes on INSERT

**What goes wrong:**
SQLite's `ALTER TABLE ... ADD COLUMN` does not support adding a `NOT NULL` column without a `DEFAULT` if the table already has rows. Attempting it raises `OperationalError`. The existing `_migrate` method (lines 282–287) swallows all exceptions with a bare `except Exception: pass`. This means a badly written migration appears to succeed, but subsequent INSERTs into `items` that don't supply the new column fail with a constraint error that surfaces only at runtime, not during startup.

**Why it happens:**
The existing pattern `except Exception: pass` was intentionally used for idempotent column additions (safe for re-running). It is correct for that purpose but dangerous when the column definition itself is wrong, because the failure is invisible.

**Consequences:**
- `tmdb_id` column silently not added; SeasonViewWindow opens and crashes on first DB write
- `episode_progress` table silently not created if the `CREATE TABLE` statement has a syntax error
- No log entry if the migration is swallowed (the existing code logs before migration but not per-step)

**Prevention:**
- Add `tmdb_id TEXT` (nullable, no NOT NULL) — never `NOT NULL` without `DEFAULT` on an existing table in SQLite.
- For `CREATE TABLE episode_progress`, do NOT wrap in the bare-except block; execute it separately and let it surface errors, or check `sqlite_master` before and after.
- Log each migration step explicitly: `log.info("v7: added tmdb_id column")` inside the try, and `log.error(...)` in the except (not `pass`).
- After migration completes, run a quick schema verification: `PRAGMA table_info(items)` and assert `tmdb_id` is present.

**Detection:** After running the app against an existing v6 DB, query `PRAGMA table_info(items)` in a SQLite browser and confirm `tmdb_id` appears.

**Phase:** DB migration phase.

---

### Pitfall C4: Updating Tkinter widgets from a background thread causes silent corruption or crash on Windows

**What goes wrong:**
Tkinter is single-threaded. Any widget method called from a non-main thread (including `.config()`, `.insert()`, `.delete()`, setting `textvariable`, or creating `ImageTk.PhotoImage`) is undefined behavior on Windows — it either silently corrupts the event queue or raises `RuntimeError: main thread is not in main loop` non-deterministically. The existing code already solves this correctly for posters (line 1128: `app_root.after(0, _finalize)`). SeasonViewWindow will load episode data in a background thread and the pitfall is updating the episode list UI directly from that thread.

**Why it happens:**
The worker function finishes the API call and it feels natural to immediately populate the UI. The bug is intermittent (works 80% of the time in testing, crashes in production under load) because the race depends on thread scheduling.

**Consequences:**
- Intermittent `RuntimeError` or silent widget state corruption when multiple seasons are loaded in quick succession
- Checkbox state written from wrong thread can leave `IntVar` values desynced from the DB

**Prevention:**
All UI updates in SeasonViewWindow must go through `self.after(0, callback)` or `self.after(0, lambda: ...)`. The pattern used in the existing `fetch_tmdb_multi` worker (line 1754) is the correct template. Never call widget methods or create `ImageTk.PhotoImage` inside the thread worker body — only compute data there, then hand off to `after()`.

```python
# WRONG — inside thread worker
def _worker():
    data = fetch_season(tmdb_id, season_num)
    self._populate_episodes(data)   # crashes on Windows

# CORRECT
def _worker():
    data = fetch_season(tmdb_id, season_num)
    self.after(0, lambda d=data: self._populate_episodes(d))
```

**Detection:** Add a `threading.current_thread()` assert at the top of every method that touches widgets:
```python
assert threading.current_thread() is threading.main_thread(), "UI touched from worker thread"
```
Run this only in dev builds (behind an `if __debug__:` guard).

**Phase:** SeasonViewWindow implementation phase.

---

### Pitfall C5: `PhotoImage` garbage-collected in scrollable episode list

**What goes wrong:**
If episode thumbnails (or any image) are created inside a loop that builds rows in a scrollable frame, and the `ImageTk.PhotoImage` objects are only referenced by local variables, Python's GC collects them after the loop ends. The `Label` widget holds only a weak C-level reference. The image disappears (shows as blank/gray) after the first GC cycle — typically a few seconds after the window opens — without any error.

**Why it happens:**
The existing poster system avoids this by storing photos in `_poster_mem` (the LRU cache) and in `self._photo` on each card. In a dynamically built episode list, developers often forget to maintain a Python-level reference.

**Consequences:**
- Episode thumbnails disappear silently after window is open a few seconds
- Only reproducible intermittently depending on GC timing

**Prevention:**
For SeasonViewWindow, maintain a list `self._episode_photos = []` as an instance attribute. Append every `ImageTk.PhotoImage` created for episode rows to this list. The list lives as long as the window. Clear it only in the window's `on_destroy` handler.

**Note:** If SeasonViewWindow v1 does not show per-episode images (only text checkboxes), this pitfall does not apply. However it applies to any decorative images (season poster banner, network logo, etc.) displayed in the window.

**Phase:** SeasonViewWindow implementation phase.

---

## Moderate Pitfalls

---

### Pitfall M1: TMDb returns HTTP 404 for a valid season number that has no episodes yet

**What goes wrong:**
TMDb returns 404 for seasons that are announced but not yet populated — upcoming seasons, or seasons where all episodes are marked "TBA". The code must distinguish: 404 means "season does not exist in TMDb", not "the series has no seasons at all". If you check only for network errors (connection refused, timeout) and treat 404 as a generic failure, the UI either crashes or shows an unhelpful error.

**Prevention:**
Check `response.status_code` explicitly:
- 200 → parse episodes
- 404 → show "No hay datos de esta temporada en TMDb aún" — not an error, a valid state
- 401/403 → API key problem, show the setup wizard prompt
- 429 → rate limited, retry with exponential backoff (start at 1s, max 3 retries)
- 5xx → TMDb server error, show "TMDb no disponible, inténtalo más tarde"

Never call `.json()` on a non-200 response without checking status first — TMDb returns JSON error bodies that look valid but contain `{"status_code": 34, "status_message": "..."}`.

**Warning signs:** The existing `fetch_tmdb_detail` (line 864) calls `.json()` directly without status check. This is the pattern to NOT replicate in the season fetcher.

**Phase:** SeasonViewWindow API integration.

---

### Pitfall M2: Episode `episode_number` is not contiguous — gaps exist for unaired or removed episodes

**What goes wrong:**
A season may have episodes numbered 1, 2, 3, 5, 6 (episode 4 removed or renumbered after airing). If you store `episode_number` as a display index (0-based position in the list) instead of TMDb's actual `episode_number`, the `episode_progress` rows will reference wrong episodes after any TMDb data refresh that changes episode ordering.

**Prevention:**
Always use TMDb's `episode_number` field (from the `episodes` array in the season response) as the canonical identifier in `episode_progress`. The table PK should be `(user_id, tmdb_id, season_number, episode_number)` — never a position index.

**Phase:** DB schema design for `episode_progress`.

---

### Pitfall M3: State desync — marking episode watched in SeasonViewWindow does not refresh DetailWindow or ContentCard

**What goes wrong:**
The existing architecture uses `on_refresh` callbacks to propagate state changes. `ContentCard._cycle_status` calls `self.on_refresh()` which triggers a full grid re-render. SeasonViewWindow is a Toplevel opened from DetailWindow, which is itself a Toplevel opened from the grid. When an episode is marked watched, the grid card's progress indicator (a new v7 requirement) is stale until the user closes both windows and the grid re-renders.

**Why it happens:**
Toplevel windows in this app use `grab_set()` (modal), so the parent is blocked. But SeasonViewWindow needs to communicate upward through two layers: SeasonView → DetailWindow → ContentCard → grid. If SeasonViewWindow is not given a callback reference to the top-level `on_refresh`, the signal is lost.

**Consequences:**
- ContentCard shows "0/12 episodios" even after the user has just marked 3 episodes watched in SeasonViewWindow
- User closes SeasonViewWindow and sees stale data; they click refresh (F5) manually, which reloads the grid and destroys their current view position

**Prevention:**
Pass the top-level `on_refresh` callback (or a targeted refresh function) from the main app all the way to SeasonViewWindow. When SeasonViewWindow's `on_destroy` fires, call this callback. This gives a single refresh on window close rather than per-episode (which would be jarring). For real-time feedback, update only the progress label within SeasonViewWindow itself on each checkbox click — the grid card updates on close.

```python
class SeasonViewWindow(tk.Toplevel):
    def __init__(self, parent, item, user_id, db, on_refresh, app_root):
        ...
        self.on_grid_refresh = on_refresh
        self.protocol("WM_DELETE_WINDOW", self._on_close)

    def _on_close(self):
        self.on_grid_refresh()   # refresh grid once on close
        self.destroy()
```

**Phase:** SeasonViewWindow integration with DetailWindow and ContentCard.

---

### Pitfall M4: `check_same_thread=False` on SQLite connection + concurrent writes from episode checkboxes

**What goes wrong:**
The existing `Database` object uses `check_same_thread=False` (line 207), which disables SQLite's built-in guard. This is safe as long as writes are serialized. The poster ThreadPool only reads (downloads files). But SeasonViewWindow may fire rapid writes — the user quickly checks multiple episode checkboxes — while the main thread is also writing (e.g., a status change from a card). WAL mode (already enabled, line 210) helps with read/write concurrency but not write/write concurrency.

**Prevention:**
Wrap all DB writes in a `threading.Lock()`. The existing `_poster_lock` is file-system level. Create a separate `_db_write_lock` at the `Database` class level and acquire it around every `conn.execute(INSERT/UPDATE/DELETE)` + `conn.commit()`. Alternatively, route all writes through `root.after(0, ...)` to serialize them on the main thread — this is simpler and consistent with the existing Tkinter threading model.

**Phase:** Episode progress persistence implementation.

---

### Pitfall M5: Series in DB with `type='series'` but no TMDb match — SeasonViewWindow opens blank or crashes

**What goes wrong:**
The app supports adding series manually (the "Añadir desde plataforma externa" feature). A manually added series may have `tmdb_id = NULL` even after v7 migration. When the user clicks "Ver temporadas" on such a series, the SeasonViewWindow has no tmdb_id to call the API with.

**Prevention:**
In SeasonViewWindow's `__init__`, check `tmdb_id` before proceeding:
1. If `tmdb_id` is present → normal flow
2. If `tmdb_id` is NULL → show a search UI inside the window: "No encontramos esta serie en TMDb. Busca el título correcto:" with a text field and "Buscar" button. On confirmation, persist the resolved `tmdb_id` back to the `items` row.

This is better UX than an error dialog and recovers the data for future opens.

**Phase:** SeasonViewWindow, handled at window initialization.

---

## Minor Pitfalls

---

### Pitfall m1: Season 0 display name — showing "Temporada 0" confuses users

**What goes wrong:**
TMDb's season 0 has `name = "Specials"` (or localized equivalent in `es-ES`: "Especiales"). If you display the tab/header as "Temporada 0", users are confused. If you display TMDb's `name` field directly it may be in English even when requesting `language=es-ES` (TMDb translation coverage is inconsistent for specials).

**Prevention:**
Use TMDb's `name` field for the display label. Fall back to `"Temporada {season_number}"` only if `name` is empty. For season_number=0 without a name, use `"Especiales"`.

**Phase:** SeasonViewWindow UI construction.

---

### Pitfall m2: API rate limiting — 40 requests/10 seconds on TMDb free tier

**What goes wrong:**
Loading a series detail then N seasons calls (e.g., 8 seasons) hits the rate limit if done in parallel. TMDb returns HTTP 429 with a `Retry-After` header.

**Prevention:**
Load seasons sequentially (one at a time), not via `ThreadPoolExecutor` with all seasons submitted at once. Show a progress indicator ("Cargando temporada 2 de 8...") so the user sees activity. The existing `BatchImportWindow` pattern (lines 2149–2180) is the correct template.

**Phase:** SeasonViewWindow data loading.

---

### Pitfall m3: `SCHEMA_VERSION` constant not bumped — migration never runs

**What goes wrong:**
The constant `SCHEMA_VERSION = 6` (line 158) controls when `_migrate` persists the new version. If a developer adds the v7 migration block (`if v < 7: ...`) but forgets to change `SCHEMA_VERSION` to `7`, the migration code runs on every app launch (because `v < SCHEMA_VERSION` is never satisfied), wasting time and potentially causing duplicate-insert errors.

**Prevention:**
Change `SCHEMA_VERSION = 6` to `SCHEMA_VERSION = 7` as the first step of the migration work, before writing any migration code. Make the constant change and the `if v < 7:` block in the same commit.

**Phase:** First line of DB migration work.

---

### Pitfall m4: Closing SeasonViewWindow while background thread is still fetching

**What goes wrong:**
User opens SeasonViewWindow, season data starts loading in a background thread, user closes the window before loading finishes. The thread completes and calls `self.after(0, ...)` on a destroyed Toplevel. This raises `TclError: invalid command name ".!toplevel2"` — harmless but logged as an error and confusing.

**Prevention:**
Set a `self._destroyed = False` flag and check `self.winfo_exists()` (or the flag) in the `after()` callback before touching any widget. The existing poster system already uses this pattern (`_stop_event`). For SeasonViewWindow, a simple instance flag is sufficient:

```python
def _on_close(self):
    self._destroyed = True
    self.on_grid_refresh()
    self.destroy()

def _populate_episodes(self, data):
    if self._destroyed or not self.winfo_exists():
        return
    # ... build UI
```

**Phase:** SeasonViewWindow threading implementation.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|---|---|---|
| DB migration v6→v7 | C3: bare-except swallows migration errors; C2: tmdb_id column nullable | Log each step, verify with PRAGMA after migration |
| tmdb_id backfill for existing rows | C2: ambiguous title re-search picks wrong series | Only backfill when title match is exact; prompt user otherwise |
| SeasonViewWindow: season list | C1: range() instead of seasons[] array | Always iterate TMDb `seasons` array from TV detail response |
| SeasonViewWindow: API calls | M1: 404 treated as generic error; m2: rate limiting | Check status_code explicitly; load seasons sequentially |
| SeasonViewWindow: checkbox | C4: UI update from worker thread | All widget updates via `self.after(0, ...)` |
| SeasonViewWindow: images | C5: PhotoImage GC'd in loop | Maintain `self._episode_photos = []` instance list |
| Episode progress writes | M4: concurrent DB writes | Use DB-level write lock or serialize writes via `after()` |
| "Ver temporadas" button | M5: tmdb_id is NULL | Inline TMDb search recovery UI, not an error dialog |
| Window close | m4: after() on destroyed Toplevel | `winfo_exists()` guard in all after() callbacks |
| SCHEMA_VERSION constant | m3: constant not bumped | Bump constant before writing migration code |
| Season 0 / Specials | C1, m1: invisible or mislabeled | Use TMDb seasons[] array; use TMDb name field for display |
| State propagation | M3: progress badge stale on ContentCard | Pass on_refresh to SeasonViewWindow; call on window close |

---

## Sources

- Direct code inspection: `D:/VideoClub/Videoclub` (lines 158, 207–316, 853–893, 1069–1132, 1340–1360)
- TMDb TV API structure: field names and Season 0 behavior based on official TMDb API v3 documentation (verified against known API shape)
- SQLite ALTER TABLE constraints: official SQLite documentation (well-established limitation, HIGH confidence)
- Tkinter threading model: Python official documentation + well-known single-thread constraint (HIGH confidence)
- Project context: `D:/VideoClub/.planning/PROJECT.md`
