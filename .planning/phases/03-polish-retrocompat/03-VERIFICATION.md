---
phase: 03-polish-retrocompat
verified: 2026-04-18T22:00:00Z
status: human_needed
score: 8/9 must-haves verified
overrides_applied: 0
human_verification:
  - test: "Open DetailWindow for a series with tmdb_id NULL and hover the disabled button"
    expected: "Tooltip 'Re-añade la serie via búsqueda para activar' appears on hover; clicking does nothing"
    why_human: "Tkinter Tooltip hover behavior cannot be triggered programmatically without a running event loop"
  - test: "Open DetailWindow for a series with a valid tmdb_id"
    expected: "Enabled red 'Ver temporadas' button visible and clickable (opens SeasonViewWindow)"
    why_human: "Requires running Tkinter app and a live series row with tmdb_id in DB"
  - test: "Add watched episodes via SeasonViewWindow for a series, then close the window"
    expected: "Series card in grid immediately shows green 'N ep' badge without manual refresh"
    why_human: "Requires full UI flow: mark episodes, close window, observe grid update"
---

# Phase 3: Polish + Retrocompat Verification Report

**Phase Goal:** Series cards in the main grid show episode progress at a glance, and series added before v7 (without tmdb_id) have a gracefully degraded "Ver temporadas" button with a helpful tooltip rather than a broken state.
**Verified:** 2026-04-18T22:00:00Z
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | A series card with watched_count >= 1 shows a green "N ep" badge at bottom-left of poster | VERIFIED | Lines 1417-1427: type guard + count guard + `text=f"{watched_count} ep"` + `.place(x=2, y=POSTER_H-18)` |
| 2 | A series card with watched_count == 0 shows no badge | VERIFIED | Line 1420: `if watched_count >= 1:` — no label created when 0 |
| 3 | A movie card never shows the badge regardless of any data | VERIFIED | Line 1418: `if self.item["type"] == "series":` outer guard blocks movies |
| 4 | Badge updates automatically after SeasonViewWindow closes (on_refresh chain triggers rebuild) | VERIFIED | SeasonViewWindow._on_close (line 1849-1852) calls `self.on_refresh()` which triggers grid rebuild; ContentCard._build re-runs and re-queries `get_total_watched_count` |
| 5 | Opening DetailWindow for a series with tmdb_id NULL shows a disabled "Ver temporadas" button | VERIFIED | Lines 1596-1616: `if itype == "series":` outer guard, `else:` branch sets `state="disabled"`, `bg=C["surface"]`, `fg=C["muted"]` |
| 6 | Hovering the disabled button shows tooltip "Re-añade la serie via búsqueda para activar" | ? HUMAN | Code confirmed at line 1616: `Tooltip(btn, "Re-añade la serie via búsqueda para activar")` — hover behavior requires running app |
| 7 | Clicking the disabled button does nothing (no crash, no dialog) | ? HUMAN | `state="disabled"` and no `command=` confirmed at lines 1609-1614 — runtime behavior needs human |
| 8 | Opening DetailWindow for a series with valid tmdb_id still shows the enabled (red) button | ? HUMAN | Enabled branch at lines 1598-1607 is structurally identical to original — runtime needs human to confirm |
| 9 | Movies still show no "Ver temporadas" button | VERIFIED | Outer `if itype == "series":` at line 1597 — movies never enter this block |

**Score:** 6/9 truths fully verified programmatically; 3 require human runtime confirmation. Code structure supports all 9 truths.

### Roadmap Success Criteria

| # | Success Criterion | Status | Evidence |
|---|------------------|--------|----------|
| SC1 | Series card with >= 1 watched episode shows progress badge without opening any window | VERIFIED | Badge block at lines 1417-1427 reads live DB via `get_total_watched_count` |
| SC2 | Series with tmdb_id=NULL shows disabled "Ver temporadas" with tooltip explaining why | VERIFIED (code) / ? HUMAN (runtime) | Lines 1608-1616: disabled branch + Tooltip present |
| SC3 | ttk.Notebook tabs in SeasonViewWindow use dark theme palette | VERIFIED | Lines 1892-1902: `Dark.TNotebook` style configured with `C["bg"]`, `C["surface"]`, `C["muted"]`, `C["text"]` — implemented in Phase 2, confirmed present |

Note: SC3 was implemented by Phase 2 (commit 8b0f08c) and no Phase 3 plan claimed it. It is present and satisfied. No gap.

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `Videoclub` (line 429) | `Database.get_total_watched_count` method | VERIFIED | Exists at line 429; SELECT COUNT(*) FROM episode_progress WHERE user_id=? AND item_id=? AND watched=1; returns int |
| `Videoclub` (lines 1417-1427) | ContentCard._build badge rendering block | VERIFIED | Full block present with both type and count guards, correct placement |
| `Videoclub` (lines 1608-1616) | DetailWindow._build disabled button branch | VERIFIED | state=disabled, bg=C["surface"], fg=C["muted"], no cursor/command, Tooltip attached |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `ContentCard._build` | `db.get_total_watched_count` | `self.db.get_total_watched_count(self.user_id, self.item["id"])` | WIRED | Line 1419 — exact call pattern confirmed |
| `SeasonViewWindow._on_close` | `ContentCard._build` | `on_refresh()` triggers grid rebuild which re-runs `_build()` | WIRED | Line 1851-1852: `on_refresh()` called; ContentCard._build queries DB fresh on each build |
| `DetailWindow._build` | `Tooltip` class | `Tooltip(btn, "Re-añade la serie via búsqueda para activar")` | WIRED | Line 1616 — Tooltip instantiated with btn variable (not anonymous) |

### Data-Flow Trace (Level 4)

| Artifact | Data Variable | Source | Produces Real Data | Status |
|----------|--------------|--------|-------------------|--------|
| `ContentCard._build` badge | `watched_count` | `self.db.get_total_watched_count(user_id, item_id)` → `SELECT COUNT(*) FROM episode_progress WHERE watched=1` | Yes — live DB query | FLOWING |
| `DetailWindow._build` disabled branch | `tmdb_id` | `self.item.get("tmdb_id") or ""` — from DB-loaded item dict | Yes — reads actual item data | FLOWING |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| `get_total_watched_count` method exists with correct signature | `grep -n "def get_total_watched_count"` | Line 429: 1 match | PASS |
| Badge uses correct placement formula | `grep -n "POSTER_H - 18"` | Line 1427: 1 match | PASS |
| Badge text format exactly "N ep" | `grep -n 'f"{watched_count} ep"'` | Line 1423: 1 match | PASS |
| Old single-branch form removed | `grep "if itype == .series. and tmdb_id"` | 0 matches | PASS |
| New outer guard present | `grep 'if itype == "series":'` | Line 1597: 1 match | PASS |
| Disabled button tooltip text exact match | `grep "Re-añade la serie via búsqueda para activar"` | Line 1616: 1 match | PASS |
| File parses without syntax errors | `python ast.parse` (utf-8) | syntax OK | PASS |
| Commits documented in summaries exist | `git log --oneline` | 6384ead and 554ef27 confirmed | PASS |

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|------------|------------|-------------|--------|----------|
| GRID-01 | 03-01-PLAN.md | Series cards with watched episodes show progress badge | SATISFIED | Badge block at lines 1417-1427 reads live data from `episode_progress` table |
| BACK-01 | 03-02-PLAN.md | Legacy series (tmdb_id=NULL) show disabled "Ver temporadas" with tooltip | SATISFIED | Disabled branch at lines 1608-1616 with Tooltip |
| BACK-02 | 03-02-PLAN.md | Clicking disabled button suggests re-adding series via search | SATISFIED | Tooltip text at line 1616 communicates this; `state="disabled"` blocks all click side-effects |

All 3 phase requirements (GRID-01, BACK-01, BACK-02) claimed in plan frontmatter are present in REQUIREMENTS.md and satisfied in code. No orphaned requirements.

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| None | — | — | — | No TODO/FIXME/placeholder/stub patterns found in modified blocks |

Scanned: badge block (lines 1417-1427), DB method (lines 429-436), disabled button branch (lines 1596-1616). All implementations are complete with live data flows.

### Human Verification Required

#### 1. Disabled Button Tooltip Hover

**Test:** Run the app, add a series manually (or find one in DB without tmdb_id), open its DetailWindow.
**Expected:** The "Ver temporadas" button appears in muted/surface colors. Hovering over it shows a tooltip: "Re-añade la serie via búsqueda para activar". Clicking does nothing — no crash, no dialog, no navigation.
**Why human:** Tkinter Tooltip uses `<Enter>`/`<Leave>` bindings on a real widget; cannot simulate without a running Tk event loop.

#### 2. Enabled Button Regression Check

**Test:** Open DetailWindow for a series with a valid tmdb_id in the DB.
**Expected:** Red "Ver temporadas" button is visible and clicking it opens SeasonViewWindow as a modal.
**Why human:** Requires a running app with a seeded series row; confirms the enabled branch was not disturbed by the split.

#### 3. Badge Refresh After SeasonViewWindow Close

**Test:** Mark one or more episodes as watched in SeasonViewWindow for a series with 0 prior watched episodes, then close SeasonViewWindow.
**Expected:** The series card in the main grid immediately shows the green "N ep" badge with the correct count — no F5 or manual refresh needed.
**Why human:** Requires full UI flow: open SeasonViewWindow, toggle checkboxes, close window, observe grid state update.

---

## Gaps Summary

No structural gaps. All code artifacts exist, are substantive, and are wired with real data flows. File parses cleanly. Both commits confirmed in git history.

The 3 human verification items are runtime behavior checks that cannot be automated without a running Tkinter event loop. They test tooltip hover, disabled button click-safety, and the on_refresh chain — all of which are structurally sound in the code.

---

_Verified: 2026-04-18T22:00:00Z_
_Verifier: Claude (gsd-verifier)_
