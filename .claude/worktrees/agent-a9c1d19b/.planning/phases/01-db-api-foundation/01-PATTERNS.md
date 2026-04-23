# Phase 1: DB + API Foundation — Pattern Map

**Mapped:** 2026-04-14
**Files analyzed:** 1 (all changes go into `d:/VideoClub/Videoclub`)
**Analogs found:** 5 / 5 (all within same file — self-referential patterns)

---

## File Classification

| Edit Target | Role | Data Flow | Closest Analog (in same file) | Match Quality |
|-------------|------|-----------|-------------------------------|---------------|
| `SCHEMA_VERSION` constant (line 158) | config | — | Same constant, just bump value | exact |
| `Database._migrate()` v7 block (lines 269–316) | migration | CRUD | Existing `if v < 6:` block (lines 298–313) | exact |
| `Database.add_item()` signature + INSERT (lines 324–345) | model/CRUD | CRUD | Same function — extend in place | exact |
| `fetch_tmdb_detail()` return dict (lines 852–893) | service | request-response | Same function — extend return dict | exact |
| `fetch_tmdb_season_episodes()` — new function | service | request-response | `fetch_tmdb_detail()` (lines 852–893) | exact role+flow |
| `SmartSearchDialog._add_result()` call site (lines 1968–1975) | controller | request-response | Same lines — extend `add_item` call | exact |
| `SmartSearchDialog._add_manual/_worker` (lines 2004–2008) | controller | request-response | `_add_result` enrichment block (lines 1940–1945) | role-match |
| `SmartSearchDialog._finish_manual()` call site (lines 2036–2043) | controller | request-response | `_add_result` call site pattern | exact |
| `BatchImportDialog._worker` call site (lines 2347–2354) | controller | batch | Same lines — no functional change needed | exact |

---

## Pattern Assignments

### 1. `SCHEMA_VERSION` constant (line 158)

**Analog:** Same line.

**Current code** (line 158):
```python
SCHEMA_VERSION = 6
```

**Change to:**
```python
SCHEMA_VERSION = 7
```

**Critical rule:** This MUST be the very first edit before any migration code is written. If left at 6, the `if v < SCHEMA_VERSION:` gate at line 315 re-writes schema version to 6 on every launch, causing the `if v < 7:` block to run repeatedly.

---

### 2. `Database._migrate()` — ALTER TABLE loop (lines 272–287)

**Analog:** Existing column-addition loop (lines 272–287).

**Current code** (lines 272–287):
```python
for col, tbl, defn in [
    ("added_at",    "items",   "TEXT DEFAULT (datetime('now'))"),
    ("status",      "watched", "TEXT NOT NULL DEFAULT 'watched'"),
    ("user_rating", "watched", "INTEGER"),
    ("notes",       "watched", "TEXT"),
    # v5 — nuevos metadatos
    ("director",    "items",   "TEXT"),
    ("actors",      "items",   "TEXT"),
    ("studio",      "items",   "TEXT"),
    ("streaming",   "items",   "TEXT"),
]:
    try:
        self.conn.execute(f"ALTER TABLE {tbl} ADD COLUMN {col} {defn}")
        self.conn.commit()
    except Exception:
        pass
```

**Change:** Add two new entries to the list AND replace `except Exception: pass` with a log call:
```python
for col, tbl, defn in [
    ("added_at",    "items",   "TEXT DEFAULT (datetime('now'))"),
    ("status",      "watched", "TEXT NOT NULL DEFAULT 'watched'"),
    ("user_rating", "watched", "INTEGER"),
    ("notes",       "watched", "TEXT"),
    # v5 — nuevos metadatos
    ("director",    "items",   "TEXT"),
    ("actors",      "items",   "TEXT"),
    ("studio",      "items",   "TEXT"),
    ("streaming",   "items",   "TEXT"),
    # v7 — episode tracking
    ("tmdb_id",     "items",   "TEXT"),
    ("num_seasons", "items",   "INTEGER"),
]:
    try:
        self.conn.execute(f"ALTER TABLE {tbl} ADD COLUMN {col} {defn}")
        self.conn.commit()
        log.info(f"_migrate: added column {col} to {tbl}")
    except Exception as e:
        log.debug(f"_migrate: {col} on {tbl} already exists or error: {e}")
```

**Rules:**
- Use nullable columns only (`TEXT` and `INTEGER` without `NOT NULL`) — SQLite rejects `ALTER TABLE ... ADD COLUMN x TEXT NOT NULL` on a table that already has rows.
- The `except` catching `OperationalError` for "column already exists" IS the idempotency mechanism — do not remove it, just log it.

---

### 3. `Database._migrate()` — new `if v < 7:` block

**Analog:** Existing `if v < 6:` block (lines 298–313).

**Current code to mirror** (lines 298–313):
```python
if v < 6:
    # Crear user_items y poblarla con el estado actual (todos los
    # ítems pertenecen a todos los usuarios ya existentes).
    self.conn.executescript("""
        CREATE TABLE IF NOT EXISTS user_items (
            user_id  INTEGER NOT NULL,
            title    TEXT    NOT NULL,
            added_at TEXT    DEFAULT (datetime('now')),
            PRIMARY KEY (user_id, title)
        );
        INSERT OR IGNORE INTO user_items(user_id, title, added_at)
            SELECT u.id, i.title, i.added_at
            FROM users u, items i;
        CREATE INDEX IF NOT EXISTS idx_uitem_user ON user_items(user_id);
    """)
    self.conn.commit()
```

**New block to insert after line 313, before line 315:**
```python
if v < 7:
    try:
        self.conn.execute(
            "CREATE INDEX IF NOT EXISTS idx_items_tmdb_id ON items(tmdb_id)"
        )
        self.conn.commit()
        log.info("_migrate v7: created idx_items_tmdb_id")
    except Exception as e:
        log.error(f"_migrate v7 index: {e}")

    try:
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
        log.info("_migrate v7: created episode_progress table")
    except Exception as e:
        log.error(f"_migrate v7 episode_progress: {e}")
```

**Key differences from `v < 6` pattern:**
- Use explicit `try/except` with `log.error(...)` (not `log.debug`) — table creation failures are not expected/benign.
- Use `executescript` for multi-statement DDL (same as `v < 6` pattern).
- `episode_progress.item_id` is an FK to `items.id` (integer PK), NOT a `tmdb_series_id TEXT` field — this is a locked architecture decision.

---

### 4. `Database.add_item()` — signature and INSERT (lines 324–345)

**Analog:** Same function — extend in place.

**Current signature and INSERT** (lines 324–334):
```python
def add_item(self, title, itype, rating, genres, year, plot, poster_url,
            director="", actors="", studio="", streaming="",
            user_id: int = 0) -> bool:
    try:
        self.conn.execute(
            "INSERT OR IGNORE INTO items"
            "(title,type,rating,genres,year,plot,poster_url,director,actors,studio,streaming)"
            " VALUES(?,?,?,?,?,?,?,?,?,?,?)",
            (title, itype, rating, genres, year, plot, poster_url,
             director or "", actors or "", studio or "", streaming or "")
        )
```

**New signature and INSERT:**
```python
def add_item(self, title, itype, rating, genres, year, plot, poster_url,
            director="", actors="", studio="", streaming="",
            tmdb_id: str = "", num_seasons: int = 0,
            user_id: int = 0) -> bool:
    try:
        self.conn.execute(
            "INSERT OR IGNORE INTO items"
            "(title,type,rating,genres,year,plot,poster_url,"
            "director,actors,studio,streaming,tmdb_id,num_seasons)"
            " VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?)",
            (title, itype, rating, genres, year, plot, poster_url,
             director or "", actors or "", studio or "", streaming or "",
             tmdb_id or "", num_seasons or 0)
        )
```

**Rules:**
- `tmdb_id` and `num_seasons` go BEFORE `user_id` — they are item-level fields.
- Both have safe defaults (`""` and `0`) so all existing callers (BatchImportDialog) are unaffected without changes.
- The rest of the function body (lines 335–345) is unchanged.

---

### 5. `fetch_tmdb_detail()` — extend return dict (lines 852–893)

**Analog:** Same function — extend return dict in place.

**Current return** (lines 889–890):
```python
        return {"Director": director, "Actors": actors,
                "Studio": studio, "Streaming": streaming}
```

**New return** (replace lines 889–890):
```python
        result = {"Director": director, "Actors": actors,
                  "Studio": studio, "Streaming": streaming}
        if media_type != "movie":
            result["_num_seasons"] = d.get("number_of_seasons", 0)
            result["_seasons"]     = d.get("seasons", [])
        return result
```

**Context:** `d` is already the full `/tv/{id}` response dict fetched at line 864. The `seasons` array and `number_of_seasons` field are present in TMDb TV detail responses at no extra API cost. The `except` block at line 892 (`return {}`) is unchanged — errors still return empty dict.

---

### 6. `fetch_tmdb_season_episodes()` — new function

**Analog:** `fetch_tmdb_detail()` (lines 852–893) — same role (module-level fetch function), same data flow (request-response with lru_cache).

**Insertion point:** After line 893 (after `fetch_tmdb_detail`), before line 896 (`fetch_duckduckgo`).

**Import pattern** (already imported at line 58):
```python
from functools import lru_cache
```

**Core pattern** (copy structure from `fetch_tmdb_detail` lines 852–893):
```python
@lru_cache(maxsize=50)
def _fetch_tmdb_season_cached(tmdb_id: str, season_number: int) -> dict:
    """Internal cached fetcher — only successful responses are cached.
    Use fetch_tmdb_season_episodes() as the public API."""
    key = CFG.get("tmdb_api_key", "")
    if not key or not tmdb_id:
        return {}
    try:
        url = (f"https://api.themoviedb.org/3/tv/{tmdb_id}"
               f"/season/{season_number}"
               f"?api_key={key}&language=es-ES")
        resp = requests.get(url, timeout=10)
        if resp.status_code == 200:
            return resp.json()
        elif resp.status_code == 404:
            log.warning(f"fetch_tmdb_season_episodes: 404 tmdb_id={tmdb_id} season={season_number} — no data in TMDb yet")
            return {"_404": True}   # sentinel so 404 is cached (permanent non-retryable)
        elif resp.status_code == 429:
            log.warning(f"fetch_tmdb_season_episodes: 429 rate limit tmdb_id={tmdb_id} season={season_number}")
            import time; time.sleep(1)
            resp2 = requests.get(url, timeout=10)
            if resp2.status_code == 200:
                return resp2.json()
            log.error(f"fetch_tmdb_season_episodes: retry after 429 also failed ({resp2.status_code})")
            return {}   # transient — NOT cached (public wrapper clears cache)
        else:
            log.error(f"fetch_tmdb_season_episodes: HTTP {resp.status_code} tmdb_id={tmdb_id} season={season_number}")
            return {}
    except Exception as e:
        log.error(f"fetch_tmdb_season_episodes: network error: {e}")
    return {}


def fetch_tmdb_season_episodes(tmdb_id: str, season_number: int) -> dict:
    """Returns TMDb season object with 'episodes' list.
    Cached per session for successful responses and permanent 404s.
    Transient network/5xx errors are NOT cached — next call retries.
    Returns {} on any error; caller checks data.get('episodes', [])."""
    result = _fetch_tmdb_season_cached(tmdb_id, season_number)
    if not result:
        # Transient error — clear cache so next call retries
        _fetch_tmdb_season_cached.cache_clear()
        return {}
    if result.get("_404"):
        return {}   # permanent 404 — already cached, return empty to caller
    return result
```

**Rules:**
- `@lru_cache` requires hashable args — `str` and `int` are both hashable. Safe.
- Check `resp.status_code` BEFORE calling `.json()` — TMDb returns JSON bodies on 404/429.
- The wrapper pattern avoids caching transient empty-dict results (PITFALL 5 from RESEARCH.md).
- `time` is already in stdlib; the `import time` inline avoids a top-level import for a single-use sleep.

---

### 7. `SmartSearchDialog._add_result()` call site (lines 1968–1975)

**Analog:** Same lines — extend `add_item()` call to pass `tmdb_id` and `num_seasons`.

**Current code** (lines 1940–1975, relevant excerpts):
```python
# Line 1940 — tmdb_id already extracted here:
tmdb_id = data.get("_tmdb_id", "")
if tmdb_id and not data.get("Director"):
    itype_hint = "movie" if data.get("Type", "movie") == "movie" else "tv"
    extra = fetch_tmdb_detail(tmdb_id, itype_hint)
    if extra:
        data.update({k: v for k, v in extra.items() if v})

# Lines 1968–1975 — current add_item call:
extra = _extract_extra(data)
ok = self.db.add_item(
    title, itype, rating,
    data.get("Genre", "") or "",
    data.get("Year", "") or "",
    data.get("Plot", "") or "",
    poster_url, user_id=uid, **extra
)
```

**After** `fetch_tmdb_detail` is extended (Pattern 5 above), `data` will contain `_num_seasons` after `data.update(extra)`. The call site becomes:
```python
extra = _extract_extra(data)
ok = self.db.add_item(
    title, itype, rating,
    data.get("Genre", "") or "",
    data.get("Year", "") or "",
    data.get("Plot", "") or "",
    poster_url,
    tmdb_id=data.get("_tmdb_id", ""),
    num_seasons=data.get("_num_seasons", 0),
    user_id=uid,
    **extra
)
```

**Note:** `_num_seasons` is available in `data` at this point because `data.update(fetch_tmdb_detail(...))` at line 1944–1945 now returns it for TV series.

---

### 8. `SmartSearchDialog._add_manual/_worker` — inject `_tmdb_id` (lines 2004–2008)

**Analog:** `_add_result` enrichment block (lines 1933–1945) — same pattern: call TMDb, extract id, update data dict.

**Current `_worker` code** (lines 2004–2008):
```python
def _worker():
    data = smart_search(title)
    data["Type"] = itype_forced        # respetar el tipo que eligió el usuario
    data["Title"] = data.get("Title") or title
    self.after(0, lambda: self._finish_manual(data, itype_forced))
```

**Extended `_worker`** (inject `_tmdb_id` for series when source is TMDb):
```python
def _worker():
    data = smart_search(title)
    data["Type"] = itype_forced        # respetar el tipo que eligió el usuario
    data["Title"] = data.get("Title") or title

    # Inject _tmdb_id for series (smart_search/fetch_tmdb does not return it)
    if (itype_forced == "series"
            and data.get("_source") == "tmdb"
            and CFG.get("tmdb_api_key")):
        candidates = fetch_tmdb_multi(title, type_hint="series")
        if candidates:
            data["_tmdb_id"] = candidates[0].get("_tmdb_id", "")

    self.after(0, lambda: self._finish_manual(data, itype_forced))
```

**Pitfall:** `smart_search` calls `fetch_tmdb` (single-result, line 779), which does NOT include `_tmdb_id` in its return dict (lines 796–805). `fetch_tmdb_multi` DOES include `_tmdb_id` (line 844). This is PITFALL 4 from RESEARCH.md.

---

### 9. `SmartSearchDialog._finish_manual()` call site (lines 2036–2043)

**Analog:** `_add_result` call site (Pattern 7 above) — identical structure.

**Current code** (lines 2036–2043):
```python
extra = _extract_extra(data)
ok = self.db.add_item(
    title, itype_forced, rating,
    data.get("Genre", "") or "",
    data.get("Year", "") or "",
    data.get("Plot", "") or "",
    poster_url, user_id=self.app_root.active_user_id, **extra
)
```

**After** (same change as Pattern 7):
```python
extra = _extract_extra(data)
ok = self.db.add_item(
    title, itype_forced, rating,
    data.get("Genre", "") or "",
    data.get("Year", "") or "",
    data.get("Plot", "") or "",
    poster_url,
    tmdb_id=data.get("_tmdb_id", ""),
    num_seasons=data.get("_num_seasons", 0),
    user_id=self.app_root.active_user_id,
    **extra
)
```

---

### 10. `BatchImportDialog._worker` call site (lines 2347–2354)

**No functional change required for Phase 1.**

**Current code** (lines 2347–2354):
```python
ok = self.db.add_item(
    real_title, itype, rating,
    data.get("Genre", ""),
    data.get("Year", ""),
    data.get("Plot", ""),
    data.get("Poster", ""),
    user_id=uid, **_extract_extra(data)
)
```

`add_item` defaults `tmdb_id=""` and `num_seasons=0`, so this call is unaffected. Series added via batch import will have `tmdb_id=NULL` — this is the retrocompat case handled by BACK-01/02 in Phase 3.

---

### 11. New DB methods for `episode_progress` table

**Analog:** Existing `set_status` upsert pattern in Database class (uses `INSERT OR REPLACE`). New methods use `INSERT ... ON CONFLICT ... DO UPDATE` (UPSERT syntax — more targeted than `INSERT OR REPLACE`).

**Insert location:** After `add_item` (after line 345), in the Database class.

```python
def get_episode_progress(self, user_id: int, item_id: int,
                         season_number: int) -> dict:
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
                        season_number: int) -> tuple:
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

## Shared Patterns

### Logging
**Source:** All existing `fetch_*` functions and `Database._migrate()`
**Apply to:** All new code
```python
log.info(...)    # successful operations
log.debug(...)   # expected/benign failures (column already exists)
log.warning(...) # partial failures (404, 429 — recoverable)
log.error(...)   # genuine errors that affect functionality
```

### HTTP fetch pattern
**Source:** `fetch_tmdb_detail` (lines 852–893)
**Apply to:** `fetch_tmdb_season_episodes`
```python
@lru_cache(maxsize=N)
def fetch_*(arg1, arg2) -> dict:
    key = CFG.get("tmdb_api_key", "")
    if not key or not required_arg:
        return {}
    try:
        url = f"https://api.themoviedb.org/3/..."
        resp = requests.get(url, timeout=10)
        # check status_code BEFORE .json()
        if resp.status_code == 200:
            return resp.json()
        # handle 404/429 distinctly with log.warning
        # handle other errors with log.error
        return {}
    except Exception as e:
        log.error(f"fetch_*: {e}")
    return {}
```

### Threading / after() pattern
**Source:** `_add_result` (line 1966), `RefreshMetaDialog._worker` (lines 2151–2180)
**Apply to:** Any future background fetch that updates UI
```python
# Worker thread — no widget calls
def _worker():
    result = fetch_something(...)
    self.after(0, lambda: self._update_ui(result))

# UI callback — always guard with winfo_exists()
def _update_ui(self, result):
    if not self.winfo_exists():
        return
    # safe to update widgets here
```

---

## No Analog Found

All Phase 1 changes have direct analogs in the existing codebase. No files require patterns from external research only.

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| (none) | — | — | All patterns self-contained within `Videoclub` |

---

## Implementation Order

Per RESEARCH.md recommendation — order matters to avoid migration bugs:

1. **`SCHEMA_VERSION = 7`** (line 158) — first, before any migration code
2. **`_migrate()` ALTER TABLE loop** — add `tmdb_id`/`num_seasons` entries + improve logging
3. **`_migrate()` `if v < 7:` block** — add `episode_progress` table + index
4. **`Database.add_item()`** — extend signature + INSERT
5. **`fetch_tmdb_detail()`** — extend return dict with `_num_seasons`/`_seasons`
6. **`fetch_tmdb_season_episodes()`** — new function (insert after `fetch_tmdb_detail`)
7. **New DB methods** (`get_episode_progress`, `set_episode_watched`, `get_season_progress`)
8. **`_add_result` call site** — pass `tmdb_id`/`num_seasons` to `add_item`
9. **`_add_manual._worker`** — inject `_tmdb_id` via `fetch_tmdb_multi`
10. **`_finish_manual` call site** — pass `tmdb_id`/`num_seasons` to `add_item`

---

## Metadata

**Analog search scope:** `d:/VideoClub/Videoclub` (single file, ~3650 lines)
**Files scanned:** 1
**Pattern extraction date:** 2026-04-14
