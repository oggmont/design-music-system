# Design System

The visual and interaction building blocks for the web app. The tokens here are the ones the [mockups](mockups/index.html) use, so a future implementation can copy them as they are. Screen behavior is specified in [UI-Design.md](../UI-Design.md).

Two visual proposals share **one component set** and differ only in tokens and layout.

> **Decision D-02 (2026-11-10): Proposal A, *Sleeve*, is the chosen direction.** Build with the **A** values. The B (*Gig poster*) columns stay here for reference only and aren't maintained further (see [Mockups and visual proposals](../UI-Design.md#mockups-and-visual-proposals)).

**Contents**

- [Principles](#principles)
- [Tokens](#tokens)
- [Typography](#typography)
- [Layout and spacing](#layout-and-spacing)
- [Components](#components)
- [Iconography and covers](#iconography-and-covers)
- [Motion](#motion)
- [Copy guidelines](#copy-guidelines)

---

## Principles

1. **Covers and names first.** The playlist is the hero object. Its cover and name are always the most visible thing on a card or page.
2. **Show the limit before it bites.** Every limit (5 favorites, 100 songs, 20 playlists) shows as a count before you reach it, and reaching it offers a way forward.
3. **Two taps to collect.** Adding a song never takes more than *Add → playlist*.
4. **Undo over confirm.** Small, reversible actions (add, remove a song, unfavorite) happen at once and offer Undo. Only deleting a playlist asks first.
5. **Calm by default.** Color marks state (favorited, full, error) and the one main action on a screen, not decoration.

---

## Tokens

Names are shared, values change per proposal. Use the semantic names in components, never the raw values.

### Color

| Token | Role | A: Sleeve (light) ✅ chosen | B: Gig poster (dark), reference |
|---|---|---|---|
| `--c-bg` | Page background | `#F4F6FA` | `#11151D` |
| `--c-surface` | Cards, menus, dialogs | `#FFFFFF` | `#1A202B` |
| `--c-surface-2` | Hovered rows, inputs, placeholders | `#E9EDF4` | `#232B38` |
| `--c-line` | Borders, dividers | `#D5DBE6` | `#2E3747` |
| `--c-text` | Main text | `#151A2D` | `#F1ECE2` |
| `--c-text-2` | Secondary text (owner, artist, counts) | `#5A6278` | `#A6A091` |
| `--c-accent` | Main action, links, focus ring | `#2445E0` | `#F5B544` |
| `--c-on-accent` | Text on the accent | `#FFFFFF` | `#1A1306` |
| `--c-fav` | Filled heart, favorite count | `#C8284B` | `#FF6B81` |
| `--c-ok` | Success toast icon, "✓ Added" | `#167349` | `#5FD39A` |
| `--c-danger` | Delete, errors | `#C42B2B` | `#FF7A6B` |

Contrast was checked for every text color (`--c-text`, `--c-text-2`, `--c-accent`, `--c-fav`, `--c-ok`, `--c-danger`) on `--c-bg`, `--c-surface` and `--c-surface-2`, and for `--c-on-accent` on `--c-accent`. Every pair meets AA for text (4.5:1); the lowest is 4.63:1 (A, `--c-fav` on `--c-surface-2`).

**Cover palette** (generated covers, see [below](#iconography-and-covers)): six pairs per proposal.
- A: `#2445E0/#9DB0FF`, `#E0365A/#FFC2CE`, `#1F8A5B/#A9E8C9`, `#C77800/#FFD99A`, `#5B36C9/#D2C4FF`, `#0E7C86/#A3E6EC`
- B: `#F5B544/#5A3A05`, `#FF6B81/#4A1220`, `#5FD39A/#0F3B27`, `#7FA7FF/#13254D`, `#C99BFF/#2E1650`, `#F2ECE2/#2B2F38`

### Radius, shadow, size

| Token | A ✅ | B (ref.) | Use |
|---|---|---|---|
| `--r-cover` | `6px` | `2px` | Covers (sleeves have nearly square corners) |
| `--r-control` | `999px` | `6px` | Buttons, chips |
| `--r-panel` | `14px` | `8px` | Menus, dialogs, cards |
| `--shadow-pop` | `0 12px 32px rgb(21 26 45 / .16)` | `0 12px 32px rgb(0 0 0 / .5)` | Menus, dialogs, toasts only |
| `--hit` | `44px` | `44px` | Minimum touch target |

---

## Typography

| Role | A: Sleeve ✅ | B: Gig poster (ref.) | Use |
|---|---|---|---|
| Display | **Archivo**, width 125, weight 800 | **Bebas Neue**, 400, uppercase | Playlist name on S7, profile name on S8, screen titles |
| Text | **Figtree**, 400 / 600 / 700 | **Manrope**, 400 / 600 / 700 | Everything else |
| Numbers | Text face with `tabular-nums` | Text face with `tabular-nums` | Durations, counts |

Fonts load from Google Fonts with system fallbacks (`system-ui, sans-serif`).

**Scale** (rem, root 16 px):

| Step | Size | Used for |
|---|---|---|
| `--t-hero` | `clamp(2.5rem, 6vw, 4.5rem)` | Playlist name (S7), profile name (S8) |
| `--t-h1` | `2rem` | Screen titles (Songs, Discover) |
| `--t-h2` | `1.25rem` | Section headings on Home |
| `--t-body` | `1rem` | Song titles, form fields |
| `--t-small` | `0.875rem` | Artist · album, owner, counts |
| `--t-label` | `0.75rem`, uppercase, letter-spacing `0.08em` | Eyebrows ("PLAYLIST"), menu group labels ("LAST USED") |

The hero name balances its lines (`text-wrap: balance`) and never truncates. Card names stop at two lines.

---

## Layout and spacing

- **Spacing scale:** 4, 8, 12, 16, 24, 32, 48, 64 px (`--s-1` … `--s-8`). Gaps between siblings use `gap`, not margins.
- **Content width:** at most 1200 px, with a 16 px gutter on phones and 32 px on desktops.
- **Grid of cards:** `repeat(auto-fill, minmax(160px, 1fr))`. On a phone: 2 columns, or a sideways shelf.
- **Breakpoints:** 640 px (phone to tablet) and 1024 px (tablet to desktop). See [Responsive behavior](../UI-Design.md#responsive-behavior).

---

## Components

Every interactive component lists its states. *All* of them also have **focus** (a 2 px `--c-accent` ring, 2 px offset) and **disabled** (50% opacity, no pointer events, and a reason in a tooltip or a hint).

| Component | Anatomy | States and rules |
|---|---|---|
| **Button** | Label, optional leading icon. Variants: *primary* (accent fill), *secondary* (outline), *ghost* (text only), *danger*. | default · hover · pressed · loading (label becomes "Creating…", width kept) · disabled. At most one primary per screen or dialog. |
| **Icon button** | 44 × 44 hit area, icon 20 px, `aria-label`. | Same as Button. |
| **Heart (favorite toggle)** | Icon button with `aria-pressed`. Outline when off, filled `--c-fav` when on. | off · on · pending (a short pulse while the request runs). Label: "Favorite {name}" / "Remove {name} from favorites". |
| **Text field** | Label above, input, hint or error below, optional counter ("14/120"). | default · focus · filled · error (border `--c-danger` + message, linked with `aria-describedby`) · disabled. |
| **Select** | Native `<select>` styled like a text field. Only for simple lists (country). | As Text field. |
| **Search field** | Search icon at the start, input (pill shape in A), clear ✕ at the end once there's text. Label "Search songs" for screen readers; placeholder "Search songs, artists or albums". | empty · typing · searching (results below are dimmed, placeholder rows) · has results (count line "9 songs match …" with `role="status"`) · no results. Esc clears. Full width on phones. |
| **Visibility badge** | Eyebrow text above the playlist name, with a lock icon when private: "Public playlist · on your profile" / "Public playlist · in 2 people's favorites" / "Private playlist · only you". | Owner only. Changes as soon as anyone favorites or unfavorites the playlist. |
| **Playlist card** | Cover (square), name (2 lines max), meta line ("by Diego Ramos · 12 songs"), optional heart over the cover's corner. The whole card is one link; the heart is a separate button. | default · hover (cover lifts 2 px) · loading placeholder. |
| **Empty favorite slot** | Dashed square the size of a cover, "+" and "Find a playlist". | Only on your own Home and profile. Links to Discover. |
| **Song row** | Position or number, title, artist · album, duration, action (Add, or ⋯ for the owner on S7). | default · hover (`--c-surface-2`) · just added (a brief `--c-ok` check). On a phone: two lines, the album is hidden. |
| **Add menu** | Title "Add to playlist", group "LAST USED" (one item), group "ALL PLAYLISTS (A–Z)", divider, "+ New playlist". Optional filter field with 8+ playlists. Each item: small cover, name, count "n/100" or status. | item: default · hover/active · full (disabled, "Full") · contains (disabled, "✓ Added"). The whole menu: no playlists ("You don't have a playlist yet"). A bottom sheet on phones. |
| **Count / meter** | "3 of 5", "7 / 100", "9 of 20" in `--t-small`, `tabular-nums`. The 100-song version also has a thin bar. | normal · near the limit (≥ 90%, `--c-text` and bold) · at the limit (says "Full" or "Limit reached"). |
| **Dialog** | Title, body, actions on the right (Cancel, then the primary). ✕ in the corner. | Opens with focus on the first field, or on the primary when there's no field. Esc closes it. Full screen on phones. Variants: create playlist, edit details (same form, filled in, "Save"), replace a favorite, last favorite, delete. |
| **Toast** | Icon, message, at most one action. | info · success · error. 5 s, paused while hovered or focused. `role="status"`. |
| **Avatar** | Image or initials on a color made from the username. Sizes 32 (header), 40 (owner line), 128 (profile). | image · initials. |
| **Header / nav** | Logo, links (Home, Songs, Discover), New playlist button, avatar menu. | The current link uses `aria-current="page"`. |
| **Section header** | Title (`--t-h2`), optional count, optional "See all →" and Shuffle. | – |
| **Placeholder (skeleton)** | `--c-surface-2` blocks the size of the content. | A slow shimmer, turned off with reduced motion. |

---

## Iconography and covers

- **Icons:** one outline set at 1.5 px stroke and 20 px (for example Lucide, MIT license). The set needed for v1: home, music, compass, search, plus, heart, heart-filled, shuffle, link, lock, pencil, more-horizontal, check, x, trash, undo, log-out, user.
- **Generated covers:** with no `cover_url`, the cover is drawn from `playlist_id`: choose a color pair from the cover palette by `playlist_id mod 6`, choose one of four patterns (rings, stripes, quarter circle, grid of dots) by `playlist_id mod 4`, and put the first letter of the name in the display face. The cover is deterministic, needs no storage, and makes the same playlist look the same everywhere.

---

## Motion

- 150 ms for hover and press, 200 ms for menus and toasts, 250 ms for dialogs. Easing `cubic-bezier(.2, .7, .2, 1)`.
- One small moment of feedback: when a song is added, the playlist's count in the toast ticks up.
- With `prefers-reduced-motion: reduce`, there are no transitions, only instant changes.

---

## Copy guidelines

- **Use the person's words.** "Song", "favorite", "playlist", "your profile". Never "track", "follow", "entity" or an error code.
- **Buttons say what happens.** "Create", "Add", "Replace", "Delete playlist". The toast then says what happened: "Created", "Added to Road Trip".
- **Errors say what went wrong and how to fix it,** without apology: "Road Trip is full (100 songs). Remove a song or pick another playlist."
- **Limits are stated as facts with a way forward:** "You have 20 playlists, the most allowed. Delete one to make a new one."
- **Owners:** "by Ana García", or "by you" for your own.
- **Public and private:** always say what it means for people: "Public · on your profile", "Private · only you", "Favorite it to make it public."
- **Times and counts:** "3:05" for a song, "35 min" or "1 h 12 min" for a playlist, "1 song" / "7 songs".
