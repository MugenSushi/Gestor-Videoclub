# Phase 2: SeasonViewWindow - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-17
**Phase:** 02-seasonviewwindow
**Areas discussed:** Seasons list initialization

---

## Seasons list initialization

| Option | Description | Selected |
|--------|-------------|----------|
| Async spinner-first | Show 'Cargando temporadas...' spinner, fetch in background pool, build tabs when done. Never blocks UI. Consistent with per-tab lazy loading. | ✓ |
| Synchronous on open | Call fetch_tmdb_detail in __init__ before building tabs. Simpler code. May block UI ~300-800ms for cold fetch. | |

**User's choice:** Async spinner-first
**Notes:** Consistent with the per-tab lazy loading pattern already in UI-SPEC.

---

## Error fallback (seasons list fetch failure)

| Option | Description | Selected |
|--------|-------------|----------|
| Error message + close button | Show 'No se pudo obtener las temporadas. Comprueba tu conexión.' + 'Cerrar' button. | ✓ |
| Fallback to num_seasons integer | Build N generic tabs (T1, T2...) from item['num_seasons'] without names. | |

**User's choice:** Error message + close button
**Notes:** No fallback to generic tabs — honest error state preferred.

---

## Claude's Discretion

- "Marcar toda la temporada" skip-future behavior: skip episodes with air_date > today
- Constructor signature: same shape as DetailWindow and RefreshMetaDialog
- File insertion point: after DetailWindow class (~line 1748)
- Worker pool: dedicated `_season_pool` (ThreadPoolExecutor, max_workers=2)

## Deferred Ideas

- Window size memory between opens — v1 always opens at 600×480 minimum
- "Marcar toda" ↔ "Desmarcar" toggle — out of scope v1
- Disabled button for tmdb_id=NULL series — Phase 3
