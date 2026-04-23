# Phase 2: SeasonViewWindow — Pattern Map

**Mapped:** 2026-04-17
**Files analyzed:** 1 file modified (single-file app: `Videoclub`)
**Analogs found:** 5 / 5 pattern categories

---

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `Videoclub` — `class SeasonViewWindow` (insert at line 1749) | component/modal | request-response + async | `DetailWindow` (line 1488) | exact |
| `Videoclub` — `_season_pool` declaration (insert at line 162) | config | async/batch | `_poster_pool` (line 161) | exact |
| `Videoclub` — `DetailWindow._open_seasons()` new method | controller method | request-response | `DetailWindow._set_status()` + `DetailWindow._load_poster()` pattern | role-match |
| `Videoclub` — `DetailWindow._build()` "Ver temporadas" button | component fragment | request-response | existing status buttons in `_build()` (lines 1553–1561) | exact |
| `Videoclub` — `VideoclubApp._on_close()` pool shutdown (line 3779) | config | lifecycle | `_poster_pool.shutdown(wait=False)` (line 3779) | exact |

---

## Pattern Assignments

### `SeasonViewWindow.__init__` — Modal Toplevel constructor

**Analog:** `DetailWindow.__init__` (lines 1488–1504) + `RefreshMetaDialog.__init__` (lines 2228–2242)

**Imports pattern** — already at top of file; no new imports needed. `datetime` is stdlib and already available.

**Constructor pattern** (analog: lines 1488–1504 for `DetailWindow`, lines 2228–2242 for `RefreshMetaDialog`):
```python
# DetailWindow.__init__ — lines 1488–1504
class DetailWindow(tk.Toplevel):
    def __init__(self, parent, item: dict, user_id: int, db: Database,
                 on_refresh, app_root: tk.Tk):
        super().__init__(parent)
        self.title(item["title"][:60])
        self.configure(bg=C["bg"])
        self.resizable(True, False)
        self.grab_set()
        self.item       = item
        self.user_id    = user_id
        self.db         = db
        self.on_refresh = on_refresh
        self.app_root   = app_root
        self._photo     = None
        self._build()
        self._center()
        self._load_poster()
        self.bind("<Escape>", lambda e: self.destroy())
```

```python
# RefreshMetaDialog.__init__ — lines 2228–2242
# Shows protocol("WM_DELETE_WINDOW") + _cancel pattern:
self.protocol("WM_DELETE_WINDOW", self._cancel)
self.bind("<Escape>", lambda _: self._cancel())
```

**SeasonViewWindow must follow this exact shape** with:
- `self._destroyed = False` flag (not present in DetailWindow — required per D-06/Pitfall 6)
- `self.protocol("WM_DELETE_WINDOW", self._on_close)` (from RefreshMetaDialog pattern)
- `self.minsize(560, 400)` + `self.geometry("700x520")` (per UI-SPEC)
- Instance dicts: `_loaded_seasons`, `_season_frames`, `_season_cache`, `_season_bvars`, `_prog_labels`

---

### `SeasonViewWindow._center()` — Screen centering

**Analog:** `DetailWindow._center()` (lines 1507–1511) — copy verbatim

```python
# DetailWindow._center — lines 1507–1511
def _center(self):
    self.update_idletasks()
    x = (self.winfo_screenwidth()  - self.winfo_width())  // 2
    y = max(0, (self.winfo_screenheight() - self.winfo_height()) // 2)
    self.geometry(f"+{x}+{y}")
```

Copy verbatim. `max(0, ...)` guard is important — prevents negative y on small screens.

---

### `SeasonViewWindow._on_close()` — Close protocol

**Analog:** `RefreshMetaDialog._cancel()` (lines 2362–2364) for the stop-flag pattern; CONTEXT D-03/D-04 for on_refresh call.

```python
# RefreshMetaDialog._cancel — lines 2362–2364
def _cancel(self):
    self._stop.set()
    self.destroy()
```

SeasonViewWindow needs `_destroyed = True` + `on_refresh()` call before `destroy()`:
```python
# Pattern to implement (from RESEARCH Pattern 4):
def _on_close(self):
    self._destroyed = True
    if callable(self.on_refresh):
        self.on_refresh()
    self.destroy()
```

---

### `_season_pool` global declaration (line 162)

**Analog:** `_poster_pool` (line 161) — insert immediately below it

```python
# Existing — line 161
_poster_pool  = ThreadPoolExecutor(max_workers=4, thread_name_prefix="poster")
_poster_lock  = threading.Lock()
_stop_event   = threading.Event()

# Insert after line 161:
# _season_pool  = ThreadPoolExecutor(max_workers=2, thread_name_prefix="season")
```

**Shutdown analog** (line 3779):
```python
# Existing — line 3779 in VideoclubApp._on_close()
_poster_pool.shutdown(wait=False)
# Add immediately below:
# _season_pool.shutdown(wait=False)
```

---

### Async fetch → `after(0, ...)` → `winfo_exists()` guard pattern

**Analog:** `get_poster_async` inner `_worker` + `_finalize` (lines 1238–1277)

```python
# get_poster_async — lines 1238–1277 (core pattern)
def _worker():
    if _stop_event.is_set():
        return
    # ... I/O work ...
    def _finalize():
        if _stop_event.is_set():
            return
        # ... widget update ...
    if not _stop_event.is_set():
        try:
            app_root.after(0, _finalize)
        except Exception:
            pass

_poster_pool.submit(_worker)
```

SeasonViewWindow uses the same structure but with `_season_pool` and `self._destroyed` guard instead of `_stop_event`:

```python
# Pattern from RESEARCH §Pattern 1 (verified against codebase threading shape):
def _worker():
    data = fetch_tmdb_detail(self._tmdb_id, "tv")
    self.app_root.after(0, lambda d=data: _on_seasons_done(d))

def _on_seasons_done(data):
    if self._destroyed or not self.winfo_exists():
        return
    # ... widget build ...

_season_pool.submit(_worker)
```

Key difference from poster pattern: `_stop_event` is app-global; `_destroyed` is instance-local. Both are needed for season fetches (window may close before fetch completes).

---

### Canvas + Frame scrollable list

**Analog:** Lines 3490–3501 (sidebar scroll panel in `VideoclubApp`)

```python
# Lines 3490–3501 — exact template
outer = tk.Frame(p, bg=C["surface"])
outer.pack(fill="both", expand=True)
cv = tk.Canvas(outer, bg=C["surface"], highlightthickness=0, width=296)
sb = tk.Scrollbar(outer, orient="vertical", command=cv.yview)
cv.configure(yscrollcommand=sb.set)
sb.pack(side="right", fill="y")
cv.pack(side="left", fill="both", expand=True)
f = tk.Frame(cv, bg=C["surface"])
cv.create_window((0, 0), window=f, anchor="nw")
f.bind("<Configure>", lambda e: cv.configure(scrollregion=cv.bbox("all")))
cv.bind("<MouseWheel>",
        lambda e: cv.yview_scroll(int(-1 * e.delta / 120), "units"))
```

SeasonViewWindow delta: bind MouseWheel to BOTH `cv` and inner `f` (D-07 — existing analog only binds canvas). Full pattern per RESEARCH §Pattern 2:
```python
for w in (canvas, inner):
    w.bind("<MouseWheel>",
           lambda e: canvas.yview_scroll(-1 * (e.delta // 120), "units"))
```

---

### ttk.Notebook dark theme

**Analog:** `RefreshMetaDialog._build()` ttk.Style override (lines 2278–2286) — closest style-configure pattern in codebase

```python
# RefreshMetaDialog — lines 2278–2283 (ttk.Style configure pattern)
style = ttk.Style()
style.theme_use("default")
style.configure("RM.Horizontal.TProgressbar",
                 troughcolor=C["border"], background=C["seen"],
                 bordercolor=C["border"],
                 lightcolor=C["seen"], darkcolor=C["seen"])
```

No `ttk.Notebook` exists yet in the codebase. SeasonViewWindow is the first. Use RESEARCH §Pattern 3 which is verified architecture-doc pattern:
```python
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

CRITICAL: `style.configure(...)` must be called BEFORE `ttk.Notebook(...)` is instantiated.

---

### Episode row button pattern (status buttons in DetailWindow)

**Analog:** Status buttons in `DetailWindow._build()` (lines 1553–1561)

```python
# DetailWindow._build() — lines 1553–1561
for lbl_txt, st, color in [
    ("⏳ Ver luego",    STATUS_PENDING, C["pending"]),
    ("✓ Marcar visto", STATUS_WATCHED, C["seen"]),
    ("✗ Sin estado",   STATUS_NONE,    C["border"]),
]:
    tk.Button(left, text=lbl_txt, bg=color, fg="white",
              font=(FONT_FAMILY, 9), relief="flat",
              command=lambda s=st: self._set_status(s),
              cursor="hand2").pack(fill="x", pady=1)
```

Episode row is different (Checkbutton not Button) — no direct analog for Checkbutton + BooleanVar. Use RESEARCH §Pattern 5 directly. The `selectcolor=C["bg"]` and `activebackground=C["surface"]` tokens follow from the app's dark theme `C` dict established throughout the file.

---

### `DetailWindow._open_seasons()` + "Ver temporadas" button

**Analog:** `DetailWindow._build()` "🗑 Eliminar" button (lines 1570–1572) for button placement; `DetailWindow._load_poster()` call pattern for opening a child window

```python
# DetailWindow._build() — lines 1570–1572
tk.Button(left, text="🗑 Eliminar", bg="#3d0000", fg="#ff6b6b",
          font=(FONT_FAMILY, 9), relief="flat",
          command=self._delete, cursor="hand2").pack(fill="x", pady=(10, 0))
```

"Ver temporadas" button goes after the Eliminar button, conditional on `itype == "series" and tmdb_id`:
```python
# Pattern: conditional button after Eliminar
if itype == "series" and self.item.get("tmdb_id"):
    tk.Button(left, text="📺 Ver temporadas", bg=C["series_badge"], fg="white",
              font=(FONT_FAMILY, 9), relief="flat",
              command=lambda: self._open_seasons(
                  self.item["tmdb_id"], self.item.get("num_seasons", 0)
              ),
              cursor="hand2").pack(fill="x", pady=(4, 0))
```

`_open_seasons()` method analog — `SmartSearchDialog` instantiation pattern in `VideoclubApp` (any place a Toplevel is opened and `<Destroy>` is bound for grab restore):
```python
# Pattern from RESEARCH §Pattern 4:
def _open_seasons(self, tmdb_id, num_seasons):
    win = SeasonViewWindow(self, item=self.item, user_id=self.user_id,
                           db=self.db, on_refresh=self.on_refresh,
                           app_root=self.app_root)
    win.bind("<Destroy>",
             lambda e: self.grab_set() if self.winfo_exists() else None)
```

---

### Progress counter label — in-place update

**Analog:** `RefreshMetaDialog._update_ui()` (lines 2345–2351) — `.config(text=...)` in-place update pattern

```python
# RefreshMetaDialog._update_ui — lines 2345–2351
def _update_ui(self, title: str, idx: int):
    if not self.winfo_exists():
        return
    short = title[:45] + "…" if len(title) > 45 else title
    self._lbl.config(text=f"🔍 {short}")
    self._bar["value"] = idx
    self._count_lbl.config(text=f"{idx} / {self._total}")
```

Progress counter follows this `.config(text=...)` pattern. SeasonViewWindow variant:
```python
# Per D-11: recount BooleanVars, update label fg based on completion
watched_now = sum(1 for v in self._season_bvars[season_number] if v.get())
total       = len(self._season_bvars[season_number])
self._prog_labels[season_number].config(
    text=f"{watched_now} / {total} episodios vistos",
    fg=C["seen"] if watched_now == total else C["muted"]
)
```

---

## Shared Patterns

### `winfo_exists()` + `_destroyed` double guard
**Source:** `get_poster_async._finalize` (line 1271 uses `_stop_event`); `RefreshMetaDialog._done` (line 2353–2354 uses `winfo_exists()` alone)
**Apply to:** Every `after(0, ...)` callback in `SeasonViewWindow`
```python
# Minimum required guard in every after() callback:
if self._destroyed or not self.winfo_exists():
    return
```
Note: existing dialogs use only `winfo_exists()`. SeasonViewWindow MUST add `_destroyed` flag in addition (PITFALLS §6 — Tcl destroy is async).

### Dark theme color tokens
**Source:** `C` dict — used throughout entire file
**Apply to:** All widgets in SeasonViewWindow
- `bg=C["bg"]` — window background, canvas, frames
- `bg=C["surface"]` — row cards
- `fg=C["text"]` — normal text
- `fg=C["muted"]` — secondary text, disabled/future episodes
- `fg=C["seen"]` — completed progress (green)
- `bg=C["series_badge"]` — "Ver temporadas" button color
- `selectcolor=C["bg"]` — Checkbutton checkbox bg (dark theme inversion)

### `after(0, lambda d=data: callback(d))` default-capture idiom
**Source:** `RefreshMetaDialog._worker` (line 2320): `self.after(0, lambda t=title, i=idx: self._update_ui(t, i))`
**Apply to:** All lambda closures inside thread workers dispatching to main thread
```python
# Correct: d=data captures current value
self.app_root.after(0, lambda d=data: _on_seasons_done(d))
# Wrong: late-binding bug
self.app_root.after(0, lambda: _on_seasons_done(data))
```

### Font pattern
**Source:** Every Label/Button throughout file
**Apply to:** All widgets in SeasonViewWindow
```python
font=(FONT_FAMILY, 10)         # body
font=(FONT_FAMILY, 9)          # secondary / buttons
font=(FONT_FAMILY, 8)          # dates
font=(FONT_FAMILY, 12, "bold") # section headers
```

---

## No Analog Found

| File fragment | Role | Data Flow | Reason |
|---------------|------|-----------|--------|
| `SeasonViewWindow._build_notebook()` — ttk.Notebook with dark style | component | event-driven | No `ttk.Notebook` exists in codebase yet; use RESEARCH §Pattern 3 |
| `SeasonViewWindow._on_tab_changed()` — lazy-load on tab switch | event handler | event-driven | No `<<NotebookTabChanged>>` binding anywhere in codebase |
| Episode row `Checkbutton` + `BooleanVar` | component | request-response | No Checkbutton widgets exist in codebase (grep confirmed no matches) |

---

## DB Method Reference (Phase 1, verified present)

| Method | Location | Signature |
|--------|----------|-----------|
| `db.get_episode_progress` | line 386 | `(user_id, item_id, season_number) → dict[int, bool]` |
| `db.set_episode_watched` | line 396 | `(user_id, item_id, season_number, episode_number, watched) → bool` |
| `db.get_season_progress` | line 414 | `(user_id, item_id, season_number) → tuple[int, int]` |
| `fetch_tmdb_detail` | line 934 | `(tmdb_id, "tv") → dict` — includes `_seasons` list |
| `fetch_tmdb_season_episodes` | line 1026 | `(tmdb_id, season_number) → dict` — lru_cached |

---

## Metadata

**Analog search scope:** Entire `D:/VideoClub/Videoclub` file (~3780 lines)
**Classes scanned:** `DetailWindow`, `SmartSearchDialog`, `RefreshMetaDialog`, `UserPickerWindow`, `SetupWizard`, `VideoclubApp`
**Pattern extraction date:** 2026-04-17
