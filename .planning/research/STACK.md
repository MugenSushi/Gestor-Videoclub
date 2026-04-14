# Technology Stack — Season/Episode Tracking

**Project:** Videoclub STARTUP v7.0 — Series episode tracking  
**Researched:** 2026-04-14  
**Confidence:** HIGH (based on direct code inspection + TMDb API v3 docs)

---

## 1. TMDb API — Season & Episode Endpoints

### 1.1 Season Details (primary endpoint)

```
GET https://api.themoviedb.org/3/tv/{series_id}/season/{season_number}
    ?api_key={key}
    &language=es-ES
```

- `series_id`: TMDb numeric ID for the TV show (integer). Already returned as `_tmdb_id` by `fetch_tmdb_multi`.
- `season_number`: Integer. Season 0 = specials/extras in TMDb — skip unless explicitly wanted.
- `language`: Use `es-ES` to match the existing app convention (already used in `fetch_tmdb_detail`).
- No pagination: a single call returns ALL episodes for that season. Typical season has 8–24 episodes. The response is one JSON object, not a paged list.

**Response structure (relevant fields):**

```json
{
  "id": 12345,
  "name": "Season 1",
  "overview": "...",
  "season_number": 1,
  "air_date": "2019-04-14",
  "episodes": [
    {
      "id": 67890,
      "episode_number": 1,
      "season_number": 1,
      "name": "Winter Is Coming",
      "overview": "...",
      "air_date": "2011-04-17",
      "runtime": 62,
      "still_path": "/path.jpg",
      "vote_average": 8.9
    },
    ...
  ],
  "poster_path": "/path.jpg"
}
```

### 1.2 TV Series Details (to get season count)

Use the already-existing `fetch_tmdb_detail` pattern:

```
GET https://api.themoviedb.org/3/tv/{series_id}
    ?api_key={key}
    &language=es-ES
```

Key fields in response:
- `number_of_seasons` (integer) — how many season tabs to show
- `number_of_episodes` (integer total)
- `seasons` (array) — each element has `season_number`, `episode_count`, `name`, `air_date`, `poster_path`

This avoids fetching all season details upfront; build tabs from the `seasons` array then lazy-load per season on tab click.

### 1.3 Per-Episode Details (do NOT use for bulk loading)

```
GET https://api.themoviedb.org/3/tv/{series_id}/season/{season_number}/episode/{episode_number}
    ?api_key={key}
    &language=es-ES
```

This endpoint exists but makes one HTTP request per episode. For a 24-episode season that is 24 calls. Do not use this for the Season View. Use the season-level endpoint (1.1) which returns all episodes in one call.

### 1.4 Rate Limits

TMDb API v3 enforces **~40 requests per 10 seconds** per API key (this is the documented soft limit; the hard limit triggers HTTP 429). In practice:

- The Season View fetches 1 call per season tab opened (lazy-load model). A typical series with 5 seasons = 5 calls spread across user clicks. Well within limits.
- The initial series detail call (to get season list) = 1 call. Total budget for opening Season View: 2 calls (detail + first season). Non-issue.
- No pagination concern: TMDb returns all episodes for a season in a single response. There is no `page` parameter on the season endpoint.
- Retry on 429: add `time.sleep(0.5)` and one retry inside the fetch function. This is sufficient for a single-user desktop app.

---

## 2. SQLite Schema — v7 Migration

### 2.1 New column on `items`: `tmdb_id`

The `items` table currently has no `tmdb_id` column. The `_tmdb_id` is returned by `fetch_tmdb_multi` but is never persisted. This must be stored to enable the Season View to call TMDb without re-searching.

Add via `ALTER TABLE` in the v7 migration block (consistent with how v5 columns were added):

```sql
ALTER TABLE items ADD COLUMN tmdb_id TEXT;
```

Type is TEXT (not INTEGER) for consistency with how `_tmdb_id` is handled in the existing code (`str(r.get("id", ""))`). An index is useful for direct lookup:

```sql
CREATE INDEX IF NOT EXISTS idx_items_tmdb_id ON items(tmdb_id);
```

Existing rows will have `tmdb_id = NULL`. That is correct — the Season View button should be disabled/hidden for items with NULL tmdb_id, showing "TMDb ID not available; re-add via search to enable episode tracking."

### 2.2 New table: `episode_progress`

```sql
CREATE TABLE IF NOT EXISTS episode_progress (
    user_id        INTEGER NOT NULL,
    tmdb_series_id TEXT    NOT NULL,
    season_number  INTEGER NOT NULL,
    episode_number INTEGER NOT NULL,
    watched        INTEGER NOT NULL DEFAULT 0,   -- 0 = unwatched, 1 = watched
    watched_at     TEXT,                          -- ISO datetime, NULL if unwatched
    PRIMARY KEY (user_id, tmdb_series_id, season_number, episode_number),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX IF NOT EXISTS idx_ep_series
    ON episode_progress(user_id, tmdb_series_id);
```

**Column rationale:**

| Column | Type | Rationale |
|--------|------|-----------|
| `user_id` | INTEGER | Multi-user support matches existing `watched` table design |
| `tmdb_series_id` | TEXT | Decoupled from `items.title` (which is mutable / not a stable FK); TMDb IDs are stable |
| `season_number` | INTEGER | Natural episode address component |
| `episode_number` | INTEGER | Natural episode address component |
| `watched` | INTEGER | SQLite has no BOOLEAN; 0/1 integer is idiomatic |
| `watched_at` | TEXT | ISO 8601 datetime (`datetime('now')`) for future "watched history" without schema change |

**Why NOT foreign key to `items`:** The `watched` table already uses `title TEXT` as a foreign key, which is fragile (title renames break it). `episode_progress` is intentionally keyed by `tmdb_series_id` (stable numeric ID) to avoid that coupling. The series can be found in `items` via `items.tmdb_id = episode_progress.tmdb_series_id` when needed.

**Why NOT a separate `episodes` metadata table:** See section 3 below.

### 2.3 Full v7 migration block

This goes in `_migrate()` as a new `if v < 7:` block:

```python
SCHEMA_VERSION = 7  # bump from 6
```

```python
if v < 7:
    # Add tmdb_id to items
    try:
        self.conn.execute("ALTER TABLE items ADD COLUMN tmdb_id TEXT")
        self.conn.commit()
    except Exception:
        pass  # already exists

    self.conn.executescript("""
        CREATE INDEX IF NOT EXISTS idx_items_tmdb_id
            ON items(tmdb_id);

        CREATE TABLE IF NOT EXISTS episode_progress (
            user_id        INTEGER NOT NULL,
            tmdb_series_id TEXT    NOT NULL,
            season_number  INTEGER NOT NULL,
            episode_number INTEGER NOT NULL,
            watched        INTEGER NOT NULL DEFAULT 0,
            watched_at     TEXT,
            PRIMARY KEY (user_id, tmdb_series_id, season_number, episode_number),
            FOREIGN KEY (user_id) REFERENCES users(id)
        );

        CREATE INDEX IF NOT EXISTS idx_ep_series
            ON episode_progress(user_id, tmdb_series_id);
    """)
    self.conn.commit()
```

---

## 3. Store Metadata vs Fetch Every Time

**Decision: Store the minimum, fetch episode metadata from TMDb on demand, cache in memory.**

### What to store in DB (episode_progress only):

Only the bare progress record: `user_id`, `tmdb_series_id`, `season_number`, `episode_number`, `watched`, `watched_at`. No episode title, no air date, no overview.

### What NOT to store in DB:

- Episode titles, air dates, overviews, runtime — these are TMDb's content, subject to correction/localization. Storing them creates a staleness problem for a desktop app that has no background sync.
- Season poster paths — same reason.

### Why this is correct for this app:

1. **Data volume:** A series with 8 seasons of 13 episodes is 104 rows of only integers/short strings. The season metadata (titles, dates) for the same series is ~50 KB of JSON. Storing it in SQLite would require a `seasons` + `episodes` metadata schema and a cache invalidation strategy. The complexity cost is not justified.

2. **TMDb is already the source of truth:** The app is already online-first for metadata (posters, ratings, plot all come from API). Episode names/dates are additional metadata, same category.

3. **Rate limits are not a concern at this scale:** Opening a season = 1 API call. A user is unlikely to open the same season repeatedly in rapid succession. The memory-level cache (see section 4) handles repeated opens in the same session.

4. **Graceful degradation:** If the user is offline, the Season View can show episode numbers and watched state from the DB, displaying "—" for title/date until TMDb responds. This is better UX than "no data at all."

---

## 4. Caching Strategy for Season Data

### Layer 1: In-memory `lru_cache` per session

Match the pattern already used for `fetch_tmdb_detail`:

```python
@lru_cache(maxsize=50)
def fetch_tmdb_season(tmdb_id: str, season_number: int) -> dict:
    """Returns full TMDb season object including episodes list."""
    key = CFG.get("tmdb_api_key", "")
    if not key or not tmdb_id:
        return {}
    try:
        url = (f"https://api.themoviedb.org/3/tv/{tmdb_id}/season/{season_number}"
               f"?api_key={key}&language=es-ES")
        return requests.get(url, timeout=10).json()
    except Exception as e:
        log.error(f"fetch_tmdb_season: {e}")
    return {}
```

`maxsize=50` covers ~6 series with 8 seasons each — more than enough for a single session.

`lru_cache` uses `(tmdb_id, season_number)` as the key since both parameters are primitives (strings/ints), which are hashable. This is consistent with how `fetch_tmdb_detail(tmdb_id, media_type)` works.

### Layer 2: ThreadPoolExecutor for non-blocking load

The Season View should load season data in the existing `ThreadPoolExecutor` (4 workers, already in the app). Pattern:

```python
def _load_season_async(self, tmdb_id, season_num):
    future = _POOL.submit(fetch_tmdb_season, tmdb_id, season_num)
    self.after(100, lambda: self._check_season_future(future, season_num))
```

This mirrors how poster loading works today: submit to pool, poll with `after()`, update UI on main thread when done. Do not use `threading.Thread` directly — the existing pool is already configured and bounded.

### Layer 3: No disk cache needed for v1

Disk caching (SQLite or file-based) would require cache invalidation logic (TTL, version checks). For episode metadata that changes rarely (only when TMDb corrects data or new episodes air), the in-memory cache per session is sufficient. If a user closes and reopens the app, the one-call-per-season cost is acceptable for a desktop app.

---

## 5. Supporting Libraries — No New Dependencies

All required capabilities are already in the app:

| Need | Existing Capability | Notes |
|------|--------------------|-|
| HTTP requests to TMDb | `requests` (already imported) | Use same pattern as `fetch_tmdb_detail` |
| In-memory caching | `functools.lru_cache` (already imported) | `@lru_cache(maxsize=50)` on new fetch function |
| Background loading | `ThreadPoolExecutor` (`_POOL`, already initialized) | Submit season fetch jobs to existing pool |
| UI | `tkinter.ttk` (already imported) | `ttk.Notebook` for season tabs, `ttk.Checkbutton` for episode rows |
| DB migrations | `sqlite3` (already used) | `ALTER TABLE` + `executescript` in `_migrate()` |

No `pip install` required. The .exe build via PyInstaller does not need changes.

---

## 6. Key Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Season data endpoint | `/tv/{id}/season/{n}` | Single call returns all episodes; no pagination |
| Series metadata endpoint | `/tv/{id}` (reuse `fetch_tmdb_detail` pattern) | Get `seasons` array for tab count without fetching all episodes upfront |
| Per-episode endpoint | Not used for Season View | Would require N calls for N episodes; use season endpoint instead |
| tmdb_id storage | `items.tmdb_id TEXT` via ALTER TABLE | Stable ID needed for API; TEXT matches existing `_tmdb_id` handling |
| Progress schema | `episode_progress` keyed by `(user_id, tmdb_series_id, season_number, episode_number)` | Stable, decoupled from mutable `title` FK; supports multi-user |
| Episode metadata in DB | No — fetch from TMDb, cache in memory | Avoids staleness, no new schema complexity, rate limits not a concern |
| Disk caching | No — LRU in-memory only | Session-level cache sufficient; no TTL/invalidation complexity |
| New libraries | None | All needs covered by existing imports |
| Schema version | 6 → 7 | Non-destructive: only ALTER TABLE + new table |

---

## Sources

- Code inspection: `D:/VideoClub/Videoclub` (lines 206-316 for schema, 778-893 for TMDb functions)
- TMDb API v3 documented endpoint pattern confirmed from existing `fetch_tmdb_detail` usage: `https://api.themoviedb.org/3/tv/{tmdb_id}?api_key=...&append_to_response=credits,...`
- PROJECT.md confirms: "El endpoint de temporadas es `/tv/{series_id}/season/{season_number}`"
- TMDb rate limit: 40 req/10 sec is the widely-documented soft limit for API v3 (confirmed by existing app making no special rate-limit handling, which works in production)
- Confidence: HIGH — TMDb API v3 has been stable since 2013; the endpoints used here are core, not beta
