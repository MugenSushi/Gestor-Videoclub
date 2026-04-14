# Architecture: SeasonViewWindow Integration

**Project:** Videoclub STARTUP v7.0
**Researched:** 2026-04-14
**Scope:** Adding SeasonViewWindow to the existing single-file Python/Tkinter app

---

## Component Boundaries

```
App (tk.Tk)
├── MainGrid  (virtual scroll, pagination)
├── DetailWindow (tk.Toplevel, grab_set)  ← existing
│   └── [button] "Ver temporadas"
│       └── SeasonViewWindow (tk.Toplevel)  ← new
│           ├── ttk.Notebook  (one tab per season)
│           │   └── SeasonTab (tk.Frame per tab)
│           │       └── ScrollableEpisodeList
│           │           └── EpisodeRow × N  (checkbox + label)
│           └── ProgressBar per season (tk.Label, computed)
│
Database
├── items            (existing, needs tmdb_id column)
├── watched          (existing, item-level status)
├── episode_progress (new, episode-level watched state)
│
TMDb API
├── /tv/{id}/season/{n}  ← new endpoint
└── (existing) /tv/{id}?append_to_response=credits,watch/providers
```

---

## Question 1: Opening SeasonViewWindow Without Breaking grab_set

`DetailWindow` calls `self.grab_set()` in `__init__`, making it modal. The correct pattern to open a child Toplevel without conflict is:

**Do not release DetailWindow's grab. Instead, let SeasonViewWindow take grab on top of it.**

Tkinter's grab model is a stack: the most recent `grab_set()` wins. When SeasonViewWindow is destroyed, grab returns to DetailWindow automatically if DetailWindow re-calls `grab_set` in a `<Destroy>` binding on the child.

```python
# Inside DetailWindow._build() — add this button for series only:
if itype == "series":
    tmdb_id = self.item.get("tmdb_id", "")
    num_seasons = self.item.get("num_seasons", 0)
    if tmdb_id:
        tk.Button(
            left, text="📺 Ver temporadas",
            bg=C["accent"], fg="white",
            font=(FONT_FAMILY, 9), relief="flat", cursor="hand2",
            command=lambda: self._open_seasons(tmdb_id, num_seasons)
        ).pack(fill="x", pady=(6, 0))

def _open_seasons(self, tmdb_id: str, num_seasons: int):
    win = SeasonViewWindow(
        self,                      # parent = DetailWindow
        tmdb_id=tmdb_id,
        series_title=self.item["title"],
        num_seasons=num_seasons,
        user_id=self.user_id,
        db=self.db,
        app_root=self.app_root,
    )
    # Re-grab after child closes (child takes grab while open)
    win.bind("<Destroy>", lambda e: self.grab_set() if self.winfo_exists() else None)
```

```python
class SeasonViewWindow(tk.Toplevel):
    def __init__(self, parent, *, tmdb_id, series_title,
                 num_seasons, user_id, db, app_root):
        super().__init__(parent)
        self.grab_set()   # takes grab away from DetailWindow while open
        self.title(f"Temporadas — {series_title[:50]}")
        self.configure(bg=C["bg"])
        self.resizable(True, True)
        self.bind("<Escape>", lambda e: self.destroy())
        ...
```

Key point: never call `grab_release()` on DetailWindow explicitly. Tkinter handles this automatically when SeasonViewWindow takes grab. When SeasonViewWindow is destroyed, the `<Destroy>` binding re-sets grab on DetailWindow.

---

## Question 2: Async Season Loading Pattern (ThreadPoolExecutor + after)

Follow the exact same pattern already used by `get_poster_async`. The rule the codebase enforces is: **network in thread, UI update via `app_root.after(0, callback)`**.

Use the existing `_poster_pool` or create a dedicated small pool. A dedicated pool is cleaner because season loads are heavier than poster fetches and you don't want them to block poster slots.

```python
# Module-level alongside _poster_pool
_season_pool = ThreadPoolExecutor(max_workers=2, thread_name_prefix="season")
```

Shutdown alongside the poster pool (already done at line 3616):
```python
_season_pool.shutdown(wait=False)
```

Loading pattern inside SeasonViewWindow:

```python
def _load_season(self, season_number: int, tab_frame: tk.Frame):
    """Called from main thread. Spawns worker, updates UI via after()."""
    # 1. Show spinner immediately (main thread)
    spinner = tk.Label(tab_frame, text="Cargando...", bg=C["bg"], fg=C["muted"],
                       font=(FONT_FAMILY, 10))
    spinner.pack(pady=20)

    def _worker():
        data = fetch_tmdb_season(self._tmdb_id, season_number)  # blocks in thread
        # 2. Schedule UI update on main thread
        self.app_root.after(0, lambda: _on_done(data))

    def _on_done(data):
        if not self.winfo_exists():
            return
        spinner.destroy()
        if data:
            self._render_episode_list(tab_frame, season_number, data)
        else:
            tk.Label(tab_frame, text="No se pudo cargar la temporada.",
                     bg=C["bg"], fg=C["muted"]).pack(pady=20)

    _season_pool.submit(_worker)
```

The fetch function follows the same shape as `fetch_tmdb_detail`:

```python
def fetch_tmdb_season(series_id: str, season_number: int) -> dict:
    """Returns TMDb season data: episodes list with name, overview, air_date."""
    key = CFG.get("tmdb_api_key", "")
    if not key or not series_id:
        return {}
    try:
        url = (f"https://api.themoviedb.org/3/tv/{series_id}"
               f"/season/{season_number}"
               f"?api_key={key}&language=es-ES")
        return requests.get(url, timeout=10).json()
    except Exception as e:
        log.error(f"fetch_tmdb_season: {e}")
    return {}
```

Do not add `@lru_cache` here — episode data can change (new episodes air) and is not as stable as credits. Cache per session using an instance dict `self._season_cache: dict[int, dict]` on SeasonViewWindow if you want to avoid re-fetching when the user switches back to an already-loaded tab.

---

## Question 3: Upfront vs. Lazy Season Fetching

**Load lazily, triggered by tab selection.** Do not fetch all seasons on window open.

Rationale:
- A long-running series (e.g., Grey's Anatomy, 20 seasons) would fire 20 parallel API requests immediately. TMDb rate-limits at 40 requests/10 seconds, which this would blow through.
- The user may only care about the current season. Fetching all upfront wastes time and bandwidth.
- The tab-switch event (`<<NotebookTabChanged>>`) is the natural trigger.

Implementation:

```python
def _build_notebook(self):
    self._nb = ttk.Notebook(self, style="Dark.TNotebook")
    self._nb.pack(fill="both", expand=True, padx=10, pady=10)
    self._loaded_seasons: set[int] = set()
    self._season_cache: dict[int, dict] = {}

    for n in range(1, self._num_seasons + 1):
        frame = tk.Frame(self._nb, bg=C["bg"])
        self._nb.add(frame, text=f"T{n}")
        # Store the frame reference for later population
        self._season_frames[n] = frame

    self._nb.bind("<<NotebookTabChanged>>", self._on_tab_changed)
    # Trigger load of the first tab immediately
    self._on_tab_changed(None)

def _on_tab_changed(self, event):
    idx = self._nb.index("current")
    season_number = idx + 1   # tabs are 0-indexed, seasons are 1-indexed
    if season_number not in self._loaded_seasons:
        self._loaded_seasons.add(season_number)
        self._load_season(season_number, self._season_frames[season_number])
```

---

## Question 4: episode_progress Table Structure

Match the patterns already established in the Database class:
- Use `INTEGER PRIMARY KEY AUTOINCREMENT` for surrogate keys where natural keys are compound.
- Use `ON CONFLICT ... DO UPDATE` (upsert) for toggle operations, matching how `set_status` works.
- Keep `user_id` as a FK so per-user tracking works automatically (multi-user is already supported).
- Use `item_id` (FK to items.id) rather than `title` as the series reference — this is more correct than the existing `watched` table which uses `title` as FK (a design debt the codebase acknowledges implicitly; don't repeat it for the new table).

```sql
CREATE TABLE IF NOT EXISTS episode_progress (
    id             INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id        INTEGER NOT NULL,
    item_id        INTEGER NOT NULL,          -- FK to items.id (not title)
    season_number  INTEGER NOT NULL,
    episode_number INTEGER NOT NULL,
    watched        INTEGER NOT NULL DEFAULT 0, -- 0=no, 1=yes (SQLite has no BOOL)
    watched_at     TEXT,                       -- datetime('now') when marked
    UNIQUE (user_id, item_id, season_number, episode_number),
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (item_id) REFERENCES items(id)
);
CREATE INDEX IF NOT EXISTS idx_ep_lookup
    ON episode_progress(user_id, item_id, season_number);
```

Database methods to add (following existing naming conventions):

```python
def get_episode_progress(self, user_id: int, item_id: int,
                         season_number: int) -> dict[int, bool]:
    """Returns {episode_number: watched} for one season."""
    rows = self.conn.execute(
        "SELECT episode_number, watched FROM episode_progress"
        " WHERE user_id=? AND item_id=? AND season_number=?",
        (user_id, item_id, season_number)
    ).fetchall()
    return {r["episode_number"]: bool(r["watched"]) for r in rows}

def set_episode_watched(self, user_id: int, item_id: int,
                        season_number: int, episode_number: int,
                        watched: bool) -> bool:
    try:
        self.conn.execute(
            "INSERT INTO episode_progress"
            " (user_id, item_id, season_number, episode_number, watched, watched_at)"
            " VALUES (?,?,?,?,?,datetime('now'))"
            " ON CONFLICT(user_id, item_id, season_number, episode_number)"
            " DO UPDATE SET watched=excluded.watched, watched_at=excluded.watched_at",
            (user_id, item_id, season_number, episode_number, int(watched))
        )
        self.conn.commit()
        return True
    except Exception as e:
        log.error(f"set_episode_watched: {e}")
        return False

def get_season_progress(self, user_id: int, item_id: int,
                        season_number: int) -> tuple[int, int]:
    """Returns (watched_count, total_count) for a season."""
    row = self.conn.execute(
        "SELECT COUNT(*) as total,"
        " SUM(CASE WHEN watched=1 THEN 1 ELSE 0 END) as done"
        " FROM episode_progress"
        " WHERE user_id=? AND item_id=? AND season_number=?",
        (user_id, item_id, season_number)
    ).fetchone()
    if not row or not row["total"]:
        return (0, 0)
    return (row["done"] or 0, row["total"])
```

---

## Question 5: Adding tmdb_id to items — Migration Pattern

The existing `_migrate()` method already uses `ALTER TABLE ... ADD COLUMN` wrapped in a try/except to handle "column already exists" gracefully (lines 272–287). Add `tmdb_id` the same way, and bump `SCHEMA_VERSION` from 6 to 7.

**Step 1 — Bump the constant at the top of the file:**

```python
SCHEMA_VERSION = 7   # was 6
```

**Step 2 — Add to the ALTER TABLE loop in `_migrate()`:**

```python
for col, tbl, defn in [
    ("added_at",    "items",   "TEXT DEFAULT (datetime('now'))"),
    ("status",      "watched", "TEXT NOT NULL DEFAULT 'watched'"),
    ("user_rating", "watched", "INTEGER"),
    ("notes",       "watched", "TEXT"),
    ("director",    "items",   "TEXT"),
    ("actors",      "items",   "TEXT"),
    ("studio",      "items",   "TEXT"),
    ("streaming",   "items",   "TEXT"),
    # v7
    ("tmdb_id",     "items",   "TEXT"),
    ("num_seasons", "items",   "INTEGER"),
]:
    try:
        self.conn.execute(f"ALTER TABLE {tbl} ADD COLUMN {col} {defn}")
        self.conn.commit()
    except Exception:
        pass
```

**Step 3 — Add the `episode_progress` table and its index inside the new version gate:**

```python
if v < 7:
    self.conn.executescript("""
        CREATE TABLE IF NOT EXISTS episode_progress (
            id             INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id        INTEGER NOT NULL,
            item_id        INTEGER NOT NULL,
            season_number  INTEGER NOT NULL,
            episode_number INTEGER NOT NULL,
            watched        INTEGER NOT NULL DEFAULT 0,
            watched_at     TEXT,
            UNIQUE (user_id, item_id, season_number, episode_number),
            FOREIGN KEY (user_id) REFERENCES users(id),
            FOREIGN KEY (item_id) REFERENCES items(id)
        );
        CREATE INDEX IF NOT EXISTS idx_ep_lookup
            ON episode_progress(user_id, item_id, season_number);
    """)
    self.conn.commit()
```

The `try/except` in the loop means this is safe to run against existing v6 databases — the ALTER TABLE for `tmdb_id` silently skips if the column already exists from a partial run. The `v < 7` gate for `episode_progress` is idempotent via `CREATE TABLE IF NOT EXISTS`.

**Step 4 — Persist `tmdb_id` when adding items.** The `add_item()` method needs `tmdb_id` and `num_seasons` parameters, and the INSERT must include them:

```python
def add_item(self, title, itype, rating, genres, year, plot, poster_url,
             director="", actors="", studio="", streaming="",
             tmdb_id="", num_seasons=0, user_id: int = 0) -> bool:
```

The call sites in `SmartSearchDialog._finish_manual()` already have `_tmdb_id` in the `data` dict (line 1940). Pass it through `_extract_extra()` or explicitly at the call site.

---

## Question 6: Widget Choice for Episode List

**Recommendation: Canvas + inner Frame with Scrollbar (the "scrollable frame" pattern).**

Do not use:
- `Listbox` — single-column text only, no checkboxes inside items, no custom row layout.
- `ttk.Treeview` — works and supports checkboxes via images, but is complex to style to match the dark theme. Styling ttk widgets in dark mode requires `ttk.Style` configuration that conflicts with the app's current approach of configuring everything via the `C` color dict on plain `tk` widgets.

Use a `Canvas` + `Frame` inside it:

```python
def _render_episode_list(self, parent: tk.Frame, season_number: int, data: dict):
    episodes = data.get("episodes", [])

    # Load existing progress from DB
    progress = self.db.get_episode_progress(
        self.user_id, self._item_id, season_number
    )

    # Scrollable container
    canvas = tk.Canvas(parent, bg=C["bg"], highlightthickness=0)
    scrollbar = tk.Scrollbar(parent, orient="vertical", command=canvas.yview)
    inner = tk.Frame(canvas, bg=C["bg"])

    inner.bind("<Configure>",
               lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
    canvas.create_window((0, 0), window=inner, anchor="nw")
    canvas.configure(yscrollcommand=scrollbar.set)

    canvas.pack(side="left", fill="both", expand=True)
    scrollbar.pack(side="right", fill="y")

    # Mouse-wheel scroll (Windows)
    canvas.bind("<MouseWheel>",
                lambda e: canvas.yview_scroll(-1 * (e.delta // 120), "units"))

    # Season progress label at top
    watched_count = sum(1 for ep in episodes
                        if progress.get(ep.get("episode_number"), False))
    total = len(episodes)
    prog_lbl = tk.Label(
        inner, bg=C["bg"], fg=C["muted"],
        font=(FONT_FAMILY, 9),
        text=f"{watched_count}/{total} episodios vistos"
    )
    prog_lbl.pack(anchor="w", padx=8, pady=(4, 8))

    # One row per episode
    for ep in episodes:
        ep_num  = ep.get("episode_number", 0)
        ep_name = ep.get("name", f"Episodio {ep_num}")
        ep_date = ep.get("air_date", "")
        is_watched = progress.get(ep_num, False)
        _make_episode_row(inner, ep_num, ep_name, ep_date, is_watched,
                          season_number, prog_lbl, watched_count, total)
```

Episode row as a helper (keep it a nested function or a small method — not a class, to stay consistent with the codebase's style):

```python
def _make_episode_row(parent, ep_num, ep_name, ep_date, is_watched,
                      season_number, prog_lbl, watched_ref, total):
    row = tk.Frame(parent, bg=C["surface"], pady=2)
    row.pack(fill="x", padx=8, pady=2)

    var = tk.BooleanVar(value=is_watched)

    def _toggle():
        new_state = var.get()
        # DB write — safe to do on main thread (SQLite WAL mode, fast write)
        self.db.set_episode_watched(
            self.user_id, self._item_id, season_number, ep_num, new_state
        )
        # Update progress label
        nonlocal watched_ref
        watched_ref += 1 if new_state else -1
        prog_lbl.config(text=f"{watched_ref}/{total} episodios vistos")

    chk = tk.Checkbutton(
        row, variable=var, command=_toggle,
        bg=C["surface"], fg=C["text"],
        activebackground=C["surface"], selectcolor=C["bg"],
        relief="flat", cursor="hand2"
    )
    chk.pack(side="left", padx=(6, 4))

    tk.Label(row,
             text=f"{ep_num:02d}. {ep_name}",
             bg=C["surface"], fg=C["text"] if is_watched else C["muted"],
             font=(FONT_FAMILY, 9),
             anchor="w"
    ).pack(side="left", fill="x", expand=True)

    if ep_date:
        tk.Label(row, text=ep_date[:7],  # YYYY-MM
                 bg=C["surface"], fg=C["border"],
                 font=(FONT_FAMILY, 8)).pack(side="right", padx=6)
```

The `Canvas + Frame` pattern is already established in the codebase (plot_txt uses a Scrollbar at line 1544). The mousewheel binding is Windows-specific — that is fine because the PROJECT.md states Windows 10/11 is the primary target.

---

## Data Flow

```
User opens DetailWindow for a series
  → DetailWindow._build() renders "Ver temporadas" button (only if itype == "series")
  → user clicks button → DetailWindow._open_seasons(tmdb_id, num_seasons)
    → SeasonViewWindow.__init__(parent=DetailWindow, ...)
      → grab_set() (takes modal focus)
      → _build_notebook(): creates N empty tab frames
      → _on_tab_changed(): fires immediately for tab 0

      [Tab 0 load]
      → _load_season(season=1, frame=tab_frame)
        → shows spinner (main thread)
        → _season_pool.submit(_worker)
          [thread]
          → fetch_tmdb_season(tmdb_id, 1)  → GET /tv/{id}/season/1
          → app_root.after(0, _on_done)
          [main thread]
          → _on_done(data)
            → db.get_episode_progress(user_id, item_id, 1)  → dict
            → _render_episode_list(frame, 1, data)
              → Canvas + inner Frame
              → one EpisodeRow per episode (checkbox + name + date)

      [User checks a checkbox]
      → _toggle()
        → db.set_episode_watched(user_id, item_id, 1, ep_num, True)  [WAL, fast]
        → update progress label text in-place

      [User switches to tab 1]
      → <<NotebookTabChanged>> → _on_tab_changed()
      → season 2 not in _loaded_seasons → _load_season(2, ...)
      → (same async flow)

      [User closes SeasonViewWindow]
      → <Destroy> binding on SeasonViewWindow in DetailWindow fires
      → DetailWindow.grab_set() restores modal focus to DetailWindow
```

---

## Build Order (Suggested Phases)

### Phase 1 — Database Foundation (no UI, fully testable)

1. Bump `SCHEMA_VERSION` to 7.
2. Add `tmdb_id`, `num_seasons` to the ALTER TABLE loop in `_migrate()`.
3. Add `v < 7` block creating `episode_progress` table and index.
4. Add `tmdb_id=`, `num_seasons=` params to `add_item()` and its INSERT.
5. Add `get_episode_progress()`, `set_episode_watched()`, `get_season_progress()` to `Database`.
6. Verify migration runs against existing v6 db without data loss (manual test with real db file).

### Phase 2 — TMDb Season Fetch

1. Add `fetch_tmdb_season(series_id, season_number) -> dict` at module level, alongside `fetch_tmdb_detail`.
2. Add `_season_pool = ThreadPoolExecutor(max_workers=2, ...)` at module level.
3. Add `_season_pool.shutdown(wait=False)` in the app shutdown handler (line 3616).
4. Test the fetch function in isolation before wiring UI.

### Phase 3 — SeasonViewWindow Skeleton

1. Create `SeasonViewWindow(tk.Toplevel)` class with `__init__`, `grab_set`, `_center`, `_build_notebook`.
2. `_build_notebook` creates N tab frames, binds `<<NotebookTabChanged>>`.
3. `_on_tab_changed` dispatches to `_load_season`.
4. `_load_season` runs async worker, calls `_render_episode_list` on done.
5. `_render_episode_list` builds Canvas + scrollable inner Frame + EpisodeRow per episode.
6. Wire checkboxes to `db.set_episode_watched()`.

### Phase 4 — DetailWindow Integration

1. In `DetailWindow._build()`, add the "Ver temporadas" button conditioned on `itype == "series"` AND `tmdb_id` being non-empty.
2. Add `_open_seasons()` method with the `<Destroy>` / `grab_set` re-binding.
3. Ensure `item` dict passed to DetailWindow includes `tmdb_id` and `num_seasons` — these come from the `items` table row after Phase 1 schema changes and are populated when the user adds a series via SmartSearchDialog.

### Phase 5 — Progress Indicator on Grid Cards (Optional, Deferred)

1. After episode_progress exists, add a query in the main grid render to fetch watched-episode counts for series.
2. Overlay a progress bar or fraction label on series cards.
3. This is deferred because it requires a query per visible card on every grid refresh — profile before implementing to ensure it stays fast with pagination.

---

## Key Constraints to Enforce

| Constraint | Enforcement |
|---|---|
| UI only from main thread | All `after(0, ...)` callbacks check `winfo_exists()` before touching widgets |
| SQLite thread safety | `check_same_thread=False` + WAL mode already set; episode writes are always from main thread (checkbox callback), so no additional locking needed |
| No new modules | Everything stays in the single `Videoclub` file |
| Dark theme consistency | Use `C["bg"]`, `C["surface"]`, `C["muted"]`, `C["text"]`, `C["border"]` — no hardcoded colors |
| tmdb_id availability | The `_tmdb_id` key is already present in TMDb search results (line 844) and already used in `fetch_tmdb_detail` calls (line 1940, 2165). It must be persisted through `add_item()` for SeasonViewWindow to work without re-searching. |
| Series with no tmdb_id | Items added via OMDb-only path or manual entry won't have a tmdb_id. The "Ver temporadas" button must be hidden for these (guard: `if tmdb_id`). |

---

## Pitfalls Specific to This Integration

**grab_set re-entry**: If DetailWindow is destroyed while SeasonViewWindow is open (e.g., user calls `self.destroy()` on DetailWindow from another path), the `<Destroy>` callback fires on a dead widget. Guard with `if self.winfo_exists()` on both sides.

**Canvas scroll on Windows**: The `<MouseWheel>` event works on Windows but the binding must be on the Canvas itself, not the inner Frame. If the inner Frame catches the event first, scrolling breaks. Add the same binding to `inner` and have it call `canvas.yview_scroll`.

**ttk.Notebook styling**: The default ttk.Notebook tab background is light gray on Windows, clashing with the dark theme. Apply a minimal `ttk.Style` override. This is the one place where ttk styling is needed:

```python
style = ttk.Style()
style.theme_use("default")
style.configure("Dark.TNotebook",        background=C["bg"])
style.configure("Dark.TNotebook.Tab",    background=C["surface"],
                foreground=C["muted"],   padding=[10, 4])
style.map("Dark.TNotebook.Tab",
          background=[("selected", C["bg"])],
          foreground=[("selected", C["text"])])
```

**num_seasons source**: TMDb's `/tv/{id}` response includes `number_of_seasons`. This is already fetched by `fetch_tmdb_detail()` (line 853, the base URL is `/tv/{tmdb_id}?append_to_response=credits,...`). The `d` dict inside that function contains `number_of_seasons`. Add it to the return dict from `fetch_tmdb_detail` for `media_type == "tv"` and persist it to `items.num_seasons` at add time. This avoids a separate API call just to know how many tabs to create.

**Episode numbers vs. list index**: Never use the list index as the episode number. TMDb sometimes includes a "special" episode (episode_number=0) or starts mid-season. Always use `ep.get("episode_number")` as the key in `episode_progress`.

---

## Sources

- Codebase analysis: `D:/VideoClub/Videoclub` lines 200–500 (Database), 840–893 (fetch_tmdb_detail), 1060–1132 (get_poster_async), 1340–1601 (DetailWindow)
- TMDb API endpoint structure: `/tv/{series_id}/season/{season_number}` (consistent with existing usage pattern at line 861)
- Tkinter modal grab behavior: documented in Tcl/Tk reference — grab is a global stack, newest caller wins
- SQLite WAL mode + `check_same_thread=False`: established at lines 207–209
