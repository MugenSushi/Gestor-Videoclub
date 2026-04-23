---
phase: 01-db-api-foundation
plan: "02"
subsystem: api
tags: [tmdb, lru_cache, seasons, error-handling, http]
dependency_graph:
  requires: [schema-v7]
  provides: [fetch-tmdb-season-episodes, tmdb-season-api-layer]
  affects: [SmartSearchDialog, SeasonViewWindow]
tech_stack:
  added: []
  patterns: [two-function lru_cache split, 404-sentinel caching, 429-retry-once, cache_clear on transient error]
key_files:
  created: []
  modified:
    - d:/VideoClub/Videoclub
decisions:
  - "Two-function split (_fetch_tmdb_season_cached + fetch_tmdb_season_episodes) prevents caching transient empty-dict results — only real 200 responses and permanent 404 sentinels are cached"
  - "{'_404': True} sentinel cached so repeated calls to non-existent seasons skip network; caller receives {} (empty) in both 404 and transient-error cases"
  - "429 handled with one-time 1s sleep + retry; if retry also fails, returns {} (not cached) so next session call retries"
  - "media_type != 'movie' guard in fetch_tmdb_detail ensures movies receive no _num_seasons/_seasons keys — backward compatible"
metrics:
  duration: "62 seconds"
  completed: "2026-04-17"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 1
requirements: [API-01, API-02, API-03, API-04, API-05]
---

# Phase 01 Plan 02: TMDb Season API Layer Summary

**One-liner:** fetch_tmdb_detail extended to return _num_seasons/_seasons for TV series, and new fetch_tmdb_season_episodes added with @lru_cache(maxsize=50), two-function transient-error split, and distinct 404/429/network error handling.

---

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Extend fetch_tmdb_detail return dict with _num_seasons and _seasons | 5806626 | d:/VideoClub/Videoclub |
| 2 | Add fetch_tmdb_season_episodes with lru_cache and error handling | 653b911 | d:/VideoClub/Videoclub |

---

## What Was Built

### Task 1: fetch_tmdb_detail return dict extension

- Replaced the single-line `return {"Director": ..., "Streaming": ...}` with a `result` dict pattern
- Added `if media_type != "movie":` guard before inserting season fields
- `result["_num_seasons"] = d.get("number_of_seasons", 0)` — integer season count from TMDb TV detail response (no extra API call)
- `result["_seasons"] = d.get("seasons", [])` — full seasons array from TMDb TV detail response (includes season_number=0 for Especiales)
- Movies are fully unaffected — the guard prevents adding season keys to movie responses
- The `except` block returning `{}` on error is unchanged

### Task 2: fetch_tmdb_season_episodes + _fetch_tmdb_season_cached

- `_fetch_tmdb_season_cached(tmdb_id: str, season_number: int) -> dict` — private function decorated with `@lru_cache(maxsize=50)`
  - Returns full TMDb season JSON on 200
  - Returns `{"_404": True}` sentinel on 404 (cached — permanent, non-retryable)
  - On 429: logs warning, sleeps 1s, retries once; returns `{}` if retry also fails (not cached)
  - On other HTTP errors: logs error with status code, returns `{}`
  - On network exception: logs `network error: {e}`, returns `{}`
- `fetch_tmdb_season_episodes(tmdb_id: str, season_number: int) -> dict` — public wrapper
  - Delegates to `_fetch_tmdb_season_cached`
  - On empty result (transient error): calls `_fetch_tmdb_season_cached.cache_clear()` then returns `{}`
  - On `{"_404": True}` sentinel: returns `{}` to caller (404 stays cached for future calls)
  - On successful result: returns full season dict (caller uses `.get('episodes', [])`)
- Both functions inserted after `fetch_tmdb_detail`, before `fetch_duckduckgo` (lines ~978–1035 after edit)
- `lru_cache` already imported at line 58 — no duplicate import added
- `import time` is inline inside the 429 branch only (edge case, not top-level)

---

## Deviations from Plan

None — plan executed exactly as written.

---

## Verification Results

All acceptance criteria met:

| Check | Result |
|-------|--------|
| `result["_num_seasons"]` present | PASS (line 970) |
| `result["_seasons"]` present | PASS (line 971) |
| `if media_type != "movie":` guard | PASS (line 969) |
| Original single-line return gone | PASS |
| `def _fetch_tmdb_season_cached` | PASS (line 979) |
| `def fetch_tmdb_season_episodes` | PASS (line 1023) |
| `@lru_cache(maxsize=50)` before private function | PASS (line 978) |
| `return {"_404": True}` sentinel | PASS (line 997) |
| `resp.status_code == 429` with `time.sleep(1)` retry | PASS (line 998) |
| `_fetch_tmdb_season_cached.cache_clear()` in public wrapper | PASS (line 1031) |
| `log.warning` for 404 | PASS |
| `log.warning` for 429 | PASS |
| `log.error` for non-200/404/429 | PASS |
| `log.error` for network exceptions | PASS |
| Both functions after fetch_tmdb_detail, before fetch_duckduckgo | PASS |
| Symbol count (_num_seasons, _seasons, fetch_tmdb_season_episodes, _fetch_tmdb_season_cached, cache_clear) | PASS (15 matches) |
| Error handling count (status_code == 404, 429, network error) | PASS (3 matches) |
| Python syntax check | PASS (syntax OK) |

---

## Threat Mitigations Applied

| Threat ID | Mitigation Applied |
|-----------|-------------------|
| T-02-02 | Two-function split: transient `{}` triggers `cache_clear()` in public wrapper; only real 200 responses and `{"_404": True}` sentinels are cached — prevents DoS from stale transient-error cache entries |
| T-02-04 | One-time 1s sleep + retry on 429; if retry also fails, returns `{}` (not cached) so the next session call retries — prevents rate-limit cascading |

T-02-01 accepted (API key in URL query string — existing codebase pattern, desktop-local key).
T-02-03 accepted (`resp.json()` on bad JSON raises ValueError, caught by `except Exception as e: log.error(...)` returning `{}`).

---

## Known Stubs

None — this is a pure API layer plan. No UI stubs, no placeholder data flows. Both functions return real data or `{}`.

---

## Threat Flags

None — no new network endpoints introduced beyond the existing TMDb API pattern. The new `/tv/{id}/season/{n}` endpoint follows the same request pattern as the existing `/tv/{id}` endpoint already in `fetch_tmdb_detail`.

---

## Self-Check: PASSED

- `d:/VideoClub/Videoclub` — EXISTS and modified
- Commit `5806626` — EXISTS (`feat(01-02): extend fetch_tmdb_detail return dict with _num_seasons and _seasons`)
- Commit `653b911` — EXISTS (`feat(01-02): add fetch_tmdb_season_episodes with lru_cache and error handling`)
- Python syntax check: `syntax OK`
- All grep counts >= 1
