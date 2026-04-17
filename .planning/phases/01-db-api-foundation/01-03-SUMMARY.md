---
phase: 01-db-api-foundation
plan: "03"
subsystem: database-wiring
tags: [tmdb, add_item, SmartSearchDialog, episode-tracking, data-flow]
dependency_graph:
  requires: [schema-v7, fetch-tmdb-season-episodes, tmdb-season-api-layer]
  provides: [tmdb-id-persistence, num-seasons-persistence, end-to-end-tmdb-wiring]
  affects: [add_item, SmartSearchDialog._add_result, SmartSearchDialog._finish_manual, SmartSearchDialog._add_manual]
tech_stack:
  added: []
  patterns: [keyword-arg ordering (tmdb_id before user_id), fetch_tmdb_multi _tmdb_id injection, defensive-or-defaults]
key_files:
  created: []
  modified:
    - d:/VideoClub/Videoclub
decisions:
  - "tmdb_id and num_seasons placed before user_id in add_item signature — item-level fields logically precede user-level fields, and safe defaults (\"\" / 0) preserve backward compat for all existing callers"
  - "_add_manual._worker injects _tmdb_id via fetch_tmdb_multi(type_hint=\"series\") because smart_search/fetch_tmdb (single-result path) does not return _tmdb_id — fetch_tmdb_multi does"
  - "**extra remains last keyword arg in both call sites to avoid SyntaxError (keyword argument follows **kwargs)"
  - "BatchImportDialog._worker left unchanged — add_item defaults cover retrocompat series; tmdb_id backfill deferred to Phase 3 (BACK-01/02)"
metrics:
  duration: "~60 seconds"
  completed: "2026-04-17"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 1
requirements: [DB-05]
---

# Phase 01 Plan 03: tmdb_id + num_seasons Wiring Summary

**One-liner:** add_item() extended with tmdb_id/num_seasons params and INSERT, SmartSearchDialog call sites wired to pass both values, _add_manual._worker injects _tmdb_id via fetch_tmdb_multi for series — completing the end-to-end Phase 1 data flow.

---

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Extend Database.add_item signature and INSERT statement | 360e41d | d:/VideoClub/Videoclub |
| 2 | Update SmartSearchDialog call sites to pass tmdb_id and num_seasons | 2532a29 | d:/VideoClub/Videoclub |

---

## What Was Built

### Task 1: Database.add_item signature and INSERT extension

- Added `tmdb_id: str = ""` and `num_seasons: int = 0` to `add_item` signature, placed before `user_id: int = 0` (item-level fields logically before user-level)
- Extended INSERT column list: `(title,type,rating,genres,year,plot,poster_url,director,actors,studio,streaming,tmdb_id,num_seasons)`
- Updated VALUES tuple from 11 to 13 placeholders with defensive defaults: `tmdb_id or ""` and `num_seasons or 0`
- All existing callers (BatchImportDialog) unaffected — safe defaults keep them working without modification

### Task 2: SmartSearchDialog call site wiring

**Edit 1 — `_add_result` call site (line ~2111):**
- Added `tmdb_id=data.get("_tmdb_id", "")` and `num_seasons=data.get("_num_seasons", 0)` before `user_id=uid`
- `data` already contains `_tmdb_id` (extracted at line 2082) and `_num_seasons` (populated by `data.update(fetch_tmdb_detail(...))` for TV series via Plan 02 extension)
- `**extra` kept last to avoid SyntaxError

**Edit 2 — `_add_manual._worker` injection (line ~2146):**
- Added conditional block: if `itype_forced == "series"` AND `data.get("_source") == "tmdb"` AND TMDb API key present → call `fetch_tmdb_multi(title, type_hint="series")` and inject `candidates[0].get("_tmdb_id", "")` into `data["_tmdb_id"]`
- This resolves the gap where `smart_search`/`fetch_tmdb` (single-result path) does NOT return `_tmdb_id` in its dict, while `fetch_tmdb_multi` does

**Edit 3 — `_finish_manual` call site (line ~2178):**
- Identical pattern to `_add_result`: added `tmdb_id=data.get("_tmdb_id", "")` and `num_seasons=data.get("_num_seasons", 0)` before `user_id=self.app_root.active_user_id`
- `_num_seasons` is populated in `data` when `_finish_manual` calls `fetch_tmdb_detail` during enrichment (same path as `_add_result`)
- `**extra` kept last

**BatchImportDialog._worker:** NOT modified — intentional retrocompat design; Phase 3 (BACK-01/02) handles tmdb_id backfill for pre-v7 series.

---

## Deviations from Plan

None — plan executed exactly as written.

---

## Verification Results

All acceptance criteria met:

| Check | Result |
|-------|--------|
| `tmdb_id: str = ""` in add_item signature | PASS (line 362) |
| `num_seasons: int = 0` in add_item signature | PASS (line 362) |
| `user_id: int = 0` still last in signature | PASS |
| `tmdb_id,num_seasons` in INSERT column list | PASS (line 368) |
| `VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?)` (13 placeholders) | PASS (line 369) |
| `tmdb_id or "", num_seasons or 0` in values tuple | PASS (line 372) |
| `tmdb_id=data.get("_tmdb_id", "")` at >= 2 call sites | PASS (lines 2120, 2202) |
| `num_seasons=data.get("_num_seasons", 0)` at >= 2 call sites | PASS (lines 2121, 2203) |
| `fetch_tmdb_multi(title, type_hint="series")` in _worker | PASS (line 2163) |
| `data["_tmdb_id"] = candidates[0].get("_tmdb_id", "")` | PASS (line 2165) |
| BatchImportDialog._worker has NO `tmdb_id=` keyword arg | PASS |
| Python syntax check | PASS (syntax OK) |

---

## Threat Mitigations Applied

| Threat ID | Mitigation Applied |
|-----------|-------------------|
| T-03-03 | `**extra` kept as last keyword arg in both `_add_result` and `_finish_manual` call sites — verified by Python `ast.parse` syntax check |
| T-03-04 | BatchImportDialog._worker explicitly NOT modified — verified by grep showing no `tmdb_id=` at lines 2489–2496 |

T-03-01 accepted (first-result heuristic from fetch_tmdb_multi — user chose the title; wrong tmdb_id = wrong season data in Phase 2, not a security issue).
T-03-02 accepted (desktop app, local SQLite, no multi-user exposure in Phase 1).

---

## Known Stubs

None — this plan wires real data through existing functions. tmdb_id and num_seasons will be populated from actual TMDb API responses at add time. No hardcoded values or placeholder data flows.

---

## Threat Flags

None — no new network endpoints or auth paths introduced. The `fetch_tmdb_multi` call in `_worker` follows the existing pattern already used in `SmartSearchDialog._do_search`.

---

## Self-Check: PASSED

- `d:/VideoClub/Videoclub` — EXISTS and modified
- Commit `360e41d` — EXISTS (`feat(01-03): extend add_item signature and INSERT with tmdb_id and num_seasons`)
- Commit `2532a29` — EXISTS (`feat(01-03): wire tmdb_id and num_seasons through SmartSearchDialog call sites`)
- Python syntax check: `syntax OK`
- All grep counts verified
