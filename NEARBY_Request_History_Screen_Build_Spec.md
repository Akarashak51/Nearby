# NEARBY — Request History Screen Build Specification
### Everything needed to build the customer's request-history screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§5.1, 5.2, endpoint table §5, page-architecture item 1.12), NEARBY Brand/PWA Guide v1.0, and order-history/filter-chip UX pattern research (Mobbin order-history/order-detail pattern surveys, filter-chip design guidance)

---

## 0. Scope — a genuine gap in the source document, named plainly, and the minimal fix proposed

This is the second build in this series to run into something the Master Build Document doesn't actually define. §5 states its endpoint table is "now the single authoritative list — every endpoint referenced anywhere else in this document appears here," and that table has no `GET /requests` or `GET /requests/me` entry — **there is currently no documented endpoint for a customer to list their own past requests.** The shop side got this exact capability in v2.6.1 (`GET /shops/me/requests`, purpose-built with pagination and a socket-drop fallback); the customer side never received its equivalent.

This screen cannot be built without that endpoint existing, so this document proposes the minimal, pattern-consistent addition needed, rather than silently assuming it or inventing something more elaborate:

> **Proposed: `GET /api/v1/requests/me`** — Bearer (customer), paginated per the existing §5.2 convention (`?page=&limit=`, default `limit=20`, capped at 100), sorted descending `createdAt` (the existing global default, §5.2), optionally filterable by `?status=` to support the filter chips in Section 3 below. Returns each request's `productQuery`, `status`, `createdAt`, and — for `selected` requests — the chosen shop's `name`. This is a direct structural mirror of `GET /shops/me/requests`, adding no new collection, field, or infrastructure; it's a read query over the existing `requests` collection, scoped by the caller's own `userId` the same way every other "me" endpoint in this API already resolves ownership from `payload.sub`, never a client-supplied id.

Everything below assumes this endpoint (or an equivalent the team names differently) exists — flagged clearly so it gets added to the endpoint table rather than discovered missing mid-build.

---

## 1. Page goals, in priority order

1. Let the customer quickly find a past search and see what happened to it — status is the single most important piece of information per row, so it gets the most visual weight.
2. Let a customer with an **active** request (still `open` or `awaiting_selection`) jump straight back into that live state from here, not just view a static record of it.
3. Keep first-time-user emptiness informative, not blank — per empty-state research already applied twice in this series (Expired screen, Cancelled screen), extended here to the simplest case: a customer who's never broadcast anything yet.
4. Don't over-build filtering for a pilot-scale personal history — a customer's own request count will be small; simple status chips are enough, no need for date-range pickers or search-within-history.

---

## 2. Layout overview

Single scrollable screen, reached via the bottom tab bar's "History" tab (Home page spec, Section [I]) — this is a normal "place" in the app, full header/tab-bar chrome, not a modal.

```
┌─────────────────────────────┐
│ [A] Header (title only)       │
├─────────────────────────────┤
│ [B] Status filter chips        │
├─────────────────────────────┤
│ [C] Request list               │  ← scrolls, paginated
│     (repeating row component)  │
├─────────────────────────────┤
│ [D] Empty state                │  ← replaces C when list is empty
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, no back-chevron (this is a tab-bar destination, not a drill-through screen), centered or left-aligned title (left-aligned is more standard for a primary tab screen, distinct from the centered-title treatment used on modal/task screens elsewhere in this series).
**Copy:** "Your searches"
**Background:** White, 1px Border-gray bottom border appears on scroll (same subtle depth cue as the Home page's top bar).
**Animation:** none.

### [B] Status filter chips
**Purpose:** lets a customer scan for a specific kind of outcome quickly — per the filter-chip research reviewed, chips are the right pattern here specifically because there are few, mutually-exclusive categories to choose from, not an open-ended filter space.
**Layout:** horizontal-scroll row of rounded-full pills, 36px height, directly below the header, 16px padding.
**Chip set:** **All** (default, selected) · **Active** · **Selected** · **Not fulfilled** — four options, deliberately collapsed from the underlying five-value `status` enum (`open`, `awaiting_selection`, `selected`, `expired`, `cancelled`) into four customer-facing buckets: "Active" groups `open` + `awaiting_selection` (both are "still in progress" from the customer's point of view, and splitting them into two separate filter chips would be a distinction that matters to the system, not to the customer browsing their own history), "Not fulfilled" groups `expired` + `cancelled` (both are "didn't end in a visit," and a customer scanning history is more likely to think in those terms than in the specific mechanism that closed each one).
**Selected-state:** filled Primary-orange background, White text; unselected: Gray-100 background, Text-primary text — matches the exact chip-selection pattern already used for the New Broadcast screen's radius quick-picks.
**Behavior:** single-select (tapping a new chip replaces the active filter, doesn't add to it — per the filter-chip research's guidance to show what's currently active clearly, a single-select row does that inherently without needing a separate "active filters" summary strip, which would be overkill for a 4-option set this small).
**Animation:** chip-selection background transition, `duration-instant` (100ms), `ease-standard` — identical to the radius quick-pick chips.

### [C] Request list — the main content
**Layout:** vertical list, one row per request, 12px vertical spacing, White background rows on the page's Gray-100 base (a subtle depth distinction between page background and row surfaces, consistent with how cards are treated everywhere else in this series).

**Row anatomy, per the order-history pattern research (status + short description + timestamp is the standard "scannable at a glance" row structure):**
- **Left:** a small status-indicator dot or icon (12×12), color-coded — see Section 4 for the exact mapping.
- **Middle (main content):** product query text (Body, 600 SemiBold, Text-primary, single line with ellipsis truncation) as the first line; second line, Caption, Text-secondary, varies by status:
  - `open`/`awaiting_selection` → "Still active · [relative time]" (e.g. "Still active · 4 min ago")
  - `selected` → "Selected **[Shop Name]** · [relative time]"
  - `expired` → "No shop selected · [relative time]"
  - `cancelled` → "Cancelled · [relative time]"
- **Right:** a small chevron (›) if the row is tappable to a detail/resume state (see below), Text-secondary.
**Tap behavior — this is the one place this screen's rows meaningfully diverge by status:**
- `open` or `awaiting_selection` → tapping resumes the customer directly on the **Live-Count** or **Shortlist** screen respectively, exactly as if they'd just created or reached that request live — this is Goal 2 from Section 1, and it's what makes this screen more than a static log.
- `selected`, `expired`, `cancelled` → tapping opens a lightweight, read-only summary (not a full separate screen build — reuse the existing recap-card content pattern already established across the Live-Count/Expired/Cancelled specs: 🔍 product query, 📍 radius, 📌 location, plus the resolved outcome) presented as a bottom-sheet/modal rather than a new route, since there's nothing actionable left to do with a resolved request beyond reviewing what happened.
**Pagination:** infinite-scroll, fetching the next page (`?page=n+1&limit=20`) as the customer approaches the bottom of the currently-loaded list — consistent with the API's existing pagination convention (§5.2), no "Load more" button needed for a pilot-scale personal history list.
**Animation:** rows fade in + translateY(8px→0) over `duration-standard` (240ms) with `stagger-step` (40ms) on initial load only — the same list-reveal pattern already used for the Home page's Repeat Request carousel and the Shortlist screen's card list; newly-loaded pages (from infinite scroll) append without the stagger, appearing directly, since a stagger effect on every subsequent page load would start to feel repetitive rather than polished.

### [D] Empty state *(replaces [C] entirely when the filtered list has zero results)*
**Two sub-cases, per the empty-state principle of explaining *why* rather than showing a bare "nothing here":**

**True empty (no requests at all, "All" filter, brand-new customer):**
- Icon: a simple outline search/radar glyph, 64×64, Text-secondary (visually related to, but distinct from, the Expired screen's zero-response icon — this is "you haven't started," not "something didn't come back").
- Copy: "No searches yet" (H2, Text-primary) + "Once you broadcast a search, it'll show up here." (Body, Text-secondary).
- Action: "Start a search" button (filled Primary orange, same treatment as every other primary CTA) — opens the New Broadcast screen directly.

**Filtered-empty (requests exist, but none match the selected chip — e.g. tapping "Selected" when the customer has never completed one):**
- Icon: same glyph, same tone.
- Copy: "Nothing here yet" (H2, Text-primary) + "You don't have any [chip label, lowercased] searches." (Body, Text-secondary) — a small templated line rather than four separately hand-written variants, since the underlying message is structurally identical across all three non-"All" chips.
- Action: "Show all searches" text link (Caption, Primary orange) — resets the filter to "All" rather than offering "Start a search" again (which would be redundant if the customer already has history, just none matching this particular filter).

**Animation:** icon fades in (`duration-standard`, `ease-out-soft`) on appearance — same restrained, one-time entrance as every other empty/outcome-state icon in this series.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | Selected filter chip, "Start a search" / "Show all searches" actions |
| Success | `#00B8A9` | Status dot for `open`/`awaiting_selection` rows (an active, in-progress state — reusing the same Success-teal association the Live-Count/Home-banner pulse already carries for "live") |
| Text primary | `#14213D` | Header title, product-query text, empty-state headline |
| Text secondary | `#5B6472` | Row second-line text, unselected chip text, chevrons, empty-state body copy |
| Border | `#D8DCE3` | Header scroll-divider |
| Surface | `#F4F5F7` / `#FFFFFF` | Page background, unselected chip background (Gray 100) / row surfaces, header (White) |

**Status-dot mapping, using existing tokens only, no new colors:**
- `open` / `awaiting_selection` → Success teal (active/live)
- `selected` → Success teal as well (a completed, successful outcome — same token, different semantic read depending on context, consistent with how Success is already used for both "in progress" and "resolved well" elsewhere: the Selection-Confirmation screen's checkmark and the Live-Count pulse both use Success teal too)
- `expired` / `cancelled` → Text-secondary gray dot (neutral, not Danger — same reasoning as every prior "outcome" screen in this series: neither of these represents an error)

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Filter chip labels | Caption / 13px | 400 Regular |
| Row product-query text | Body / 15px | 600 SemiBold |
| Row status/timestamp line | Caption / 13px | 400 Regular |
| Empty-state headline | H2 / 18px | 600 SemiBold |
| Empty-state body copy | Body / 15px | 400 Regular |
| Empty-state action button/link | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Filter-chip selection, tap-scale on rows/buttons |
| `duration-standard` | 240ms | Row-list entrance stagger (initial load only), empty-state icon fade-in |
| `stagger-step` | 40ms | Delay between rows on initial paint |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Row entrance, empty-state icon entrance |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Chip-selection transitions |

**No looping animation anywhere on this screen** — this is a browsable record screen, not a live/real-time one (even the `open`/`awaiting_selection` rows just link out to where the live state actually lives, rather than reproducing the pulse animation inline in a list row, which would be visually noisy across potentially multiple simultaneous rows).
**Reduced motion:** row-entrance stagger and empty-state fade both covered by the app-wide `prefers-reduced-motion` query, degrading to instant appearance.

---

## 7. Images, icons & video — asset list

**No video, no photography.**

| Asset | Format/size | Source |
|---|---|---|
| Status dot | CSS-drawn circle, 12×12, color per Section 4 | — |
| Chevron (row tap affordance) | Line icon, 16×16, Text-secondary | Icon library |
| Empty-state search/radar glyph | Outline icon, 64×64, Text-secondary | Icon library |
| Recap-card icons in the read-only detail bottom-sheet (🔍 📍 📌) | Emoji, consistent with every earlier recap card in this series | Reused, no new asset |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B]/[C] | `GET /api/v1/requests/me` (Section 0's proposed endpoint) with `?page=&limit=20&status=` — the `status` query param maps the four customer-facing chip buckets to the underlying enum values server-side (e.g. "Active" → `status=open,awaiting_selection`) |
| [C] resume tap (`open`/`awaiting_selection`) | Reuses the existing `GET /api/v1/requests/:id` call already specified for the Live-Count/Shortlist screens — this screen doesn't need its own separate live-state fetch logic, it just navigates to those existing screens with the tapped request's id |
| [C] read-only detail tap (`selected`/`expired`/`cancelled`) | Data already present in the list row itself (`productQuery`, `status`, `createdAt`, shop name if `selected`) — no additional fetch needed for the bottom-sheet summary unless radius/location weren't included in the list payload, in which case a single `GET /api/v1/requests/:id` call on tap covers it |
| [D] "Start a search" | Client-side navigation to the New Broadcast screen, no endpoint |

**Loading state:** initial load shows 3–4 skeleton rows (gray placeholder blocks, shimmer) while the first page fetches; subsequent infinite-scroll pages show a small inline spinner at the bottom of the list rather than a full skeleton reset.
**Error state:** a failed fetch (initial or pagination) uses the shared toast/error-boundary pattern (§6.1) with a retry option; a failed initial load can fall back to rendering the true-empty-state copy with an added note ("Couldn't load your search history — try again") rather than a blank screen, so the customer isn't left looking at nothing with no explanation.

---

## 9. Responsive & platform notes

- **Viewport:** standard vertical list, max-width 480px centered, consistent with every screen in this series; the filter-chip row scrolls horizontally on narrow viewports rather than wrapping, keeping the header area a fixed, predictable height.
- **Pilot-scale expectation:** like the Shortlist screen's own note on list length, a single customer's personal history is expected to stay short in the pilot context — infinite scroll is more than sufficient, and no virtualization is needed at this scale (explicitly a non-concern for now, not an oversight).
- **PWA standalone mode:** this screen keeps the bottom tab bar (Home page spec, Section [I]) active and highlighted on "History" — it's one of the app's four permanent destinations, not a task screen, so it should never appear without the tab bar the way every modal/task screen in this series deliberately does.
- **Endpoint dependency:** flagged again for visibility — this entire screen is blocked on the `GET /requests/me` endpoint proposed in Section 0 existing; this should be raised with whoever owns the Master Build Document's endpoint table before implementation starts, not discovered as a blocker mid-sprint.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 5.1, 5.2, and the master endpoint table in Section 5) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). Unlike every other build in this series, this one identifies a genuine, unaddressed gap in the source document — no customer-side request-listing endpoint currently exists — and proposes the minimal, convention-consistent addition (`GET /requests/me`, mirroring the existing `GET /shops/me/requests` pattern exactly) needed to make this screen buildable at all.*
