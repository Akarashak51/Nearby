# NEARBY — Shop Dashboard: Request History (Shop Side)
### Full Page Build Specification — Content, Layout, Visual Design, Copy, Motion & Data

**Version 1.0**
**Companion to:** NEARBY Master Build Document v2.6.2 (§3.1.1, §3.1.3, §4.3, §4.4, §5.2) · NEARBY Brand & PWA Guide v1.0 · NEARBY Page Architecture & Competitor Analysis (§2.9)
**Surface:** `/shop/*` — Teal-forward · **Role:** shop

---

## 0. Scope & How to Use This Document

This document specifies everything required to design and build the Request History (shop side) page, listed as page 2.9 in the Page Architecture & Competitor Analysis document — pattern-sourced from the Zomato Restaurant Partner app's order history screen, adapted to a broadcast-and-answer model with no order, cart, or fulfillment step (Master Build Document §1).

This is a **read-only, past-tense list** — every request shown here has already left the shop's incoming-request feed (page 2.5) one way or another. It exists so a shop owner can look back and answer questions like "did I actually get picked yesterday?" or "how many of my Yeses this week turned into an actual visit?" without those answers living only in the ephemeral, real-time feed.

Nothing in this document contradicts the Master Build Document v2.6.2 or the Brand & PWA Guide v1.0 — every field shown is read directly from the existing `requests` (§4.3) and `shopResponses` (§4.4) collections, and no new database fields or API routes are introduced.

---

## 1. Overview & Purpose

### 1.1 — What it is

A reverse-chronological list of every broadcast request this shop has ever responded Yes or No to, each row showing the outcome — **selected**, **not selected**, or **expired/cancelled before a selection was made** — with the option to drill into any entry for full detail. It is the shop-side mirror of the customer's own Request History (page 1.12), but framed entirely from the shop's point of view: not "what did I search for," but "what did I answer, and what happened next."

### 1.2 — Why it's explicitly a no-penalty zone

Master Build Document §3.1.3 states plainly: a Yes that isn't selected **never penalizes reliability**. This page is the one place a shop owner sees their full answer history side by side, including every not-selected Yes — so the page's entire visual and copy treatment must reinforce that a "Not selected" row is a neutral outcome, not a strike, or the page would quietly contradict the very formula the Reliability & Ratings view (page 2.8) works hard to explain fairly.

### 1.3 — What this page does not do

- It never shows another shop's response to the same request, or how many other shops said Yes — that comparative information isn't exposed to a shop, consistent with NEARBY not being a bidding or competitive-visibility marketplace.
- It never lets the shop owner edit a past response — every row here is immutable history.
- It does not duplicate the Reliability & Ratings view's aggregate score (page 2.8) — this page is the itemized ledger; that page is the summary and the formula explainer. They're deliberately different: a list you scroll versus a score you check.

### 1.4 — User story

> *"As a shop owner, I want to look back at the requests I answered — see which ones picked me, which didn't, and which never went anywhere — without digging through push notifications I might have dismissed."*

### 1.5 — Pattern source

Inspired by: Zomato Partner's order history screen (Page Architecture doc §2.9, pattern source). NEARBY's version has three outcome states rather than a delivery-order's typical states (preparing, out for delivery, delivered, cancelled), because there's no fulfillment leg — just answer, and then either get picked or not (Page Architecture doc §0, §5).

---

## 2. Functional Specification

### 2.1 — Data this page reads (all read-only, no writes)

| Source | Field(s) | Shown on this page as |
|---|---|---|
| `shopResponses` (§4.4) | `responseValue` (yes/no) | Filters which requests appear at all — only `yes` responses produce a meaningful history row; a `no` response is technically loggable but not surfaced here (see §2.2) |
| `shopResponses` (§4.4) | `selected` (Boolean) | Drives the outcome badge — Selected vs. Not selected |
| `shopResponses` (§4.4) | `distanceMeters`, `responseSeconds` | Shown in the detail drill-down (§4.3) only, not the list row |
| `requests` (§4.3) | `productQuery` | The headline text of each row — what the customer was looking for |
| `requests` (§4.3) | `status` (open, awaiting_selection, selected, expired, cancelled) | Drives the outcome badge when the shop's Yes never led to a selection (expired or cancelled) |
| `requests` (§4.3) | `createdAt` | Relative/absolute date shown per row |
| `requests` (§4.3) | `selectedShopId` | Compared against this shop's own ID to derive "Selected" vs. not, alongside `shopResponses.selected` |

### 2.2 — Why "No" responses aren't listed

A shop that answered No to a broadcast has nothing to look back on — there's no outcome to track, no selection that could have happened, and surfacing it here would just be noise in a history screen meant to answer "what happened after I said yes." This is a deliberate scope decision, not an oversight: the page is a history of **engaged** answers, not a full response log. (A full response log, if ever needed, would belong in an export/analytics context, not this page.)

### 2.3 — Outcome derivation (read-only logic, computed for display, not stored)

| Request status | `shopResponses.selected` | Outcome shown |
|---|---|---|
| `selected` | `true` | **Selected** |
| `selected` | `false` | **Not selected** |
| `expired` | `false` | **Expired — no selection made** |
| `cancelled` | `false` | **Cancelled by customer** |

No new field is needed to represent this — it's a pure client-side (or lightweight server-side join) derivation from two existing collections, exactly as the Master Build Document's own note on analytics (§5, intro to §5.4) describes computing metrics on read rather than storing derived state redundantly.

### 2.4 — API contract (existing, no changes needed)

`GET /api/v1/shops/me/requests` is the closest existing endpoint in name (§5.2) but is scoped to **currently-open** pending decisions (the real-time incoming feed, page 2.5) — **not** this page's data. Request History instead needs a **historical**, paginated equivalent already implied by the pagination convention in §5.2 of the Master Build Document, filtered to `shopResponses` entries belonging to this shop where the parent request's status is no longer `open` or `awaiting_selection`.

> **Note (implementation detail, not a new concept):** this is the same underlying pagination and filtering pattern already established for every other list endpoint in the Master Build Document (§5.2) — sorted by `requests.createdAt` descending, cursor- or page-based per that section's existing convention. No new pagination mechanism is introduced.

### 2.5 — Sorting & filtering

- **Default sort:** newest first (`requests.createdAt` descending) — matches every other history list in the app (customer Request History, page 1.12).
- **Filter chips:** All · Selected · Not selected · Expired/Cancelled — client-side filter over the already-fetched/paginated set, not a separate query per filter, keeping this simple for a v1 list of this size.
- No date-range picker in v1 — the list is short enough for a single shop's history that infinite scroll alone is sufficient; a date filter is flagged as a possible future addition if shop tenure grows long enough to need it, not built now.

---

## 3. Where It Lives

- **Entry point:** Shop Dashboard nav → same nav level as Home, Outlet, Reliability — labeled **"History"**.
- Reachable from nowhere else in v1 (unlike Opening Hours, which has a nudge entry point from the Live Dashboard) — this is a look-back screen a shop owner visits deliberately, not one the app needs to surface proactively.
- Tapping any row opens a detail view **in place** (expand/accordion) rather than navigating to a new route, since the full detail (§4.3) is only a few extra fields — not enough content to justify a page transition.

---

## 4. Section-by-Section Content & Layout

### 4.1 — Page header & filter bar

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Page header | "Request history" | H1 | Gray 900 |
| Filter chips | "All" · "Selected" · "Not selected" · "Expired/Cancelled" | Button / SemiBold | Horizontal scroll row if space-constrained; active chip filled Signal Teal, inactive chips outline Gray 300 |
| Result count (below chips) | e.g. "24 requests" | Caption | Gray 600, updates as filter changes |

### 4.2 — History row (list item)

Each row is a single-tap-to-expand card, not a separate page — full-width, 12px vertical padding, 1px Gray 300 bottom divider between rows (no card border/shadow per row, since a long scrolling list of individually-bordered cards would feel heavier than necessary for a look-back list).

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Product query (headline) | e.g. "Looking for: AA batteries" | Body / Medium | Gray 900, truncates to one line with ellipsis if long |
| Date | e.g. "Yesterday, 4:12 PM" (relative for < 7 days, absolute date beyond that) | Caption | Gray 600 |
| Outcome badge | "Selected" / "Not selected" / "Expired" / "Cancelled" | Caption / SemiBold, pill-shaped | Color per §5.1 — this is the single most important piece of information in the row, so it's the one element given a distinct colored pill rather than plain text |
| Chevron | Expand/collapse indicator | — | 16×16, Gray 600, rotates 180° on expand (§7) |

### 4.3 — Expanded row detail (accordion content)

Revealed in place below the row when tapped — no navigation, no modal.

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Your response time | "You responded in 14 sec" | Body | Gray 900 — the human-readable form of `shopResponses.responseSeconds` |
| Distance at time of request | "You were 0.8 km away" | Body | Gray 900 — human-readable form of `shopResponses.distanceMeters` |
| Outcome explainer (context-sensitive, one of four) | See §6 for exact copy per outcome | Caption | Gray 600 — this is where the no-penalty reassurance (§1.2) is stated explicitly for "Not selected" rows specifically |
| Collapse action | Tapping anywhere in the expanded area, or the row header again, collapses it | — | No separate "Close" button needed — the whole expanded block is tappable to collapse |

### 4.4 — Empty state (no history yet)

Shown when a shop has answered zero Yes responses to date (a brand-new shop, or one that has only ever said No):

> **Copy:** *"You haven't answered any requests yet. Once you say Yes to a broadcast, it'll show up here — whether or not you get picked."*

A single static instance of the logomark's pin (Brand Guide §9.4 empty-state pattern), no looping animation — same treatment as the Reliability & Ratings view's zero-history empty state, for visual consistency across the two "nothing here yet" moments a new shop can encounter.

### 4.5 — Filtered-empty state (a filter chip returns zero results)

Distinct copy from the true empty state above, since this means history *exists* but the filter excludes all of it:

> **Copy:** *"No requests match this filter yet."*

### 4.6 — End-of-list indicator (infinite scroll)

> **Copy:** *"That's everything — you're all caught up."*

Shown once pagination reaches the last page, in Caption / Gray 600, centered, with no further loading spinner below it.

---

## 5. Visual Design System Application

### 5.1 — Color mapping, including outcome badge colors

The outcome badge is the one place this page uses color to carry meaningful distinctions beyond the standard semantic set — specified precisely here to keep the four states visually and emotionally accurate to how neutral or positive each one actually is.

| Outcome | Badge color | Rationale |
|---|---|---|
| **Selected** | Signal Teal #00B8A9 fill, white text | Reuses the existing Success token (§3.2) — this is the unambiguously positive outcome |
| **Not selected** | Gray 100 #F4F5F7 fill, Gray 600 text (no colored fill at all) | Deliberately the *quietest* badge on the page — per §1.2, this must never look like a warning or a miss; a neutral gray pill with no accent color is the strongest possible visual statement that nothing went wrong |
| **Expired — no selection made** | Gray 100 #F4F5F7 fill, Gray 600 text | Same neutral treatment as "Not selected" — an expired request with no selection is not the shop's fault or failure either |
| **Cancelled by customer** | Gray 100 #F4F5F7 fill, Gray 600 text | Same neutral treatment — a customer-initiated cancellation has nothing to do with the shop's performance |

This means three of the four outcome badges intentionally share the same quiet neutral gray styling, and only "Selected" gets a colored, celebratory-adjacent treatment — reinforcing that Selected is the one outcome genuinely worth visually distinguishing, and everything else is simply "this one didn't lead anywhere," not a graded scale of lesser outcomes.

| Signal Teal #00B8A9 | Gray 100 #F4F5F7 | Gray 900 #14213D | Gray 600 #5B6472 | Gray 300 #D8DCE3 |
|---|---|---|---|---|

| Token | Used for in this feature | Rationale |
|---|---|---|
| Signal Teal #00B8A9 | "Selected" badge, active filter chip | Shop surface's dominant accent (§3.3), doubles as Success semantic token (§3.2) |
| Gray 100 / Gray 600 | Three of the four outcome badges, row dividers' surrounding whitespace | Deliberately under-styled per §5.1 rationale above |
| Gray 900 | Product-query headline text, expanded-detail body text | Standard body-text accessibility rule (§3.4) |
| Gray 300 | Row dividers, inactive filter chip borders | Border/divider token (§3.2) |
| Surface / White | Page background | Standard page token (§3.2) |

Orange is not used anywhere on this page, consistent with every other Shop Dashboard spec in this series — reserved exclusively for Yes/No response buttons elsewhere on this surface (§3.3), and this is a read-only history page with no such action.

### 5.2 — Typography mapping

| Role (Brand Guide §4.2) | Size / line-height / weight | Used here for |
|---|---|---|
| H1 | 24px / 32px, 600 SemiBold | Page header ("Request history") |
| Body | 15px / 22px, 400 Regular (Medium weight for the product-query headline) | Row headline text, expanded-detail response-time and distance lines |
| Caption | 13px / 18px, 400 Regular (SemiBold for badge text) | Date stamps, result count, outcome badges, outcome explainer text, empty states |
| Button | 15px / 20px, 600 SemiBold | Filter chip labels |

### 5.3 — Spacing, grid & component styling

- Row: 12px vertical padding, 16px horizontal padding, 1px Gray 300 bottom divider, no divider after the last visible row before the end-of-list indicator.
- Outcome badge: pill shape, fully rounded, 4px vertical / 10px horizontal padding, positioned at the row's trailing edge opposite the product-query text.
- Filter chip row: 8px gap between chips, horizontally scrollable if it overflows the viewport width, 36px chip height (slightly smaller than the 44px primary-action minimum, since these are secondary filter controls, not primary actions — though each chip's full tap target is still padded to meet the 44px accessibility minimum, §9).
- Expanded detail block: 12px top padding, indented slightly (4px) from the row's left edge to visually nest it under its parent row, Gray 100 background tint to distinguish it from the collapsed row list above and below it.
- Chevron rotation: pivots around its own center, no layout shift in surrounding content when it rotates.

### 5.4 — Iconography

- Chevron/caret (16×16, Gray 600) — same reused system glyph as the expand affordances on the Outlet Management and Opening Hours pages, rotates on expand/collapse (§7).
- Logomark pin (existing brand asset) — single static instance, used only in the true empty state (§4.4), identical treatment to the Reliability & Ratings view's own empty state for cross-page consistency.
- No new custom iconography is introduced anywhere on this page.

---

## 6. Full Copy Deck

| State / Location | Exact copy |
|---|---|
| Page header | Request history |
| Filter chips | All · Selected · Not selected · Expired/Cancelled |
| Result count | {N} requests |
| Row headline | Looking for: {productQuery} |
| Row date (recent) | Today, {time} · Yesterday, {time} |
| Row date (older) | {Month} {day}, {year} |
| Outcome badge — selected | Selected |
| Outcome badge — not selected | Not selected |
| Outcome badge — expired | Expired |
| Outcome badge — cancelled | Cancelled |
| Detail — response time | You responded in {T} |
| Detail — distance | You were {D} away |
| Detail explainer — Selected | This customer picked you — nice work. |
| Detail explainer — Not selected | The customer picked another shop this time. This doesn't affect your reliability score. |
| Detail explainer — Expired | No one selected a shop before this request's window closed. This doesn't affect your reliability score. |
| Detail explainer — Cancelled | The customer cancelled this request before choosing. This doesn't affect your reliability score. |
| True empty state | You haven't answered any requests yet. Once you say Yes to a broadcast, it'll show up here — whether or not you get picked. |
| Filtered-empty state | No requests match this filter yet. |
| End-of-list indicator | That's everything — you're all caught up. |
| Load error | Couldn't load your request history. Pull down to try again. |

---

## 7. Motion & Animation Specification

This is a scrolling, read-only list — motion is used only for the expand/collapse interaction, filter switching, and load feedback. Every token below is reused from the existing Motion Token table (Brand Guide §9.1) — nothing new is introduced.

| Interaction | Token(s) used | Behavior |
|---|---|---|
| Row expand/collapse | duration-quick (180ms), ease-out-soft | Detail block height-animates open/closed; chevron rotates 180° over the same duration, synced so both complete together |
| Filter chip switch | duration-instant (100ms) | Selected chip fills Signal Teal with the same quick tap-feedback scale used on other chip controls across this series (Rush-Hour duration chips, Opening Hours day toggles) |
| List filtering (chip change) | duration-quick (180ms), simple crossfade | Rows that no longer match the active filter fade out; remaining rows do not reflow with a slide animation — a straight crossfade avoids a jarring reshuffle-and-slide effect on a filter change |
| Initial page load | Skeleton-to-content pattern (Brand Guide §9.4) | Gray 100/Gray 300 shimmer placeholders for row shapes, swapped for real rows once the paginated request resolves |
| Infinite scroll — next page load | Small inline spinner at list bottom, standard rotating style (§9.3) | No full-page loading state on subsequent pages, only the initial load |
| Pull-to-refresh | Rotating-arrow spinner (§9.3, REST-fallback polling pattern) | Manual reload trigger, same as other list/dashboard pages in this series |
| Reduced motion | Single global override (§9.5) | Expand/collapse and crossfade both collapse to instant state changes — no page-specific reduced-motion work required |

---

## 8. Imagery, Graphics, Icons & Video

No photography, illustration, or video anywhere on this page. The only illustrative asset is the existing logomark pin, used once, statically, in the true empty state (§4.4, §5.4) — reusing the same established brand asset and treatment as the Reliability & Ratings view's own empty state, rather than introducing a second empty-state visual language for what is conceptually the same "nothing here yet" moment.

- Chevron icon: standard two-tone vector glyph, no custom illustration work required.
- No skeleton-loader design work beyond the standard shimmer pattern already defined in the Brand Guide (§9.4) — reused directly.

---

## 9. States & Edge Cases

| Case | Behavior |
|---|---|
| Brand-new shop, zero Yes responses ever given | True empty state shown (§4.4) |
| Shop has history, but the active filter matches none of it | Filtered-empty state shown (§4.5), distinct copy from the true empty state |
| A request the shop said Yes to is still `open` or `awaiting_selection` (in progress) | Not shown on this page at all — it belongs on the real-time incoming-request feed (page 2.5) until it resolves to one of the four outcome states in §2.3; this page only ever shows resolved history |
| Very long `productQuery` text | Truncates to one line with an ellipsis in the collapsed row; shown in full, wrapped, in the expanded detail view |
| Shop's response time was effectively instant (< 5 sec) | Displayed as "< 5 sec" rather than "2 sec", matching the same low-end precision rule used on the Reliability & Ratings view (§9 of that document) |
| Pagination/network failure on scroll | Inline retry affordance at the point of failure — "Couldn't load more. Tap to retry." — rather than a full-page error, since earlier pages already loaded successfully |
| Shop status becomes blocked while viewing this page | Page remains fully viewable — this is a read-only history screen with no action a blocked shop could attempt and fail at, same reasoning as the Reliability & Ratings view (§9 of that document) |
| Two requests with the exact same `createdAt` timestamp (rare) | Stable secondary sort by `_id` to guarantee deterministic ordering across page loads, consistent with the existing pagination convention (§5.2 of the Master Build Document) |

---

## 10. Accessibility

- Each row's outcome badge always carries a text label ("Selected", "Not selected", etc.) — color is reinforcement only, never the sole signal, especially important here since three of the four badges intentionally share the same neutral gray (§5.1).
- The expand/collapse chevron has an `aria-expanded` state and an accessible label ("Show details for this request" / "Hide details"), not a bare icon with no announced purpose.
- Filter chips use `role="tab"`/`aria-selected` semantics (or an equivalent toggle-button pattern) so screen-reader users know which filter is currently active.
- Minimum 44×44px touch target on every row's tap area, each filter chip's full hit area, and the pull-to-refresh/retry affordances — consistent with every other page in this series.
- Infinite-scroll loading is announced via `aria-live="polite"` on the inline spinner region, so screen-reader users are notified when more content has loaded without an intrusive interruption.
- Reduced motion: fully covered by the existing global override (§7) — no page-specific work required.

---

## 11. Admin Visibility & Data Management

- No new admin-facing surface is required — an admin drilling into a shop's profile through the All Shops directory (Page Architecture page 3.4) already has access to that shop's full `shopResponses` history through the same underlying data this page reads; this page is simply the shop-owner-facing presentation layer over data admin can already see in aggregate.
- This page has no interaction with the Reports queue (page 3.6) — it's a personal read-only history view with no action that could generate a report.
- No write path exists on this page at all (§1.3), so there is nothing here for admin moderation to govern.

---

## 12. Component Inventory — Developer Handoff

| Component | Type | States | Key tokens |
|---|---|---|---|
| RequestHistoryFilterBar | Horizontal chip row | all-active, selected-active, not-selected-active, expired-cancelled-active | Signal Teal, Gray 300, Button, duration-instant |
| RequestHistoryRow | Expandable list item | collapsed, expanded, loading-skeleton | Body, Caption, Gray 300 divider, duration-quick |
| OutcomeBadge | Pill label | selected, not-selected, expired, cancelled | Signal Teal (selected only), Gray 100/Gray 600 (other three) |
| RequestHistoryEmptyState | Composite (logomark + copy) | true-empty, filtered-empty | Body, Caption, logomark pin (static) |
| EndOfListIndicator | Text row | visible once last page reached | Caption |
| Toast (reused) | Existing shared component | load-error | duration-quick, ease-out-soft |

Five page-specific composites plus one reused shared component — no new design-system primitive is introduced by this page; every visual element (chip, pill badge, expandable row, skeleton) draws on patterns already established across the Rush-Hour Toggle, Outlet Management, Opening Hours, and Reliability & Ratings specifications in this series.

---

## Appendix A — Quick Reference: What's New vs. Reused

| New for this feature | Reused as-is from existing docs |
|---|---|
| Outcome-derivation logic (§2.3) — a display-only join of two existing collections, no new field | Every color token (§3.2, Brand Guide) |
| Copy deck (§6 of this document) | Every type-scale role (§4.2, Brand Guide) |
| Outcome badge color convention (§5.1) — three-of-four-neutral, one-positive | Existing pagination convention, unchanged (§5.2, Master Build Document) |
| Filter-chip scope decision (client-side over an already-fetched page, no per-filter query) | `requests` and `shopResponses` collections, unchanged (§4.3, §4.4, Master Build Document) |
| | Skeleton-loader, pull-to-refresh, and expand/collapse interaction patterns (§9.3, §9.4, Brand Guide) |
| | Empty-state pattern using the static logomark pin (§9.4, Brand Guide) — same treatment as the Reliability & Ratings view |
| | Reduced-motion global override (§9.5, Brand Guide) |

---

*— End of specification —*
