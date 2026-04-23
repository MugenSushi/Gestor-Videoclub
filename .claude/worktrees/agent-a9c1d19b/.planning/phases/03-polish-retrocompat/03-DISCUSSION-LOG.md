# Phase 3: Polish + Retrocompat - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-18
**Phase:** 03-polish-retrocompat
**Areas discussed:** Badge format (GRID-01), Disabled button UX (BACK-01/02)

---

## Badge format (GRID-01)

| Option | Description | Selected |
|--------|-------------|----------|
| Total watched count | e.g. "4 ep" — COUNT(watched=1) from episode_progress | ✓ |
| Watched / loaded ratio | e.g. "4/18" — watched out of loaded episodes | |
| Last position T2E5 | e.g. "T2E5" — last watched episode via MAX(updated_at) | |

**User's choice:** Total watched count

---

### Badge text

| Option | Description | Selected |
|--------|-------------|----------|
| "4 ep" | Short, fits card width | ✓ |
| "4 eps vistos" | More explicit but may be too long | |
| "4" | Minimal — number only | |

**User's choice:** "4 ep"

---

### Badge trigger

| Option | Description | Selected |
|--------|-------------|----------|
| Any watched episode (≥1) | Show as soon as watched_count ≥ 1 | ✓ |
| Only if ≥1 full season done | More conservative | |

**User's choice:** Any watched episode

---

### Badge refresh

| Option | Description | Selected |
|--------|-------------|----------|
| On grid refresh (on_refresh) | Computed at card build time | ✓ |
| Live while SeasonViewWindow open | Requires ContentCard DB subscription | |

**User's choice:** On grid refresh

---

### Badge placement

| Option | Description | Selected |
|--------|-------------|----------|
| Bottom-left of poster | .place() consistent with existing badges | ✓ |
| Below title, above buttons | Always visible, no overlap | |

**User's choice:** Bottom-left of poster

---

## Disabled button UX (BACK-01/02)

### Button appearance

| Option | Description | Selected |
|--------|-------------|----------|
| Same button, grayed out | state=disabled, bg=C["surface"], fg=C["muted"] | ✓ |
| Hidden, replaced by tooltip label | No button at all | |

**User's choice:** Same button, grayed out

---

### Click behavior (BACK-02)

| Option | Description | Selected |
|--------|-------------|----------|
| Nothing (state=disabled blocks click) | Tooltip explains the situation | ✓ |
| Show a small dialog | messagebox.showinfo with re-add suggestion | |

**User's choice:** Nothing — disabled state is sufficient

---

### Tooltip text

| Option | Description | Selected |
|--------|-------------|----------|
| "Re-añade la serie via búsqueda para activar" | Short, actionable | ✓ |
| "Esta serie no tiene tmdb_id. Re-añádela..." | More explicit, mentions tmdb_id | |

**User's choice:** "Re-añade la serie via búsqueda para activar"

---

## Claude's Discretion

- Exact y-coordinate for badge placement (bottom-left of poster)
- Whether to inline the watched count query or add a named DB method

## Deferred Ideas

- Live badge update while SeasonViewWindow is open
- T2E5 last-position badge format (v2 consideration)
- Click-to-dialog on disabled button
