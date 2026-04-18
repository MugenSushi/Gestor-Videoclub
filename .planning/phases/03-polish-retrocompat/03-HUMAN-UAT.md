---
status: partial
phase: 03-polish-retrocompat
source: [03-VERIFICATION.md]
started: 2026-04-18T21:40:00Z
updated: 2026-04-18T21:40:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Tooltip hover on disabled button
expected: Hovering the disabled "📺 Ver temporadas" button on a legacy series (no tmdb_id) shows tooltip "Re-añade la serie via búsqueda para activar"
result: [pending]

### 2. Enabled button regression
expected: Opening DetailWindow for a series with a valid tmdb_id still shows the enabled (red) "📺 Ver temporadas" button — clicking it opens SeasonViewWindow normally
result: [pending]

### 3. Badge refresh after SeasonViewWindow close
expected: After closing SeasonViewWindow (marking episodes watched), the series card in the grid updates its "N ep" badge automatically without manual refresh
result: [pending]

## Summary

total: 3
passed: 0
issues: 0
pending: 3
skipped: 0
blocked: 0

## Gaps
