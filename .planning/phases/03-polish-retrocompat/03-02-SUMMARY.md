---
phase: 03-polish-retrocompat
plan: "02"
subsystem: detail-window-retrocompat
tags: [ui, retrocompat, tooltip, disabled-button, DetailWindow]
dependency_graph:
  requires: [Ver temporadas button (Phase 2), Tooltip class (existing), C["surface"]/C["muted"] palette]
  provides: [DetailWindow disabled button branch, graceful degradation for legacy series]
  affects: [DetailWindow._build for series items without tmdb_id]
tech_stack:
  added: []
  patterns: [Tkinter state=disabled, Tooltip hover widget, conditional branch split]
key_files:
  created: []
  modified:
    - Videoclub
decisions:
  - "Disabled branch uses bg=C['surface'], fg=C['muted'] — matches muted/inactive visual language, not accent red"
  - "No cursor='hand2' on disabled button — state=disabled owns pointer behavior per UI-SPEC guard 4"
  - "No command= on disabled button — state=disabled blocks clicks; .invoke() path also guarded per UI-SPEC guard 5"
  - "btn variable assigned before Tooltip() call — anonymous widget cannot be referenced per UI-SPEC guard 6"
metrics:
  duration: "~3 minutes"
  completed: "2026-04-18T21:30:00Z"
  tasks_completed: 1
  files_modified: 1
---

# Phase 03 Plan 02: Retrocompat Disabled Button Summary

**One-liner:** Split DetailWindow._build "Ver temporadas" into enabled/disabled branches; legacy series (tmdb_id NULL) get a surface-colored disabled button with hover Tooltip explaining how to re-add via search.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Split DetailWindow._build "Ver temporadas" into enabled/disabled branches | 554ef27 | Videoclub (lines 1596-1616) |

## What Was Built

### Task 1: Enabled/disabled branch split

Replaced `if itype == "series" and tmdb_id:` single branch with a two-level guard:

- Outer: `if itype == "series":` — movies still get no button (unchanged)
- Inner enabled (`if tmdb_id:`): character-for-character identical to original button (D-11 compliance)
- Inner disabled (`else:`):
  - `tk.Button` with `state="disabled"`, `bg=C["surface"]`, `fg=C["muted"]`, `font=(FONT_FAMILY, 9)`, `relief="flat"`
  - No `cursor="hand2"`, no `command=` (UI-SPEC guards 4 and 5)
  - `btn` variable holds reference before `.pack()` call
  - `Tooltip(btn, "Re-añade la serie via búsqueda para activar")` applied after pack (UI-SPEC guard 6)

## Verification Results

```
grep "Re-añade la serie via búsqueda para activar" → line 1616 (1 match)
grep 'state="disabled"'                            → line 1613 (in DetailWindow._build)
grep "if itype == .series. and tmdb_id"            → 0 matches (old form gone)
grep 'if itype == "series":'                       → line 1597 (1 match, new outer guard)
python ast.parse                                   → syntax OK
```

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None. Disabled button reads live `tmdb_id` from `self.item` dict; Tooltip text is static and intentional.

## Threat Flags

None. No new network endpoints, auth paths, file access patterns, or schema changes introduced.

## Self-Check: PASSED

- Videoclub modified: confirmed (1 file changed, 19 insertions, 9 deletions)
- Commit 554ef27 exists: confirmed
- Syntax: clean (ast.parse exits 0)
- All verification grep patterns: matched
- Old `and tmdb_id` form removed: confirmed (0 matches)
