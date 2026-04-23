# Phase 2: SeasonViewWindow — Research

**Researched:** 2026-04-17
**Domain:** Python/Tkinter modal Toplevel — ttk.Notebook tabs, Canvas+Frame scroll, async ThreadPoolExecutor, SQLite episode progress
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01**: Season list fetched asynchronously — full-window spinner while `fetch_tmdb_detail(tmdb_id, "tv")` runs in background pool. Build ttk.Notebook only after seasons array arrives. Never call synchronously in `__init__`.
- **D-02**: Error fallback = `tk.Label` with `"No se pudo obtener las temporadas. Comprueba tu conexión."` + `"Cerrar"` button. No integer-based T1/T2 fallback tabs.
- **D-03**: `grab_set()` in `__init__`. `DetailWindow._open_seasons()` binds `win.bind("<Destroy>", lambda e: self.grab_set() if self.winfo_exists() else None)`.
- **D-04**: `protocol("WM_DELETE_WINDOW", self._on_close)` + `bind("<Escape>", lambda e: self._on_close())`. Both call `on_refresh` then `destroy`.
- **D-05**: Season list from TMDb `seasons[]` array, field `s["season_number"]`. Never `range(1, num_seasons+1)`. Tab labels from `s["name"]`; fallback `"Especiales"` for season_number=0 empty name; fallback `f"T{season_number}"` for numbered with empty name.
- **D-06**: Each tab = empty `tk.Frame(bg=C["bg"])` populated lazily on `<<NotebookTabChanged>>`. `_loaded_seasons: set` tracks loaded. `self.after(0, self._on_tab_changed)` triggers first tab after notebook build.
- **D-07**: Canvas + inner Frame (not Treeview, not Listbox). MouseWheel bound to BOTH canvas AND inner Frame.
- **D-08**: Episode row = Checkbutton + `"{ep_num:02d}. {ep_name}"` label + air date `ep_date[:7]` right-aligned. Future episodes: `state=DISABLED`, `fg=C["muted"]`.
- **D-09**: Checkbox toggle writes synchronously to SQLite via `db.set_episode_watched()` on main thread. Updates progress counter label in-place.
- **D-10**: Bulk-mark skips future episodes (`air_date > today`). Marks only episodes with `air_date <= today` or empty `air_date`. Calls `set_episode_watched` per-episode in a loop.
- **D-11**: Counter format `"X / Y episodios vistos"`. Updates by recounting BooleanVars. When `X == Y`: `fg=C["seen"]`. Initialized from `db.get_episode_progress()` on tab load.
- **D-12**: Constructor signature `SeasonViewWindow(parent, item: dict, user_id: int, db: Database, on_refresh, app_root: tk.Tk)`. `item["id"]` → `item_id`, `item["tmdb_id"]` → TMDb series ID, `item["title"]` → window title.
- **D-13**: Insert class after `DetailWindow` (after line 1748). Section header `# ══... 10.5. SEASON VIEW WINDOW ...══`.
- **D-14**: Dedicated `_season_pool = ThreadPoolExecutor(max_workers=2)` near line 161.

### Claude's Discretion

- "Marcar toda" skip-future behavior (D-10)
- Constructor signature alignment (D-12)
- File insertion point (D-13)
- Worker pool (D-14)

### Deferred Ideas (OUT OF SCOPE)

- Window geometry memory — always open at minimum 600×480
- "Marcar toda" → "Desmarcar" toggle text — v1 out of scope
- Disabled "Ver temporadas" button for `tmdb_id = NULL` series — Phase 3 (BACK-01/02)
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| UI-01 | "Ver temporadas" button in DetailWindow — visible only for `type=series` with `tmdb_id` | Integration point confirmed: `_build()` left column after "🗑 Eliminar" button at line 1572 |
| UI-02 | SeasonViewWindow opens with `grab_set()`; restores DetailWindow grab on close | Pattern confirmed in ARCHITECTURE.md §Question 1; `<Destroy>` binding is the mechanism |
| UI-03 | ttk.Notebook tabs + `<<NotebookTabChanged>>`; lazy load only active tab | Confirmed: `_loaded_seasons: set` guard; `self.after(0, self._on_tab_changed)` for first tab |
| UI-04 | Progress counter "X / Y episodios vistos" per tab header | Counter recomputed from BooleanVars on each toggle; `fg=C["seen"]` when complete |
| UI-05 | Canvas+Frame scrollable episode list; dark theme; number, title, air date per row | Canvas+Frame pattern confirmed from lines 3492–3501; MouseWheel on both canvas and inner |
| UI-06 | Checkbutton with BooleanVar; toggle writes to SQLite immediately | `db.set_episode_watched()` verified present in codebase at line 396 |
| UI-07 | "Marcar toda la temporada" button in season header | Iterates `_season_cache[season_number]["episodes"]`; skips `air_date > today` |
| UI-08 | Future episodes: grey color, non-interactive checkbox | `state=DISABLED`, `fg=C["muted"]`; compare `air_date` to `datetime.date.today().isoformat()` |
| UI-09 | Loading spinner visible during fetch; error visible on failure | Spinner label show/hide pattern identical to poster loading; error label for 404 vs network |
| UI-10 | `on_refresh` called when SeasonViewWindow closes | `_on_close()` calls `self.on_refresh()` before `self.destroy()` |
</phase_requirements>

---

## Summary

Phase 2 builds `SeasonViewWindow` — a `tk.Toplevel` modal opened from `DetailWindow` for series items with a valid `tmdb_id`. All Phase 1 infrastructure (DB methods, API functions, `_season_pool` placeholder, schema v7) was confirmed present in the codebase during research. The phase is purely UI work: no new DB methods, no new API functions, no schema changes.

The technical stack is identical to what is already in the file: plain `tk`/`ttk`, the `C` color dict, `FONT_FAMILY`, `ThreadPoolExecutor`, `after(0, ...)` for UI updates, and `winfo_exists()` guards. The Canvas+Frame scrollable pattern exists at lines 3492–3501 and can be copied verbatim. The `DetailWindow.__init__` structure (lines 1488–1504) is the direct template for `SeasonViewWindow.__init__`.

The only new global needed is `_season_pool = ThreadPoolExecutor(max_workers=2, thread_name_prefix="season")` at line 161 (alongside `_poster_pool`). Everything else stays in the single `Videoclub` file as a new class inserted at line 1749 (between `DetailWindow` and `SmartSearchDialog`).

**Primary recommendation:** Follow the exact ARCHITECTURE.md Data Flow diagram — init → grab_set → async seasons fetch → build notebook → lazy tab load → render episode rows. Every deviation from the established `after(0, ...)`/`winfo_exists()` threading pattern is a defect waiting to surface.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Modal window management | Frontend (Tkinter Toplevel) | — | `grab_set` / `<Destroy>` / `protocol` all Tkinter-owned |
| Season list fetch | Background thread (`_season_pool`) | Main thread (render via `after`) | Network I/O must not block event loop |
| Episode list render | Main thread | — | All widget creation must be on main thread |
| Watched state read | Main thread (on tab load) | — | `db.get_episode_progress()` is fast; called from `_on_done` (already `after`-dispatched) |
| Watched state write | Main thread (checkbox callback) | — | `_toggle()` fires on main thread; `set_episode_watched()` is a synchronous upsert |
| Progress counter update | Main thread | — | Label `.config(text=...)` in-place; recount BooleanVars |
| Bulk mark season | Main thread | — | Loop over episodes in `_season_cache`; per-episode `set_episode_watched()` |
| Tab lazy load guard | Main thread | — | `_loaded_seasons: set[int]` check in `_on_tab_changed` |

---

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| tkinter (tk + ttk) | stdlib | All widgets, modal, scrollable frame | Already the entire app UI framework |
| threading.ThreadPoolExecutor | stdlib | Background season fetch | Already used for poster loading (`_poster_pool`) |
| sqlite3 (via Database class) | stdlib | Episode progress persistence | Existing DB layer; methods already added in Phase 1 |
| datetime | stdlib | Air date comparison for future episodes | Compare `ep["air_date"]` to `datetime.date.today().isoformat()` |

### No New Dependencies

This phase introduces zero new packages. All tooling is stdlib or already imported.

**Installation:** None required.

---

## Architecture Patterns

### System Architecture Diagram

```
User click "📺 Ver temporadas"
  │
  ▼
DetailWindow._open_seasons(tmdb_id, num_seasons)
  │  instantiate SeasonViewWindow
  │  bind <Destroy> → self.grab_set()
  ▼
SeasonViewWindow.__init__
  │  grab_set()                     ← takes modal focus
  │  show full-window spinner
  │  _season_pool.submit(_fetch_seasons_worker)
  │
  ├─[thread]─ fetch_tmdb_detail(tmdb_id, "tv")
  │               returns seasons[] array
  │           app_root.after(0, _on_seasons_done)
  │
  └─[main]── _on_seasons_done(seasons)
               │  destroy spinner
               │  _build_notebook(seasons)
               │    create N empty tk.Frame tabs
               │    bind <<NotebookTabChanged>>
               │  self.after(0, self._on_tab_changed)  ← trigger tab 0
               ▼
          _on_tab_changed()
               │  if season_number in _loaded_seasons → return
               │  _loaded_seasons.add(season_number)
               │  show spinner in tab frame
               │  _season_pool.submit(_worker)
               │
               ├─[thread]─ fetch_tmdb_season_episodes(tmdb_id, season_number)
               │           app_root.after(0, lambda d=data: _on_done(d))
               │
               └─[main]── _on_done(data)
                            │  winfo_exists() guard
                            │  destroy spinner
                            │  if data → _render_episode_list()
                            │  else   → error label
                            ▼
                       _render_episode_list(tab_frame, season_number, data)
                            │  db.get_episode_progress(user_id, item_id, season_number)
                            │  build Canvas + Scrollbar + inner Frame
                            │  per episode: BooleanVar + Checkbutton + labels
                            │  season header: progress counter + "Marcar toda" button
                            ▼
                       User toggles checkbox
                            │  _toggle()
                            │  db.set_episode_watched(...)
                            │  recount BooleanVars → update progress counter label
                            ▼
                       User closes window
                            │  _on_close()
                            │  self.on_refresh()         ← propagates to grid
                            │  self.destroy()
                            ▼
                       <Destroy> fires on SeasonViewWindow
                            └─ DetailWindow.grab_set() restores focus
```

### Recommended Project Structure

No new files. Single-file app: insert `SeasonViewWindow` class at line 1749.

```
Videoclub  (single file, ~3700 lines after Phase 2)
├── [line ~161]  _season_pool = ThreadPoolExecutor(max_workers=2, ...)
├── [line ~1488] class DetailWindow
│   ├── _build(): add "📺 Ver temporadas" button (conditional)
│   └── _open_seasons(): new method
├── [line ~1749] # ══ 10.5. SEASON VIEW WINDOW ══
│   └── class SeasonViewWindow(tk.Toplevel)
└── [line ~3620] _season_pool.shutdown(wait=False)  (alongside _poster_pool.shutdown)
```

### Pattern 1: Async Season Load (ThreadPoolExecutor + after)

**What:** Background fetch, result dispatched to main thread via `after(0, ...)`.
**When to use:** Any network call that must not block the event loop.

```python
# Source: ARCHITECTURE.md §Question 2 + existing get_poster_async pattern (lines 1238–1273)

def _load_season(self, season_number: int, tab_frame: tk.Frame):
    spinner = tk.Label(tab_frame, text="Cargando temporada...",
                       bg=C["bg"], fg=C["muted"], font=(FONT_FAMILY, 10))
    spinner.pack(pady=16)

    def _worker():
        data = fetch_tmdb_season_episodes(self._tmdb_id, season_number)
        self.app_root.after(0, lambda d=data: _on_done(d))

    def _on_done(data):
        if self._destroyed or not self.winfo_exists():
            return
        spinner.destroy()
        if data.get("episodes"):
            self._render_episode_list(tab_frame, season_number, data)
        elif data.get("_404"):
            tk.Label(tab_frame, text="Esta temporada aún no tiene datos en TMDb.",
                     bg=C["bg"], fg=C["muted"], font=(FONT_FAMILY, 10)).pack(pady=16)
        else:
            tk.Label(tab_frame, text="Error al conectar con TMDb. Comprueba tu conexión.",
                     bg=C["bg"], fg=C["muted"], font=(FONT_FAMILY, 10)).pack(pady=16)

    _season_pool.submit(_worker)
```

### Pattern 2: Canvas + Frame Scrollable Episode List

**What:** Standard Tkinter scrollable container for custom rows.
**When to use:** When rows need mixed widgets (checkbox + label + date) in a dark-themed list.

```python
# Source: ARCHITECTURE.md §Question 6; existing usage at Videoclub lines 3492–3501

canvas    = tk.Canvas(parent, bg=C["bg"], highlightthickness=0)
scrollbar = tk.Scrollbar(parent, orient="vertical", command=canvas.yview)
inner     = tk.Frame(canvas, bg=C["bg"])

inner.bind("<Configure>",
           lambda e: canvas.configure(scrollregion=canvas.bbox("all")))
canvas.create_window((0, 0), window=inner, anchor="nw")
canvas.configure(yscrollcommand=scrollbar.set)

canvas.pack(side="left", fill="both", expand=True)
scrollbar.pack(side="right", fill="y")

for w in (canvas, inner):
    w.bind("<MouseWheel>",
           lambda e: canvas.yview_scroll(-1 * (e.delta // 120), "units"))
```

### Pattern 3: ttk.Notebook Dark Theme Override

**What:** Forces dark bg/fg on notebook tabs (ttk defaults are light grey on Windows).
**When to use:** Must be applied BEFORE creating the Notebook widget.

```python
# Source: UI-SPEC.md §Color; ARCHITECTURE.md §Pitfalls

style = ttk.Style()
style.theme_use("default")
style.configure("Dark.TNotebook",     background=C["bg"])
style.configure("Dark.TNotebook.Tab", background=C["surface"],
                foreground=C["muted"], padding=[10, 4])
style.map("Dark.TNotebook.Tab",
          background=[("selected", C["bg"])],
          foreground=[("selected", C["text"])])

nb = ttk.Notebook(parent, style="Dark.TNotebook")
```

### Pattern 4: Modal Stack (grab_set chain)

**What:** Child Toplevel takes grab from parent; parent re-grabs on child destroy.
**When to use:** Any `tk.Toplevel` opened from an existing modal window.

```python
# Source: ARCHITECTURE.md §Question 1; PITFALLS.md §grab_set re-entry

# In DetailWindow._open_seasons():
win = SeasonViewWindow(self, item=self.item, user_id=self.user_id,
                       db=self.db, on_refresh=self.on_refresh,
                       app_root=self.app_root)
win.bind("<Destroy>",
         lambda e: self.grab_set() if self.winfo_exists() else None)

# In SeasonViewWindow.__init__:
self.grab_set()   # takes grab from DetailWindow automatically
self._destroyed = False
self.protocol("WM_DELETE_WINDOW", self._on_close)
self.bind("<Escape>", lambda e: self._on_close())

def _on_close(self):
    self._destroyed = True
    if callable(self.on_refresh):
        self.on_refresh()
    self.destroy()
```

### Pattern 5: BooleanVar Checkbox Toggle with Inline DB Write

**What:** Checkbox state → immediate SQLite write → counter update, all on main thread.
**When to use:** Episode watched toggle — no confirmation dialog, no background thread.

```python
# Source: CONTEXT.md D-09; REQUIREMENTS.md UI-06

var = tk.BooleanVar(value=is_watched)

def _toggle(ep_num=ep_num, bvar=var):
    new_state = bvar.get()
    self.db.set_episode_watched(
        self.user_id, self._item_id, season_number, ep_num, new_state
    )
    # Recount all BooleanVars for this season (fast — no extra DB query)
    watched_now = sum(1 for v in self._season_bvars[season_number] if v.get())
    total       = len(self._season_bvars[season_number])
    self._prog_labels[season_number].config(
        text=f"{watched_now} / {total} episodios vistos",
        fg=C["seen"] if watched_now == total else C["muted"]
    )

chk = tk.Checkbutton(row, variable=var, command=_toggle,
                     bg=C["surface"], fg=C["text"],
                     activebackground=C["surface"], selectcolor=C["bg"],
                     relief="flat", cursor="hand2")
```

### Anti-Patterns to Avoid

- **`range(1, num_seasons+1)` for tab building:** Misses Season 0, breaks on irregular numbering. Use `seasons[]` array from TMDb detail response (PITFALLS.md §C1).
- **Calling widget methods from `_worker` thread:** Undefined behavior on Windows; always use `self.after(0, ...)` (PITFALLS.md §C4).
- **Not guarding `after()` callbacks:** Window may be destroyed while fetch runs; always check `if self._destroyed or not self.winfo_exists(): return` (PITFALLS.md §m4).
- **Using episode list index as episode_number:** Gaps exist (episode_number=4 may be missing). Use `ep.get("episode_number")` from TMDb data (PITFALLS.md §M2).
- **Binding MouseWheel only to Canvas:** On Windows, inner Frame receives the event first; bind to both (PITFALLS.md §Canvas scroll).
- **Applying ttk.Style after Notebook creation:** Must call `style.configure(...)` before `ttk.Notebook(parent, style=...)` (ARCHITECTURE.md §Pitfalls).
- **`_on_close` calling `destroy()` without setting `_destroyed` first:** Race with in-flight `after()` callbacks; set flag first.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Episode progress persistence | Custom file/JSON storage | `db.set_episode_watched()` (Phase 1, line 396) | Already implemented, upsert, WAL-safe |
| Season episodes fetch | New HTTP function | `fetch_tmdb_season_episodes()` (Phase 1, line 1026) | Already implemented, lru_cache, 404/429 handling |
| Season list fetch | New HTTP function | `fetch_tmdb_detail(tmdb_id, "tv")` returns `_seasons` | Already returns seasons array at no extra cost |
| Thread pool for season fetches | `threading.Thread` ad-hoc | `_season_pool = ThreadPoolExecutor(max_workers=2)` | Matches existing `_poster_pool` pattern; lifecycle already managed |
| Scrollable widget with mixed content | Treeview, Listbox | Canvas + inner Frame (copy lines 3492–3501) | Only pattern that supports checkboxes + labels in dark theme |
| Progress counter initial value | Extra DB query | `db.get_episode_progress(user_id, item_id, season_number)` → count keys with `True` values | One call covers both watched state and count |
| Window centering | Manual geometry calculation | Copy `DetailWindow._center()` (line 1507) verbatim | Identical requirement |

**Key insight:** Phase 1 implemented all backend infrastructure. Phase 2 is exclusively UI wiring. Every helper needed already exists.

---

## Common Pitfalls

### Pitfall 1: `_season_pool` not declared at module level

**What goes wrong:** `SeasonViewWindow._load_season()` references `_season_pool` but it doesn't exist yet — `NameError` on first tab switch.
**Why it happens:** Developer adds the class but forgets to add the pool at line 161.
**How to avoid:** Add `_season_pool = ThreadPoolExecutor(max_workers=2, thread_name_prefix="season")` alongside `_poster_pool` at line 161, and `_season_pool.shutdown(wait=False)` alongside `_poster_pool.shutdown()` at the app exit block.
**Warning signs:** Test immediately with a series that has a valid `tmdb_id` — first tab switch crashes.

### Pitfall 2: Seasons initial fetch (D-01) using `fetch_tmdb_detail` synchronously in `__init__`

**What goes wrong:** Freezes the entire UI for 1–3 seconds while TMDb responds. Blocks DetailWindow and all parent widgets.
**Why it happens:** Easiest naive implementation.
**How to avoid:** Show spinner immediately, submit `fetch_tmdb_detail(tmdb_id, "tv")` to `_season_pool`, dispatch result via `self.app_root.after(0, _on_seasons_done)`.
**Warning signs:** Window appears but is unresponsive during open.

### Pitfall 3: Building notebook tabs with `range(1, num_seasons+1)`

**What goes wrong:** Season 0 ("Especiales") silently missing; HTTP 404 if TMDb season_number has gaps.
**Why it happens:** `num_seasons` integer is in the item dict and feels sufficient.
**How to avoid:** Iterate `seasons[]` array from `fetch_tmdb_detail` result's `_seasons` key. Already documented in D-05 and PITFALLS.md §C1.
**Warning signs:** Series like Breaking Bad (has Season 0) shows no Especiales tab.

### Pitfall 4: `<<NotebookTabChanged>>` not triggered for the first tab

**What goes wrong:** First tab stays as empty frame indefinitely; user must click away and back to trigger load.
**Why it happens:** The event does not fire during initial construction.
**How to avoid:** Call `self.after(0, self._on_tab_changed)` after binding the event (D-06).
**Warning signs:** Window opens, first tab is blank, no spinner.

### Pitfall 5: Bulk-mark includes future episodes

**What goes wrong:** Episodes not yet aired get marked watched, violating UI-08 semantics.
**Why it happens:** Loop iterates all episodes without date check.
**How to avoid:** In "Marcar toda" handler, skip episodes where `ep.get("air_date", "") > datetime.date.today().isoformat()` (D-10).
**Warning signs:** Series with upcoming episodes shows `Y / Y episodios vistos` when some are not yet aired.

### Pitfall 6: `winfo_exists()` check alone is insufficient race guard

**What goes wrong:** `winfo_exists()` can return True for a brief moment after `destroy()` is called; the window is being torn down but Tcl widget tree not yet fully cleaned.
**Why it happens:** Tkinter destroy is async at the Tcl level.
**How to avoid:** Use BOTH `self._destroyed` flag AND `winfo_exists()` check in every `after()` callback (UI-SPEC §Critical Implementation Guards point 3).
**Warning signs:** `TclError: invalid command name` in logs after fast close during load.

### Pitfall 7: `_season_cache` not populated before "Marcar toda" runs

**What goes wrong:** "Marcar toda" iterates `self._season_cache[season_number]["episodes"]` but cache is empty → `KeyError`.
**Why it happens:** Cache populated in `_on_done()` but button click can fire before data arrives.
**How to avoid:** The "Marcar toda" button is only created inside `_render_episode_list()`, which is called from `_on_done()` after data is received. Button cannot exist before `_season_cache` is populated. Document this dependency explicitly in the plan task.

---

## Code Examples

### SeasonViewWindow Constructor Skeleton

```python
# Source: CONTEXT.md D-12, D-13; ARCHITECTURE.md §Question 1; UI-SPEC.md §Component Inventory

# ══════════════════════════════════════════════════════════════
# 10.5. SEASON VIEW WINDOW
# ══════════════════════════════════════════════════════════════

class SeasonViewWindow(tk.Toplevel):
    def __init__(self, parent, item: dict, user_id: int,
                 db: "Database", on_refresh, app_root: tk.Tk):
        super().__init__(parent)
        self.item       = item
        self.user_id    = user_id
        self.db         = db
        self.on_refresh = on_refresh
        self.app_root   = app_root
        self._tmdb_id   = str(item.get("tmdb_id") or "")
        self._item_id   = item["id"]
        self._destroyed = False

        self._loaded_seasons: set  = set()
        self._season_frames: dict  = {}   # season_number → tk.Frame
        self._season_cache:  dict  = {}   # season_number → TMDb data dict
        self._season_bvars:  dict  = {}   # season_number → list[BooleanVar]
        self._prog_labels:   dict  = {}   # season_number → tk.Label

        self.title(f"Temporadas — {item['title'][:50]}")
        self.configure(bg=C["bg"])
        self.resizable(True, True)
        self.minsize(560, 400)
        self.geometry("700x520")
        self.grab_set()
        self.protocol("WM_DELETE_WINDOW", self._on_close)
        self.bind("<Escape>", lambda e: self._on_close())
        self._center()
        self._show_seasons_spinner()

    def _center(self):
        self.update_idletasks()
        x = (self.winfo_screenwidth()  - self.winfo_width())  // 2
        y = max(0, (self.winfo_screenheight() - self.winfo_height()) // 2)
        self.geometry(f"+{x}+{y}")

    def _on_close(self):
        self._destroyed = True
        if callable(self.on_refresh):
            self.on_refresh()
        self.destroy()
```

### Seasons Initial Async Load (D-01)

```python
# Source: CONTEXT.md D-01, D-02; ARCHITECTURE.md §Question 2

def _show_seasons_spinner(self):
    self._spinner_frame = tk.Frame(self, bg=C["bg"])
    self._spinner_frame.pack(fill="both", expand=True)
    tk.Label(self._spinner_frame, text="Cargando temporadas...",
             bg=C["bg"], fg=C["muted"],
             font=(FONT_FAMILY, 10)).pack(expand=True)

    def _worker():
        data = fetch_tmdb_detail(self._tmdb_id, "tv")
        self.app_root.after(0, lambda d=data: _on_seasons_done(d))

    def _on_seasons_done(data):
        if self._destroyed or not self.winfo_exists():
            return
        self._spinner_frame.destroy()
        seasons = data.get("_seasons", [])
        if not seasons:
            tk.Label(self, text="No se pudo obtener las temporadas. Comprueba tu conexión.",
                     bg=C["bg"], fg=C["muted"],
                     font=(FONT_FAMILY, 10)).pack(expand=True)
            tk.Button(self, text="Cerrar", command=self.destroy,
                      bg=C["surface"], fg=C["muted"],
                      font=(FONT_FAMILY, 9), relief="flat").pack(pady=8)
            return
        self._build_notebook(seasons)

    _season_pool.submit(_worker)
```

### Episode Row with Future Detection

```python
# Source: CONTEXT.md D-08; UI-SPEC.md §Episode Row; REQUIREMENTS.md UI-08

import datetime

def _make_episode_row(self, parent, ep, is_watched, season_number):
    ep_num  = ep.get("episode_number", 0)
    ep_name = ep.get("name", f"Episodio {ep_num}")
    ep_date = ep.get("air_date", "")
    today   = datetime.date.today().isoformat()
    is_future = bool(ep_date) and ep_date > today

    row = tk.Frame(parent, bg=C["surface"])
    row.pack(fill="x", padx=8, pady=4)
    row.bind("<Enter>", lambda e: row.config(bg=C["hover"]))
    row.bind("<Leave>", lambda e: row.config(bg=C["surface"]))

    var = tk.BooleanVar(value=is_watched)
    self._season_bvars[season_number].append(var)

    def _toggle(n=ep_num, v=var):
        self.db.set_episode_watched(
            self.user_id, self._item_id, season_number, n, v.get()
        )
        watched_now = sum(1 for bv in self._season_bvars[season_number] if bv.get())
        total       = len(self._season_bvars[season_number])
        self._prog_labels[season_number].config(
            text=f"{watched_now} / {total} episodios vistos",
            fg=C["seen"] if watched_now == total else C["muted"]
        )

    chk = tk.Checkbutton(
        row, variable=var, command=_toggle,
        bg=C["surface"], fg=C["text"],
        activebackground=C["surface"], selectcolor=C["bg"],
        relief="flat", cursor="hand2"
    )
    if is_future:
        chk.config(state="disabled", fg=C["muted"])
    chk.pack(side="left", padx=(6, 4))

    tk.Label(row, text=f"{ep_num:02d}. {ep_name}",
             bg=C["surface"],
             fg=C["muted"] if (is_future or not is_watched) else C["text"],
             font=(FONT_FAMILY, 9), anchor="w"
             ).pack(side="left", fill="x", expand=True)

    if ep_date:
        tk.Label(row, text=ep_date[:7],
                 bg=C["surface"], fg=C["border"],
                 font=(FONT_FAMILY, 8)).pack(side="right", padx=8)
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `range(1, num_seasons+1)` for season tabs | Iterate `detail["seasons"]` array | Phase 1 decision | Captures Season 0, handles irregular numbering |
| `tmdb_id` re-searched by title each open | Persisted `tmdb_id` in `items` table (v7) | Phase 1 DB migration | SeasonViewWindow always has stable series ID |
| Single `_poster_pool` for all async work | Dedicated `_season_pool` for season fetches | Phase 2 (new) | No contention with poster loading |

---

## Assumptions Log

All critical claims verified directly in the codebase. No assumptions required.

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| — | All Phase 1 DB methods and API functions are present in `Videoclub` | verified line 386–426, 981–1038 | — |

**All claims in this research are VERIFIED or CITED — no user confirmation needed.**

---

## Open Questions

None. All decisions locked in CONTEXT.md. All integration points verified in codebase.

---

## Environment Availability

Step 2.6 SKIPPED — no external dependencies. Phase 2 is pure Python/Tkinter UI code in the existing single file. All runtime dependencies (SQLite, TMDb API key, ThreadPoolExecutor) were established in Phase 1.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Manual smoke tests (no automated test framework detected in project) |
| Config file | none |
| Quick run command | `python Videoclub` → open DetailWindow for a series with tmdb_id → click "Ver temporadas" |
| Full suite command | Manual checklist against 6 success criteria from ROADMAP.md |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| UI-01 | "Ver temporadas" button visible for series+tmdb_id, hidden otherwise | smoke | Manual | N/A |
| UI-02 | SeasonViewWindow grabs focus; DetailWindow re-grabs on close | smoke | Manual | N/A |
| UI-03 | Tab switch triggers lazy load; already-loaded tab does not re-fetch | smoke | Manual | N/A |
| UI-04 | Counter shows correct X/Y; updates on toggle; turns green at 100% | smoke | Manual | N/A |
| UI-05 | Episode list scrolls; dark theme consistent; number/title/date shown | smoke | Manual | N/A |
| UI-06 | Checkbox toggle writes to SQLite immediately (verify in DB browser) | smoke | Manual | N/A |
| UI-07 | "Marcar toda" marks past+present episodes; skips future | smoke | Manual | N/A |
| UI-08 | Future episodes grey + disabled checkbox | smoke | Manual | N/A |
| UI-09 | Spinner shown during load; error label on network failure | smoke | Manual | N/A |
| UI-10 | Closing window triggers on_refresh; grid updates without F5 | smoke | Manual | N/A |

### Wave 0 Gaps

None — no automated test infrastructure is required for this phase. All validation is manual smoke testing against the 6 ROADMAP success criteria.

---

## Security Domain

No new attack surface in this phase. SeasonViewWindow:
- Reads from SQLite via existing parameterized queries (`db.get_episode_progress`)
- Writes to SQLite via existing parameterized upsert (`db.set_episode_watched`)
- Makes outbound HTTPS calls to TMDb (existing pattern, existing API key handling)
- No user-supplied input rendered as SQL or HTML
- No new file I/O, no new network endpoints, no auth changes

ASVS V5 (Input Validation): episode data from TMDb is displayed via `.config(text=...)` — no injection risk in Tkinter label widgets.

---

## Sources

### Primary (HIGH confidence — verified in codebase)

- `D:\VideoClub\Videoclub` lines 158–163 — SCHEMA_VERSION=7, `_poster_pool`, `_poster_lock`, `_stop_event` (pool pattern to follow)
- `D:\VideoClub\Videoclub` lines 386–426 — `get_episode_progress`, `set_episode_watched`, `get_season_progress` (Phase 1 DB methods confirmed present)
- `D:\VideoClub\Videoclub` lines 981–1038 — `_fetch_tmdb_season_cached`, `fetch_tmdb_season_episodes` (Phase 1 API confirmed present)
- `D:\VideoClub\Videoclub` lines 1488–1747 — `DetailWindow` class (template for SeasonViewWindow; insertion point confirmed at line 1749)
- `.planning/phases/02-seasonviewwindow/02-UI-SPEC.md` — complete widget inventory, color roles, spacing tokens, interaction protocols
- `.planning/research/ARCHITECTURE.md` — threading pattern, Canvas+Frame scroll, grab_set chain, data flow
- `.planning/research/PITFALLS.md` — C1 (seasons array), C4 (threading), m4 (destroyed Toplevel), M2 (episode numbering), M3 (state desync)
- `.planning/phases/02-seasonviewwindow/02-CONTEXT.md` — all 14 locked decisions

### Secondary (MEDIUM confidence)

- ROADMAP.md + REQUIREMENTS.md — 6 success criteria, UI-01 through UI-10 mapped to components

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — verified in codebase
- Architecture: HIGH — all integration points line-verified
- Pitfalls: HIGH — sourced from project-specific PITFALLS.md (code-inspected during Phase 1 research)

**Research date:** 2026-04-17
**Valid until:** Phase 2 execution (no external dependencies to go stale)
