# Phase 1: DB + API Foundation — Research

**Researched:** 2026-04-14
**Domain:** SQLite schema migration (v6→v7), Python function signatures, TMDb API season endpoint
**Confidence:** HIGH (based on direct source code inspection of `d:/VideoClub/Videoclub` + prior stack/architecture/pitfalls research)

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| DB-01 | `items` table gains `tmdb_id TEXT` column (non-destructive, idempotent migration) | ALTER TABLE pattern confirmed at lines 272–287; exact DDL in §Standard Stack |
| DB-02 | `items` table gains `num_seasons INTEGER` column | Same ALTER TABLE loop; `number_of_seasons` already in `fetch_tmdb_detail` response |
| DB-03 | New `episode_progress` table with composite PK `(user_id, item_id, season_number, episode_number)` | Full DDL in §Standard Stack; DB methods in §Architecture Patterns |
| DB-04 | `SCHEMA_VERSION` bumped to 7; migration runs only if `v < 7` | Line 158 is the constant; `if v < SCHEMA_VERSION:` gate is at line 315; exact edit in §Standard Stack |
| DB-05 | `tmdb_id` persisted when adding or enriching a series via TMDb | Two call sites: `_add_result` (line 1969) and `_finish_manual` (line 2037); `add_item` signature change in §Architecture Patterns |
| API-01 | `fetch_tmdb_seasons(tmdb_id)` gets season list from `/tv/{id}` (no extra call — data already in detail response) | `fetch_tmdb_detail` already fetches `/tv/{id}`; extend its return dict with `seasons` and `number_of_seasons` |
| API-02 | `fetch_tmdb_season_episodes(tmdb_id, season_number)` fetches episodes from `/tv/{id}/season/{n}` | New function; exact signature and pattern in §Architecture Patterns |
| API-03 | Season list built from `seasons[]` array (not `range(1, N+1)`) to capture Season 0 and irregular numbering | PITFALL C1 confirms this; `seasons` array already present in TMDb TV detail response |
| API-04 | `@lru_cache(maxsize=50)` on `fetch_tmdb_season_episodes` | Matches `fetch_omdb`/`fetch_tmdb_detail` existing pattern; confirmed safe because both args are primitives |
| API-05 | User-visible error when TMDb season endpoint does not return data | HTTP 404/429/network errors need distinct log entries and visible messages; PITFALL M1 documents this |
</phase_requirements>

---

## Summary

Phase 1 makes exactly five categories of change to the single `Videoclub` source file: (1) bump the `SCHEMA_VERSION` constant, (2) extend the `_migrate()` ALTER TABLE loop with two new columns, (3) add a `v < 7` gate creating the `episode_progress` table, (4) add `tmdb_id` and `num_seasons` parameters to `add_item()` and its two call sites, and (5) add `fetch_tmdb_season_episodes()` with `@lru_cache`. No new imports, no new modules, no new dependencies.

The existing migration pattern at lines 272–287 uses a bare `except Exception: pass` that is correct for idempotent column additions but silently swallows genuine errors. The v7 migration block must log errors explicitly rather than continuing with `pass`. The `episode_progress` table creation must use `executescript` in a `v < 7` gate (not inside the bare-except loop) so schema errors surface.

The `fetch_tmdb_detail` function (lines 852–893) already fetches `/tv/{tmdb_id}?append_to_response=credits,...` for TV series, and the response contains both `seasons[]` array and `number_of_seasons`. No additional API call is needed for DB-02 or API-01 — only extending what `fetch_tmdb_detail` returns.

**Primary recommendation:** Make SCHEMA_VERSION=7 the very first edit, then extend the migration method, then update `add_item`, then add the season fetch function — in that exact order. Test each step against the live v6 database before proceeding to the next.

---

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Schema migration (ALTER TABLE, CREATE TABLE) | Database class (`_migrate`) | — | All schema changes live in `Database._migrate()` per existing pattern |
| `tmdb_id` / `num_seasons` persistence | Database class (`add_item`) | SmartSearchDialog (call site) | DB owns INSERT; dialog owns passing the correct values |
| Season episode fetch | Module-level function (`fetch_tmdb_season_episodes`) | — | Matches existing `fetch_tmdb_detail` / `fetch_omdb` module-level pattern |
| Session caching of season data | `@lru_cache` on fetch function | — | Already used for `fetch_tmdb_detail` and `fetch_omdb`; no new mechanism needed |
| Error surfacing to user | Caller (SeasonViewWindow, Phase 2) | fetch function returns `{}` on error | Fetch function logs and returns empty dict; caller shows user-visible message |

---

## Standard Stack

### No new dependencies

All required capabilities are already present:

| Need | Existing Capability | Location |
|------|--------------------|-|
| HTTP requests | `requests` (imported at top) | Used by all `fetch_*` functions |
| In-memory cache | `functools.lru_cache` (imported) | Used by `fetch_omdb`, `fetch_tmdb_detail` |
| SQLite persistence | `sqlite3` (imported) | `Database` class |
| Background loading | `_poster_pool` (ThreadPoolExecutor, line 161) | Phase 2 concern — not needed for Phase 1 functions |

**No `pip install` required.** PyInstaller build unchanged.

### SQL DDL — v7 Migration

#### Column additions (extend existing ALTER TABLE loop at line 272)

```python
# Add these two entries to the list at lines 272–281:
("tmdb_id",     "items",   "TEXT"),
("num_seasons", "items",   "INTEGER"),
```

Full loop after edit (lines 272–287 area):

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
    # v7 additions
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

Note: change `except Exception: pass` to `log.debug(...)` at a minimum — the column-already-exists case is expected and benign but should not be invisible. [VERIFIED: direct code inspection line 286-287]

#### New v7 gate (insert after existing `if v < 6:` block, before `if v < SCHEMA_VERSION:`)

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

**IMPORTANT:** `episode_progress` uses `item_id INTEGER NOT NULL` (FK to `items.id`) — NOT `tmdb_series_id TEXT`. This is the final schema decision recorded in STATE.md (critical note 2). ARCHITECTURE.md also defines this schema at lines 200–213. [VERIFIED: ARCHITECTURE.md lines 200-213, STATE.md critical note 2]

#### `episode_progress` table: DB method signatures

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

[VERIFIED: direct code inspection of `Database` class patterns at lines 205–404; UPSERT syntax confirmed against existing `set_status` pattern]

---

## Architecture Patterns

### System Architecture Diagram

```
TMDb API /tv/{id}/season/{n}
         |
         v
fetch_tmdb_season_episodes(tmdb_id, season_num)  [lru_cache(maxsize=50)]
         |
         | returns dict with "episodes" list
         v
[Phase 2: SeasonViewWindow consumes this]

TMDb API /tv/{id}?append_to_response=credits,...
         |
         v
fetch_tmdb_detail(tmdb_id, "tv")  [existing, lru_cache(maxsize=200)]
         |
         | EXTEND: also return "seasons" list + "num_seasons" int
         v
SmartSearchDialog._add_result (line 1924)
SmartSearchDialog._finish_manual (line 2012)
         |
         | pass tmdb_id= and num_seasons= as new kwargs
         v
Database.add_item(title, itype, ..., tmdb_id="", num_seasons=0, user_id=0)
         |
         | INSERT INTO items ... tmdb_id, num_seasons
         v
SQLite items table (v7 schema: + tmdb_id TEXT, + num_seasons INTEGER)
+ episode_progress table (new)
```

### add_item() Signature Change

**Current** (line 324–326):
```python
def add_item(self, title, itype, rating, genres, year, plot, poster_url,
            director="", actors="", studio="", streaming="",
            user_id: int = 0) -> bool:
```

**New** (add `tmdb_id` and `num_seasons` before `user_id`):
```python
def add_item(self, title, itype, rating, genres, year, plot, poster_url,
            director="", actors="", studio="", streaming="",
            tmdb_id: str = "", num_seasons: int = 0,
            user_id: int = 0) -> bool:
```

**INSERT update** (line 329–333): add `tmdb_id, num_seasons` to column list and values tuple:
```python
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

### Call Site 1: SmartSearchDialog._add_result (line 1924)

**What needs to change:** Line 1969–1974 calls `self.db.add_item(...)`. The `tmdb_id` is already extracted at line 1940 as `tmdb_id = data.get("_tmdb_id", "")`. Need to pass it through.

`num_seasons` comes from extending `fetch_tmdb_detail` return dict (see API patterns below). At the call site, `num_seasons` can be read from `data.get("_num_seasons", 0)` after the detail enrichment at lines 1941–1945.

**Before** (lines 1968–1975):
```python
extra = _extract_extra(data)
ok = self.db.add_item(
    title, itype, rating,
    data.get("Genre", "") or "",
    data.get("Year", "") or "",
    data.get("Plot", "") or "",
    poster_url, user_id=uid, **extra
)
```

**After:**
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

### Call Site 2: SmartSearchDialog._finish_manual (line 2012)

`_finish_manual` is called from `_add_manual` via `smart_search`. `smart_search` calls `fetch_tmdb` (single-result, no `_tmdb_id`), not `fetch_tmdb_multi`. So `_tmdb_id` is NOT in `data` at this call site unless we add it.

**Options:**
1. After `data = smart_search(title)` in `_worker` (line 2005), check if `_source == "tmdb"` and call `fetch_tmdb_multi` to get `_tmdb_id`. This is the cleanest path.
2. Alternatively, modify `_add_manual` to call `fetch_tmdb_multi` before `smart_search`.

**Recommended approach:** In `_worker` inside `_add_manual` (line 2004–2008), after `data = smart_search(title)`, if `_source == "tmdb"` and `itype_forced == "series"`, call `fetch_tmdb_multi(title, type_hint="series")` and take the first result's `_tmdb_id` if the title matches closely. Then set `data["_tmdb_id"] = tmdb_id`. Then `_finish_manual` can pass it through just like call site 1.

**_finish_manual call** (lines 2037–2043) changes mirror call site 1 exactly:
```python
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

### Call Site 3: BatchImportDialog._worker (line 2334)

Line 2347–2354 calls `self.db.add_item(...)` via `_extract_extra`. `BatchImportDialog` uses `smart_search` (which does not return `_tmdb_id`). Since batch import is not a series-specific flow, passing `tmdb_id=""` is acceptable here — series added via batch import will have `tmdb_id=NULL` (retrocompat case handled by BACK-01/02 in Phase 3).

No functional change required at this call site for Phase 1 — `add_item` defaults `tmdb_id=""` so existing callers are unaffected.

### fetch_tmdb_detail: Extend Return Dict

**Current** (lines 852–893): returns `{"Director", "Actors", "Studio", "Streaming"}` only.

**Extension needed:** When `media_type != "movie"` (i.e., TV series), also return `_num_seasons` and `_seasons` from the response dict `d`:

```python
# At line 889, extend the return dict for TV:
result = {"Director": director, "Actors": actors,
          "Studio": studio, "Streaming": streaming}
if media_type != "movie":
    result["_num_seasons"] = d.get("number_of_seasons", 0)
    result["_seasons"] = d.get("seasons", [])
return result
```

This means `_add_result` (call site 1) can read `data.get("_num_seasons", 0)` after `data.update(fetch_tmdb_detail(...))` at line 1944–1945.

### fetch_tmdb_season_episodes: New Function

Insert after `fetch_tmdb_detail` (after line 893), before `fetch_duckduckgo`:

```python
@lru_cache(maxsize=50)
def fetch_tmdb_season_episodes(tmdb_id: str, season_number: int) -> dict:
    """Returns TMDb season object with 'episodes' list.
    Cached per session — (tmdb_id, season_number) are both primitives (hashable).
    Returns {} on any error; caller is responsible for user-visible messaging."""
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
            return {}
        elif resp.status_code == 429:
            log.warning(f"fetch_tmdb_season_episodes: 429 rate limit tmdb_id={tmdb_id} season={season_number}")
            import time; time.sleep(1)
            resp2 = requests.get(url, timeout=10)
            if resp2.status_code == 200:
                return resp2.json()
            log.error(f"fetch_tmdb_season_episodes: retry after 429 also failed ({resp2.status_code})")
            return {}
        else:
            log.error(f"fetch_tmdb_season_episodes: HTTP {resp.status_code} tmdb_id={tmdb_id} season={season_number}")
            return {}
    except Exception as e:
        log.error(f"fetch_tmdb_season_episodes: network error: {e}")
    return {}
```

**Critical design notes:**
- `@lru_cache` requires both arguments to be hashable — `str` and `int` are both hashable. Safe. [VERIFIED: Python docs]
- Do NOT call `.json()` before checking `resp.status_code`. TMDb returns JSON error bodies on 404/429 that look valid. [VERIFIED: PITFALLS.md M1]
- The `time.sleep(1)` retry on 429 is acceptable for a single-user desktop app with lazy tab loading. [VERIFIED: STACK.md §1.4]
- The function returns `{}` (empty dict) on all error paths. The Phase 2 caller (SeasonViewWindow) is responsible for checking `data.get("episodes", [])` and showing a user-visible message when empty. This satisfies API-05. [ASSUMED: exact UI message text is Phase 2 concern]

### Season List Construction (API-03)

When building season tabs in Phase 2, iterate the `_seasons` list, NOT `range(1, num_seasons+1)`:

```python
# CORRECT — use seasons array from fetch_tmdb_detail extended return
seasons_data = item.get("_seasons", [])  # or re-call fetch_tmdb_detail
for s in seasons_data:
    season_num  = s["season_number"]    # may be 0 (Specials)
    season_name = s.get("name") or (
        "Especiales" if season_num == 0 else f"Temporada {season_num}"
    )
    ep_count = s.get("episode_count", 0)
```

This satisfies API-01 and API-03. The `_seasons` array is available from the extended `fetch_tmdb_detail` return at add-time and is also available by calling `fetch_tmdb_detail(tmdb_id, "tv")` again from SeasonViewWindow (it is `lru_cache`'d).

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Session-level episode data cache | Custom dict + TTL logic | `@lru_cache(maxsize=50)` | Already in stdlib; (tmdb_id, season_number) are hashable primitives; eviction is automatic |
| Idempotent column additions | Custom schema version checks per column | `ALTER TABLE ... ADD COLUMN` in `try/except` | SQLite raises OperationalError when column exists; catching it IS the idempotency mechanism |
| Upsert for episode watched state | SELECT then INSERT or UPDATE | `INSERT ... ON CONFLICT ... DO UPDATE` | One atomic statement; no race condition; already used in `set_status` pattern |
| HTTP retry on 429 | Thread-based retry with queue | `time.sleep(1)` + one retry inline | Desktop app, single user; no concurrency; over-engineering a retry manager is wasteful |

---

## Common Pitfalls

### Pitfall 1: SCHEMA_VERSION not bumped before migration code is written
**What goes wrong:** If `SCHEMA_VERSION` stays at 6, the `if v < SCHEMA_VERSION:` gate at line 315 fires `_set_version(6)` — which writes 6 to the meta table. The `if v < 7:` block runs on every launch because `v` starts at 6 and is immediately re-written to 6. Migration logic runs repeatedly. `ALTER TABLE` silently skips, but `episode_progress` gets re-created attempts every launch.
**How to avoid:** Change line 158 `SCHEMA_VERSION = 6` to `SCHEMA_VERSION = 7` as the FIRST edit, before writing any migration code.
**Source:** PITFALLS.md Pitfall m3, STATE.md "Critical Implementation Notes" note 1. [VERIFIED]

### Pitfall 2: Bare `except Exception: pass` swallows migration errors
**What goes wrong:** The existing loop at lines 283–287 uses `except Exception: pass`. If the column definition is wrong (e.g., `NOT NULL` without `DEFAULT` on an existing table), the `ALTER TABLE` raises an `OperationalError` that is silently swallowed. The column is never added; subsequent INSERTs that assume the column exists fail with "table items has no column named tmdb_id" at runtime, not at startup.
**How to avoid:** Change `pass` to `log.debug(f"_migrate: {col} already exists: {e}")`. For the `v < 7` block, use explicit `try/except` with `log.error(...)`, never bare `pass`.
**Source:** PITFALLS.md Pitfall C3. [VERIFIED: lines 283-287]

### Pitfall 3: `NOT NULL` column without `DEFAULT` on existing table
**What goes wrong:** SQLite does not allow `ALTER TABLE ... ADD COLUMN x TEXT NOT NULL` on a table that already has rows. The `OperationalError` is swallowed by bare `except pass`, leaving the column missing silently.
**How to avoid:** `tmdb_id TEXT` (nullable, no constraint) and `num_seasons INTEGER` (nullable) — never `NOT NULL` without `DEFAULT`. Existing rows will have `NULL` values, which is correct behavior.
**Source:** PITFALLS.md Pitfall C3, SQLite documentation. [VERIFIED]

### Pitfall 4: `_tmdb_id` is in `fetch_tmdb_multi` results but NOT in `smart_search` / `fetch_tmdb`
**What goes wrong:** `_add_result` already has `_tmdb_id` because it displays `fetch_tmdb_multi` results. But `_finish_manual` calls `smart_search` → `fetch_tmdb` (line 779), whose return dict (lines 796–805) does NOT include `_tmdb_id`. So `data.get("_tmdb_id", "")` at `_finish_manual` call site always returns `""`.
**How to avoid:** In `_add_manual._worker`, after `data = smart_search(title)`, if source is TMDb, call `fetch_tmdb_multi(title, type_hint=itype_forced)` to get the first result and extract `_tmdb_id`. Set `data["_tmdb_id"] = ...` before calling `_finish_manual`.
**Source:** Direct inspection of `fetch_tmdb` (lines 778-808) vs `fetch_tmdb_multi` (lines 811-849). [VERIFIED]

### Pitfall 5: `lru_cache` invalidation — cached empty dict masks retry
**What goes wrong:** If `fetch_tmdb_season_episodes` returns `{}` due to a transient network error, that `{}` is cached. Subsequent calls for the same `(tmdb_id, season_number)` within the same session get the empty dict from cache, never retrying. User sees "no data" permanently until app restart.
**How to avoid:** Only cache successful responses. Use a conditional: if the result is empty, do NOT let lru_cache store it. Implement with a wrapper pattern:
```python
@lru_cache(maxsize=50)
def _fetch_tmdb_season_cached(tmdb_id: str, season_number: int) -> dict:
    ...  # the actual HTTP call

def fetch_tmdb_season_episodes(tmdb_id: str, season_number: int) -> dict:
    result = _fetch_tmdb_season_cached(tmdb_id, season_number)
    if not result:
        _fetch_tmdb_season_cached.cache_clear()  # clear so next call retries
    return result
```
Alternatively, accept the behavior and document that a 404 is a valid permanent state (no data in TMDb). For transient errors, clear cache explicitly if needed.
**Note:** For v1, a simpler approach is acceptable: empty dict result is cached; user can restart the app if they get a transient failure. Document this as a known limitation.

### Pitfall 6: `episode_number` as list index vs TMDb field
**What goes wrong:** If you store `enumerate(episodes)` index as `episode_number` in `episode_progress`, and TMDb later corrects episode numbering (or a season has gaps), the stored rows reference the wrong episodes.
**How to avoid:** Always use `ep.get("episode_number")` from the TMDb `episodes` array. Never use the list index.
**Source:** PITFALLS.md Pitfall M2. [VERIFIED]

### Pitfall 7: `fetch_tmdb_detail` called without status check — `.json()` on error body
**What goes wrong:** The existing `fetch_tmdb_detail` at line 864 calls `requests.get(...).json()` directly. TMDb returns JSON bodies for 404/429 errors that include `{"status_code": 34, "status_message": "..."}` — these parse without exception but contain no useful data, potentially returning partial dicts that contaminate `items` table with empty strings.
**How to avoid:** `fetch_tmdb_season_episodes` must check `resp.status_code == 200` before calling `.json()`. Do not replicate the existing `fetch_tmdb_detail` pattern for the new function.
**Source:** PITFALLS.md Pitfall M1. [VERIFIED]

---

## Code Examples

### Pattern: Existing migration gate structure (reference before editing)

```python
# Source: Videoclub lines 269-316 (verified)
def _migrate(self):
    v = self._get_version()
    log.info(f"DB schema: v{v} → v{SCHEMA_VERSION}")
    for col, tbl, defn in [...]:
        try:
            self.conn.execute(f"ALTER TABLE {tbl} ADD COLUMN {col} {defn}")
            self.conn.commit()
        except Exception:
            pass          # ← THIS changes to log.debug(...)

    if v < 4: ...
    if v < 6: ...
    # INSERT v < 7 block here (before the version gate below)
    if v < SCHEMA_VERSION:
        self._set_version(SCHEMA_VERSION)
```

### Pattern: Existing `lru_cache` on API function (reference)

```python
# Source: Videoclub lines 852-893 (verified)
@lru_cache(maxsize=200)
def fetch_tmdb_detail(tmdb_id: str, media_type: str = "movie") -> dict:
    key = CFG.get("tmdb_api_key", "")
    if not key or not tmdb_id:
        return {}
    try:
        ...
        return {"Director": director, "Actors": actors, ...}
    except Exception as e:
        log.error(f"fetch_tmdb_detail: {e}")
    return {}
```

### Pattern: Existing `add_item` INSERT (reference before editing)

```python
# Source: Videoclub lines 328-334 (verified)
self.conn.execute(
    "INSERT OR IGNORE INTO items"
    "(title,type,rating,genres,year,plot,poster_url,director,actors,studio,streaming)"
    " VALUES(?,?,?,?,?,?,?,?,?,?,?)",
    (title, itype, rating, genres, year, plot, poster_url,
     director or "", actors or "", studio or "", streaming or "")
)
```

---

## Runtime State Inventory

> This is a migration phase — schema changes to existing database.

| Category | Items Found | Action Required |
|----------|-------------|------------------|
| Stored data | SQLite `items` table: existing rows will have `tmdb_id=NULL`, `num_seasons=NULL` after migration | No data migration required; NULL values are correct (retrocompat, BACK-01/02) |
| Stored data | `episode_progress` table: does not exist in v6 | Created by `v < 7` migration block |
| Live service config | None — app is a local desktop tool, no external service config | None |
| OS-registered state | None — no scheduled tasks or OS registrations | None |
| Secrets/env vars | `tmdb_api_key` in `CFG` dict — key name unchanged | None |
| Build artifacts | PyInstaller .exe if it exists — schema changes are runtime, not build-time | Rebuild not required; schema migration runs at app startup |

**Existing rows after migration:** All existing `items` rows will have `tmdb_id = NULL` and `num_seasons = NULL`. This is correct — the "Ver temporadas" button in Phase 2 will be disabled for these rows (BACK-01). No backfill is needed in Phase 1.

---

## Environment Availability

> Phase 1 is entirely code/config changes to a single Python file — no new external dependencies.

Step 2.6: SKIPPED (no new external dependencies; `requests` and `sqlite3` are already present and verified working in the existing app).

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Manual testing against live SQLite database (no automated test suite exists) |
| Config file | None — single-file app, no test runner config |
| Quick run command | Run `python Videoclub` against existing v6 DB |
| Full suite command | Same — check PRAGMA + SQLite browser after startup |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | How to Verify |
|--------|----------|-----------|---------------|
| DB-01 | `tmdb_id TEXT` column in `items` | Manual | `PRAGMA table_info(items)` in SQLite browser after first launch |
| DB-02 | `num_seasons INTEGER` column in `items` | Manual | Same PRAGMA query |
| DB-03 | `episode_progress` table exists with correct schema | Manual | `SELECT * FROM sqlite_master WHERE name='episode_progress'` |
| DB-04 | `SCHEMA_VERSION=7`; migration runs once, not on every launch | Manual | Check `meta` table: `SELECT value FROM meta WHERE key='schema_version'` — must show `7` |
| DB-05 | Adding a TMDb series persists non-null `tmdb_id` | Manual | Add a known series (e.g., "Breaking Bad") via search; `SELECT tmdb_id FROM items WHERE title='Breaking Bad'` |
| API-01 | `fetch_tmdb_detail` returns `_seasons` and `_num_seasons` for TV | Manual (Python REPL) | `fetch_tmdb_detail("1396", "tv")` — inspect return dict keys |
| API-02 | `fetch_tmdb_season_episodes` returns dict with `episodes` list | Manual (Python REPL) | `fetch_tmdb_season_episodes("1396", 1)` — check `result.get("episodes")` is non-empty list |
| API-03 | Season list from `seasons[]`, not `range()` | Code review | Verify no `range(1, N+1)` pattern in new season-building code |
| API-04 | Second call to `fetch_tmdb_season_episodes` with same args returns cached result | Manual (Python REPL) | Call twice; second call returns instantly (add `print` to function body to verify cache hit) |
| API-05 | HTTP 404/429/network errors produce log entry and return `{}` | Manual | Mock a bad `tmdb_id`; check log file for warning/error entries |

### Phase 1 Success Criteria (from ROADMAP.md)

1. Running app against v6 DB completes startup without errors; `PRAGMA table_info(items)` confirms `tmdb_id` and `num_seasons` columns; `sqlite_master` confirms `episode_progress` table and indexes exist.
2. Adding a series via TMDb search persists non-null `tmdb_id` and `num_seasons` in the `items` row.
3. `fetch_tmdb_season_episodes(tmdb_id, season_number)` from Python REPL returns dict with `episodes` list; second call returns cached value without network request.
4. Season list for a series is built from TMDb `seasons[]` array, so Season 0 appears when it exists in TMDb data.
5. HTTP 404, 429, and network errors produce distinct log entries and return `{}` (user-visible message is Phase 2 concern).

---

## Exact Line Numbers for Every Change

| Change | File Location | Current Content | Action |
|--------|--------------|-----------------|--------|
| Bump SCHEMA_VERSION | Line 158 | `SCHEMA_VERSION = 6` | Change to `= 7` |
| Fix bare except in migration loop | Lines 283–287 | `except Exception: pass` | Change to `except Exception as e: log.debug(...)` |
| Extend ALTER TABLE loop | Lines 272–282 | 8 entries in list | Add `("tmdb_id", "items", "TEXT")` and `("num_seasons", "items", "INTEGER")` |
| Add v7 migration gate | After line 313 (after `if v < 6:` block) | `if v < SCHEMA_VERSION:` | Insert new `if v < 7:` block before this line |
| Update `add_item` signature | Lines 324–326 | No `tmdb_id`, `num_seasons` params | Add `tmdb_id: str = ""` and `num_seasons: int = 0` |
| Update `add_item` INSERT | Lines 329–334 | 11 columns in INSERT | Extend to 13 columns including `tmdb_id, num_seasons` |
| Add DB methods | After `update_meta` at line 702 | `def close(self):` at line 704 | Insert 3 new methods before `close()` |
| Extend `fetch_tmdb_detail` return | Lines 889–890 | Returns 4-key dict | Add `_num_seasons` and `_seasons` keys for TV |
| Add `fetch_tmdb_season_episodes` | After line 893 | `def fetch_duckduckgo(...)` at line 896 | Insert new cached function |
| Fix call site 1 (`_add_result`) | Lines 1968–1975 | `add_item(... **extra)` | Add `tmdb_id=` and `num_seasons=` kwargs |
| Fix call site 2 (`_finish_manual`) | Lines 2036–2043 | `add_item(... **extra)` | Add `tmdb_id=` and `num_seasons=` kwargs |
| Fix `_add_manual._worker` | Lines 2004–2008 | `data = smart_search(title)` | After smart_search, resolve `_tmdb_id` via `fetch_tmdb_multi` if source is tmdb |
| Add shutdown for season pool | Line 3616 | `_poster_pool.shutdown(wait=False)` | No pool needed for Phase 1 (lru_cache only) — Phase 2 concern |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `BatchImportDialog` does not need `tmdb_id` threading in Phase 1 (batch-added series are retrocompat, BACK-01 handles them) | Call Site 3 analysis | If wrong: batch-imported series can never track episodes without re-adding; acceptable per BACK-01/02 design |
| A2 | The exact message text for API-05 user-visible error is a Phase 2 concern; Phase 1 only needs `{}` return + log entry | API-05 in test map | If wrong: Phase 2 planner needs to also specify exact error message strings |
| A3 | `_season_pool` (ThreadPoolExecutor) is NOT needed in Phase 1 since `fetch_tmdb_season_episodes` is synchronous and only consumed by Phase 2 UI | Architecture | If wrong: Phase 1 would need to add `_season_pool` and its shutdown — adds 2 extra line changes |

---

## Open Questions

1. **Should `_finish_manual` resolve `tmdb_id` via `fetch_tmdb_multi` or accept `tmdb_id=""` silently?**
   - What we know: `smart_search` → `fetch_tmdb` does not return `_tmdb_id`. `_add_manual` uses `smart_search`.
   - What's unclear: Is it acceptable for series added via "Añadir desde plataforma externa" to have `tmdb_id=NULL` in v1? The BACK-01/02 retrocompat requirements suggest yes — but those requirements are for Phase 3.
   - Recommendation: Add the `fetch_tmdb_multi` resolution in `_add_manual._worker` for clean v1 behavior. The code is simple (3 lines). If time-constrained, accept `NULL` and let BACK-01/02 handle it.

2. **Should `fetch_tmdb_detail` extension be guarded to avoid breaking existing `movie` callers?**
   - What we know: `fetch_tmdb_detail` is called with `media_type="movie"` in many places; extending the return dict only for `media_type != "movie"` does not break movie callers.
   - What's unclear: Does anything do `if "Director" in fetch_tmdb_detail(...)` that would break if extra keys are present?
   - Recommendation: The extension is additive (new keys only); Python callers using `.get()` are unaffected. Safe.

---

## Sources

### Primary (HIGH confidence — direct code inspection)
- `d:/VideoClub/Videoclub` lines 158, 205–404 (Database class, schema, migration)
- `d:/VideoClub/Videoclub` lines 711–893 (all fetch functions)
- `d:/VideoClub/Videoclub` lines 1924–1988 (`_add_result` call site)
- `d:/VideoClub/Videoclub` lines 2004–2054 (`_add_manual`, `_finish_manual` call sites)
- `d:/VideoClub/Videoclub` lines 2329–2370 (`BatchImportDialog._worker` call site)
- `d:/VideoClub/Videoclub` line 3616 (shutdown handler)
- `d:/VideoClub/.planning/research/STACK.md` — SQL DDL, TMDb endpoint structure, caching strategy
- `d:/VideoClub/.planning/research/ARCHITECTURE.md` — episode_progress schema with `item_id`, DB method signatures, data flow
- `d:/VideoClub/.planning/research/PITFALLS.md` — all pitfall patterns cited
- `d:/VideoClub/.planning/STATE.md` — critical implementation notes (schema decision: `item_id` not `tmdb_series_id`)

### Secondary (MEDIUM confidence)
- TMDb API v3 `/tv/{id}/season/{n}` endpoint structure: confirmed from existing `fetch_tmdb_detail` usage pattern and STACK.md documentation
- SQLite `ALTER TABLE ... ADD COLUMN` with NOT NULL constraint: well-established limitation confirmed by PITFALLS.md C3

---

## Metadata

**Confidence breakdown:**
- Standard stack (SQL DDL, function signatures): HIGH — all verified against source code
- Architecture (call site changes, data flow): HIGH — traced through source code line by line
- Pitfalls: HIGH — verified against actual code patterns in source file
- API-05 error handling details: MEDIUM — HTTP status code handling is standard pattern; exact TMDb error response bodies documented in PITFALLS.md

**Research date:** 2026-04-14
**Valid until:** Indefinite (single source file, no external dependency changes)
