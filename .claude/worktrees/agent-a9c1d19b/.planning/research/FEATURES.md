# Feature Landscape: Season/Episode Tracking View

**Domain:** Personal TV series episode tracking — desktop Season View window
**Project:** Videoclub STARTUP v7.0 (Python/Tkinter)
**Researched:** 2026-04-14
**Confidence:** MEDIUM — based on deep training-data knowledge of Trakt.tv, TVTime, Episodate,
Serializd, and Plex/Infuse season views. Web verification unavailable in this session.
Patterns are consistent across all known implementations, raising confidence.

---

## Answering the Specific Questions

### 1. Minimum Viable Episode Tracking UI — What Must Be Visible Per Episode

Every credible tracker (Trakt, TVTime, Infuse, Plex) shows exactly this per episode row:

```
[watched indicator]  S01E03 · Episode Title                    air date
```

- **Episode number** (S01E03 or just "3") — orientation anchor, non-negotiable
- **Episode title** — users scan for titles they remember; missing this breaks recall
- **Watched indicator** — the whole point; a checkbox or filled/empty dot
- **Air date** — lets users tell apart "haven't seen it yet" from "not aired yet"

Runtime is secondary (nice if space allows). Synopsis is tertiary (tooltip or expand).

### 2. Standard Interaction for Marking Episodes Watched

**Dominant pattern across all desktop/web trackers:** single click on a checkbox or the
episode row itself toggles watched. No drag, no swipe (those are mobile patterns).

- Trakt.tv: checkbox per episode, clicking anywhere in the row also works
- TVTime (mobile): tap the episode card
- Plex: hover reveals a checkmark button; click toggles
- Infuse: long-press (mobile) / click checkmark (desktop)

**Recommendation for Tkinter:** A `ttk.Checkbutton` or a styled toggle button (Canvas-drawn
circle/checkmark matching the dark theme) on each row. Click = immediate DB write + visual
update. No confirmation dialog — the action is trivially reversible.

### 3. Do Users Mark One Episode at a Time, or "Mark All Up to Here"?

Both behaviors exist and serve different use cases:

**One at a time** — most common for current-season tracking. User watches ep 4, marks ep 4.

**"Mark all up to here" (bulk mark through episode N)** — extremely common for:
- Users catching up on a show after it finished
- Importing a show they watched before using the app
- Correcting after forgetting to mark episodes

Trakt explicitly has a "mark all watched" per season button. TVTime has "I've watched
everything up to here." This is the second most-requested feature on trackers after the
basic checkbox.

**Also common:** "Mark entire season watched" — one button at the season header level.

### 4. Standard Progress Indicators

**Universal across all trackers:** `X / Y episodes` text counter at the season level.
Example: `4 / 10 episodes watched`

**Also widely used:**
- Thin progress bar under the season header (fills left to right as episodes complete)
- Percentage is rare — most apps skip it; `X/Y` is more scannable
- Color change on the season header when 100% complete (often green or accent color)

**On the series card in the main grid** (already in scope per PROJECT.md):
- Most apps show `S2 · 4/10` or a small progress bar as a subtitle on the card
- Plex shows a partial progress bar at the bottom of the poster

### 5. Episode Metadata: Useful vs. Clutter

| Metadata | Verdict | Reasoning |
|---|---|---|
| Episode number (S01E03) | Essential | Orientation; how users refer to episodes |
| Episode title | Essential | Memory anchor; users scan for "the one where..." |
| Air date | Essential | Distinguishes "haven't seen" from "not released" |
| Runtime | Useful (secondary) | Nice for planning; rarely the focus |
| Synopsis/overview | Useful but expandable | Show on hover/expand, not always-visible; clutters row |
| TMDb rating per episode | Clutter for v1 | Invites comparison, adds noise, no action follows |
| Writer/director credits | Clutter | Power-user data; irrelevant to tracking |
| Still image/thumbnail | Differentiator | Nice but costly (extra API call + cache per image) |
| Guest stars | Clutter | Nobody tracks this; belongs in a detail panel |

**Row design rule from real trackers:** Always-visible = number + title + date + watched state.
Everything else lives in a collapsed detail or tooltip.

---

## Table Stakes

Features users expect in any season view. Missing = product feels broken or half-done.

| Feature | Why Expected | Complexity | Notes |
|---|---|---|---|
| Checkbox/toggle per episode | The core purpose of the view | Low | Tkinter Checkbutton or Canvas-drawn toggle |
| Episode number visible on every row | Orientation; users think "S02E05" | Low | Never omit |
| Episode title on every row | Memory recall; "the one where..." | Low | Truncate with ellipsis if needed |
| Air date per episode | Distinguish unseen vs unaired | Low | TMDb provides `air_date` per episode |
| Season-level progress counter (X/Y) | Users want to know how far through | Low | Derived from episode_progress table |
| Season header with season number | Structure; multiple seasons need clear grouping | Low | Collapsible is ideal |
| Seasons navigable without closing window | Switching between S1, S2, etc. | Medium | Tab strip or listbox on left/top |
| Watched state persists across sessions | Core functionality | Medium | SQLite episode_progress table |
| Loading state while fetching TMDb | Network call can take 1-3s | Low | Spinner or "Loading..." label |
| Error state if TMDb fails | API can be down or key missing | Low | Clear message, retry option |
| "Mark entire season watched" button | Bulk action; universally expected for catch-up | Low | Single DB write, loop over episodes |

## Differentiators

Features that elevate the experience. Not expected, but appreciated when present.

| Feature | Value Proposition | Complexity | Notes |
|---|---|---|---|
| "Mark all watched up to episode N" | Huge for catch-up; distinguishes good trackers | Medium | Right-click context menu or dedicated button |
| Collapsible seasons | Keeps window compact for long-running shows | Medium | Toggle season body visibility |
| Progress bar under season header | Visual progress glance; satisfying when it fills | Low | Canvas line or ttk.Progressbar |
| "Next unwatched" highlight | Jump straight to where you left off | Low | Find first unchecked episode, scroll to it, highlight |
| Episode count badge on series card in main grid | Shows progress at-a-glance without opening Season View | Low | Already in scope per PROJECT.md |
| Still image on hover/expand | Rich context; Trakt shows these | High | Extra TMDb API call per ep + image cache; defer to v2 |
| Aired/unaired visual distinction | Greyed-out future episodes; avoids confusion | Low | Compare air_date to today() |
| Specials/Season 0 handling | TMDb has Season 0 for specials; most users ignore it | Low | Hide by default, toggle to show |

## Anti-Features

Features that look reasonable but add complexity with no payoff in this context.

| Anti-Feature | Why Avoid | What to Do Instead |
|---|---|---|
| Episode-level ratings (stars/score) | Adds a second interaction on every episode; bloats DB; out of scope per PROJECT.md | Aggregate series rating already exists |
| "Currently watching" playback tracker (timestamp within episode) | Requires video player integration; completely outside this app's purpose | Not applicable |
| Social/sharing features (share watched episode) | Desktop personal-library app; no social layer exists | Keep it private/local |
| Push notifications for new episodes | Requires background polling daemon; out of scope per PROJECT.md | Can add in a future milestone |
| Watch history timeline/calendar view | Interesting but irrelevant to the Season View window; separate feature entirely | Defer indefinitely |
| Per-episode notes/comments | Adds a text field to every row; clutters UI; low usage in practice | If needed, add at series level only |
| Streaming service links per episode | Requires mapping TMDb → provider; licensing complexity | Out of scope; separate feature |
| Manual episode reorder / custom sort | TMDb order is canonical; no user need for reordering | Trust TMDb data |
| Pagination within a season | Seasons are 6-26 episodes; all fit in a scrollable list | Use a scrollable frame, not pages |
| Confirmation dialogs on toggle | Trivially reversible action; dialogs add friction | Undo via re-click; no dialog |

---

## Feature Dependencies

```
Season View opens
  └── TMDb series_id must be persisted in items table
        └── Requires schema migration (items.tmdb_id column)

Episode rows render
  └── TMDb /tv/{id}/season/{n} must return data
        └── Requires API key in config.json

Checkbox toggles
  └── episode_progress table must exist
        └── Requires schema v7 migration

Season progress counter (X/Y)
  └── episode_progress row count per season

Series card progress badge (main grid)
  └── episode_progress aggregate query on grid load
```

---

## MVP Recommendation

For the Season View v1 (this milestone), prioritize in order:

1. **Season header with season number + episode count + X/Y progress counter**
   — Structure and feedback at a glance

2. **Scrollable episode list with: number, title, air date, checkbox**
   — The core interaction loop; must work perfectly

3. **Checkbox toggles persist to SQLite immediately**
   — Without this, nothing matters

4. **"Mark all watched" button per season**
   — Catch-up use case is extremely common; effort is minimal (loop + bulk INSERT)

5. **Loading and error states for TMDb calls**
   — Network is unreliable; silent failure is unacceptable

6. **Season selector (tab strip or listbox) to navigate between seasons**
   — Multi-season series need this; single-season shows still work without it
   — But omitting it makes the feature feel like a prototype

**Defer to v2:**
- "Mark all up to episode N" (right-click context)
- Collapsible season sections
- Progress bar (visual; X/Y text covers the need for v1)
- Still images per episode
- Specials/Season 0 toggle
- Next-unwatched highlight (can add with ~5 lines once base is working)

---

## Interaction Model Details (Tkinter-specific)

These patterns translate directly to Tkinter implementation decisions:

**Episode row layout (left to right):**
```
[Checkbutton]  [Ep# label]  [Title label — expands]  [Date label — right-aligned]
```

**Checkbutton behavior:**
- `BooleanVar` bound to each episode
- `command` callback writes to SQLite and updates season counter label
- No confirmation; no undo stack needed for v1

**Season selector:**
- `ttk.Notebook` tabs work but look generic on dark themes
- A `Listbox` or a row of styled `Button` widgets on the left sidebar is more consistent
  with the app's existing dark-theme aesthetic (described as using palette dict `C`)

**Scroll:**
- Wrap episode list in a `Canvas` + `Frame` with custom scrollbar — the standard Tkinter
  virtual scroll pattern already used by the app (virtual scroll + pagination exists)
- Do NOT use `ttk.Treeview` — it fights dark theming and checkbox styling is awkward

**"Mark all watched" placement:**
- Button in the season header row, right-aligned
- Label: "Mark all watched" or just a checkmark icon button
- On click: loop all episodes for that season, set watched=True, bulk-insert to SQLite,
  refresh all BooleanVars and the X/Y counter

---

## Sources

Note: Web access was unavailable during this research session. Findings are based on
training-data knowledge of the following products, which the author has documented
extensively through August 2025:

- Trakt.tv — web app season/episode view
- TVTime — iOS/Android episode tracking
- Episodate — web episode calendar and tracking
- Serializd — Letterboxd-style series tracker
- Plex Media Server — season view in web/desktop clients
- Infuse (Firecore) — Apple TV/iOS media player season view
- JustWatch — season episode lists (no tracking, but layout reference)

Confidence for table stakes items: MEDIUM-HIGH (consistent across all sources)
Confidence for differentiator ranking: MEDIUM (community feedback patterns from memory)
Confidence for Tkinter-specific patterns: HIGH (derived from app's existing code context)
