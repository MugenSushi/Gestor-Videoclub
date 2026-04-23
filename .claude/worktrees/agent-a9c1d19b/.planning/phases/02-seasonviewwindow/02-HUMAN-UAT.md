---
status: partial
phase: 02-seasonviewwindow
source: [02-VERIFICATION.md]
started: 2026-04-18T00:00:00Z
updated: 2026-04-18T00:00:00Z
---

## Current Test

[awaiting human testing]

## Tests

### 1. Modal grab_set chain
expected: DetailWindow is unresponsive (grayed out / no clicks) while SeasonViewWindow is open; closing SeasonViewWindow returns focus and interaction to DetailWindow
result: [pending]

### 2. Lazy tab re-fetch guard
expected: switching back to an already-loaded tab shows no spinner and makes no duplicate network call; only unvisited tabs trigger a fetch
result: [pending]

### 3. SQLite write on toggle
expected: checking or unchecking an episode checkbox writes a row to `episode_progress` in SQLite immediately — verify in a DB browser or log; no explicit save button needed
result: [pending]

### 4. Counter accuracy and green flip
expected: "X / Y episodios vistos" counter updates the instant a checkbox is toggled; turns C["seen"] green color when X == Y
result: [pending]

### 5. Future episode disabled state
expected: episode rows with air_date in the future render in a muted/grey color and their checkboxes are non-interactive (cannot be clicked)
result: [pending]

### 6. "Marcar toda" skips future episodes
expected: clicking "Marcar toda la temporada" marks all non-future episodes watched but leaves future episode checkboxes unticked
result: [pending]

### 7. on_refresh grid update
expected: closing SeasonViewWindow causes the parent grid to refresh (episode counts / progress badges update) without requiring a manual F5 refresh
result: [pending]

## Summary

total: 7
passed: 0
issues: 0
pending: 7
skipped: 0
blocked: 0

## Gaps
