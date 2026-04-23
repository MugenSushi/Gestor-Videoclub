# Phase 3: Polish + Retrocompat - Context

**Gathered:** 2026-04-18
**Status:** Ready for planning

<domain>
## Phase Boundary

Two polishing requirements:
1. **GRID-01** — Add an episode progress badge to `ContentCard` for series with watched episodes
2. **BACK-01/02** — Show the "Ver temporadas" button in a disabled state (with tooltip) for series without `tmdb_id`, instead of hiding it

The `ttk.Notebook` dark theme (success criterion 3) was fully implemented in Phase 2 (`_build_notebook` applies `Dark.TNotebook` style). No additional work needed there.

Scope: GRID-01, BACK-01, BACK-02.

</domain>

<decisions>
## Implementation Decisions

### GRID-01: Progress Badge

- **D-01**: Badge text format: `"N ep"` (e.g. `"4 ep"`). Short, fits narrow card width.

- **D-02**: Badge trigger: show when `watched_count >= 1`. Hide (don't render) when count is 0.

- **D-03**: Badge placement: bottom-left of `poster_zone` using `.place()`, consistent with existing ✓ VISTO and ⏳ PDTE badges. Coordinates: `x=2, y=POSTER_H - 18` (or similar bottom-left position that avoids overlap with top-left status badges).

- **D-04**: Badge color: `bg=C["seen"]` (green), `fg="white"`, `font=(FONT_FAMILY, 7, "bold")`. Consistent with existing badge style.

- **D-05**: Badge data source: `COUNT(watched=1)` across all seasons for the item — needs a new DB method `get_total_watched_count(user_id, item_id) -> int` (or equivalent inline query). `get_season_progress` is per-season only and not usable here without knowing the seasons list.

- **D-06**: Badge refresh: computed when `ContentCard._build()` runs. `SeasonViewWindow._on_close` already calls `on_refresh()`, which rebuilds cards — badge updates automatically without any extra wiring.

- **D-07**: Only show badge for `type == "series"` items. Movie cards never get a progress badge.

### BACK-01/02: Disabled "Ver temporadas" Button

- **D-08**: When `itype == "series"` and `tmdb_id` is falsy (NULL or empty string), render the "📺 Ver temporadas" button in a **disabled state** instead of hiding it:
  - `state="disabled"`, `bg=C["surface"]`, `fg=C["muted"]`
  - No `cursor="hand2"` (disabled state removes pointer cursor automatically)

- **D-09**: Attach a `Tooltip` to the disabled button with text:
  `"Re-añade la serie via búsqueda para activar"`

- **D-10**: Click behavior: `state="disabled"` blocks all click events — no additional handler needed. The tooltip already explains the situation. No dialog or toast.

- **D-11**: The existing `if itype == "series" and tmdb_id:` block in `DetailWindow._build()` becomes:
  ```python
  if itype == "series":
      if tmdb_id:
          # enabled button (existing code)
      else:
          # disabled button + tooltip (new code)
  ```

### Dark Theme (already complete)

- **D-12**: `Dark.TNotebook` style (`_build_notebook` in `SeasonViewWindow`) was applied in Phase 2. Success criterion 3 is already satisfied. No action needed in Phase 3.

### Claude's Discretion

- Exact y-coordinate for badge placement (bottom-left of poster) — pick a value that doesn't overlap with status badges
- Whether to inline the watched count query or add a named DB method

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements
- `.planning/REQUIREMENTS.md` — GRID-01, BACK-01, BACK-02

### Phase Goal & Success Criteria
- `.planning/ROADMAP.md` — Phase 3 success criteria (3 criteria)

### Codebase Integration Points
- `Videoclub` lines 1358–1487 — `ContentCard` class (badge placement target)
- `Videoclub` lines 1280–1312 — `Tooltip` class (reuse for disabled button tooltip)
- `Videoclub` lines 1553–1589 — `DetailWindow._build()` "Ver temporadas" button (retrocompat target)
- `Videoclub` lines 415–427 — `db.get_season_progress()` (existing per-season method — not sufficient for GRID-01 alone)

### Prior Phase Context
- `.planning/phases/02-seasonviewwindow/02-CONTEXT.md` — D-03, D-13 (modal stack, button placement)

No external specs — requirements fully captured in decisions above.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `Tooltip(widget, text)` (line ~1280) — already used on `title_lbl` in ContentCard; reuse directly for disabled button
- `.place()` badge pattern (lines 1393–1406) — ✓ VISTO at `x=2, y=2`; type badge at `x=CARD_W-24, y=2`; new badge goes bottom-left
- `C["seen"]` green, `C["surface"]` dark grey, `C["muted"]` muted text — established color tokens

### Established Patterns
- Badges on poster: `tk.Label(poster_zone, ...).place(x=N, y=N)` — no pack/grid
- Disabled buttons: `state="disabled"` with `bg=C["surface"]`, `fg=C["muted"]` (existing pattern in search dialog)
- `on_refresh` already called by `SeasonViewWindow._on_close` → grid rebuilds automatically

### Integration Points
- `ContentCard._build()` — add watched count query + conditional badge in poster_zone
- `DetailWindow._build()` — extend the `if itype == "series" and tmdb_id:` block to also handle `not tmdb_id` case
- `Database` class — may need `get_total_watched_count(user_id, item_id) -> int` or inline SQL

</code_context>

<specifics>
## Specific Ideas

- Badge text exactly `"N ep"` (not "N eps", not "N vistos") — short form fits card
- Bottom-left placement for progress badge, same `.place()` style as existing badges
- Disabled button tooltip: `"Re-añade la serie via búsqueda para activar"`

</specifics>

<deferred>
## Deferred Ideas

- Live badge update while SeasonViewWindow is open (requires ContentCard DB subscription — out of scope v1)
- Badge showing last position (T2E5) instead of total count — revisit in v2 if users want it
- "Ver temporadas" click on disabled button showing a dialog — decided against; tooltip is sufficient

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 03-polish-retrocompat*
*Context gathered: 2026-04-18*
