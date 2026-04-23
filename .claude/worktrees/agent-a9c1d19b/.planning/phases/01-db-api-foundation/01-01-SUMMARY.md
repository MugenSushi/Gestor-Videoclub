---
phase: 01-db-api-foundation
plan: "01"
subsystem: database
tags: [sqlite, migration, schema, episode-tracking]
dependency_graph:
  requires: []
  provides: [schema-v7, episode_progress-table, episode-progress-db-methods]
  affects: [database-startup, add_item, series-tracking]
tech_stack:
  added: []
  patterns: [ALTER TABLE idempotent migration, if-v-lt-N gate, ON CONFLICT UPSERT]
key_files:
  created: []
  modified:
    - d:/VideoClub/Videoclub
decisions:
  - "SCHEMA_VERSION bumped to 7 as the first edit to prevent repeated migration execution"
  - "episode_progress uses item_id INTEGER FK to items.id (not tmdb_series_id TEXT) per locked architecture decision"
  - "set_episode_watched uses ON CONFLICT DO UPDATE UPSERT (not INSERT OR REPLACE) to preserve watched_at on updates"
  - "ALTER TABLE loop exception handler changed from bare pass to log.debug for migration observability"
metrics:
  duration: "79 seconds"
  completed: "2026-04-17"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 1
requirements: [DB-01, DB-02, DB-03, DB-04]
---

# Phase 01 Plan 01: DB Schema Migration v6 to v7 Summary

**One-liner:** SQLite schema migrated to v7 with tmdb_id/num_seasons columns on items, episode_progress table with composite unique constraint, and three DB helper methods using ON CONFLICT UPSERT.

---

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Bump SCHEMA_VERSION and extend ALTER TABLE loop | dc13654 | d:/VideoClub/Videoclub |
| 2 | Add v<7 migration gate and episode_progress DB methods | b35003a | d:/VideoClub/Videoclub |

---

## What Was Built

### Task 1: SCHEMA_VERSION bump + ALTER TABLE loop extension

- Changed `SCHEMA_VERSION = 6` to `SCHEMA_VERSION = 7` at line 158 (first edit, preventing repeated migration execution)
- Extended the ALTER TABLE loop with two new nullable column entries:
  - `("tmdb_id", "items", "TEXT")` — stores TMDb series ID for season API calls
  - `("num_seasons", "items", "INTEGER")` — stores series season count from TMDb detail response
- Replaced `except Exception: pass` with `log.debug(f"_migrate: {col} on {tbl} already exists or error: {e}")` for migration observability (mitigates T-01-03)

### Task 2: v<7 migration gate + DB methods

- Inserted `if v < 7:` block between the `if v < 6:` block and `if v < SCHEMA_VERSION:` gate
- Block creates `idx_items_tmdb_id` index on `items(tmdb_id)` with `log.error` on failure
- Block creates `episode_progress` table with:
  - `id INTEGER PRIMARY KEY AUTOINCREMENT` surrogate PK
  - `user_id INTEGER NOT NULL` FK to `users(id)`
  - `item_id INTEGER NOT NULL` FK to `items(id)` (NOT tmdb_series_id — locked decision)
  - `UNIQUE (user_id, item_id, season_number, episode_number)` composite constraint
  - `idx_ep_lookup` index on `(user_id, item_id, season_number)`
- Added three DB helper methods to the `Database` class:
  - `get_episode_progress(user_id, item_id, season_number) -> dict` — returns `{episode_number: watched}` map
  - `set_episode_watched(user_id, item_id, season_number, episode_number, watched) -> bool` — uses `ON CONFLICT DO UPDATE` UPSERT
  - `get_season_progress(user_id, item_id, season_number) -> tuple` — returns `(watched_count, total_count)`

---

## Deviations from Plan

None — plan executed exactly as written.

---

## Verification Results

All acceptance criteria met:

| Check | Result |
|-------|--------|
| `SCHEMA_VERSION = 7` present | PASS (count: 1) |
| `tmdb_id` in ALTER TABLE loop | PASS (count: 11 occurrences total) |
| `num_seasons` in ALTER TABLE loop | PASS (count: 1) |
| `log.debug` replaces bare `pass` | PASS |
| `if v < 7:` block exists | PASS (line 319) |
| `CREATE TABLE IF NOT EXISTS episode_progress` | PASS |
| `item_id INTEGER NOT NULL` (not tmdb_series_id) | PASS |
| `UNIQUE (user_id, item_id, season_number, episode_number)` | PASS |
| `CREATE INDEX IF NOT EXISTS idx_ep_lookup` | PASS |
| `def get_episode_progress` | PASS (line 383) |
| `def set_episode_watched` | PASS (line 393) |
| `def get_season_progress` | PASS (line 411) |
| `ON CONFLICT(user_id, item_id, season_number, episode_number)` | PASS |
| Python syntax check | PASS (syntax OK) |

---

## Threat Mitigations Applied

| Threat ID | Mitigation Applied |
|-----------|-------------------|
| T-01-01 | Nullable-only columns (TEXT, INTEGER without NOT NULL); `IF NOT EXISTS` guards on CREATE INDEX |
| T-01-02 | `try/except` with `log.error` in v<7 gate — failures are visible and do not silently corrupt DB |
| T-01-03 | `except Exception: pass` replaced with `log.debug(...)` in ALTER TABLE loop |

T-01-04 accepted (single-user desktop app, no privilege boundary in Phase 1).

---

## Known Stubs

None — this is a pure schema/DB layer plan. No UI stubs, no placeholder data flows.

---

## Threat Flags

None — no new network endpoints, auth paths, or file access patterns introduced. All changes are local SQLite migrations within the existing `Database` class.

---

## Self-Check: PASSED

- `d:/VideoClub/Videoclub` — EXISTS and modified
- Commit `dc13654` — EXISTS (`feat(01-01): bump SCHEMA_VERSION to 7 and extend ALTER TABLE loop`)
- Commit `b35003a` — EXISTS (`feat(01-01): add v<7 migration gate and episode_progress DB methods`)
- Python syntax check: `syntax OK`
- All grep counts >= 1
