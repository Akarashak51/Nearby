# NEARBY — Direct Shop Search Screen Build Specification (FR-1.15 / UC-20)
### Everything needed to build the direct-browse screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§2.7, 5.2, 5.3, endpoint table §5, FR-1.15, UC-20), NEARBY Brand/PWA Guide v1.0, and location-search UI pattern research (NN/g's maps-and-location-finders research, category-as-first-level-filter pattern research)

---

## 0. Scope — what makes this screen categorically different from everything else built so far

Every customer-facing screen in this series up to now has served the **broadcast model** — ask, wait, shops answer, pick one. This screen is the deliberate exception, and your own doc frames it that way explicitly: FR-1.15 describes it as letting a customer search nearby shops directly, **"separate from broadcasting a request."** This is for the customer who already knows what they want (a specific kind of shop, not an item to ask around about) and wants to browse and go, the way they'd use any ordinary local-business finder — no waiting, no response window, no shortlist ranking by reliability.

Two backend facts shape this screen precisely:
1. **`GET /shops/nearby` is Public** (no auth required) and takes exactly `lat`, `lng`, `radiusMeters` (100–5000, same bounds as a broadcast), and an optional `category` — nothing else. There is no shop-name search parameter on this endpoint.
2. **Results are pre-filtered server-side to open, in-hours, approved shops only** (§2.7's eligibility rule, the same one that governs broadcast eligibility) — a shop that's closed or outside its schedule simply won't appear here at all, unlike the Shortlist screen (which can show a since-closed shop honestly, because it already said Yes before closing). This screen has no equivalent case to handle — the API guarantees every result is currently reachable.

Location-search UX research (NN/g's own study of maps/location-finder mobile apps) is unambiguous on one structural point directly relevant here: **list should be the default view, map optional** — users cared about distance being listed, not about a map being present, and a map view is best reserved for a single-place detail screen (already true in this product — the Shop Profile screen's "Get directions" deep link, not an inline map). This screen follows that finding directly rather than defaulting to a map-first layout the way a generic "find nearby places" screen might.

---

## 1. Page goals, in priority order

1. Let a customer who already knows what kind of shop they want find one fast — category filtering is the primary tool here, not free-text search (since the backend doesn't support shop-name search, and category is genuinely the more useful axis for "I want a pharmacy nearby" than typing a name they may not know yet).
2. Present results as a scannable list, sorted by distance (server-guaranteed, §5.2) — no map-first layout, per the research above.
3. Reuse, don't reinvent — this screen's card component and the Shop Profile screen it links to are both near-total reuses of components already built for the Shortlist flow, with one meaningful behavioral difference noted in Section 3.
4. Make clear, without being heavy-handed about it, that this is a different mode from broadcasting — a customer shouldn't confuse "I found this shop by browsing" with "shops responded to my specific ask."

---

## 2. Layout overview

Single scrollable screen, reached from the Home page's [D] Quick Category chips row via a "Browse shops" entry point (a small addition to that row's existing behavior — see Section 3[B] below for how it coexists with the broadcast-prefill behavior already specced there), or from a dedicated icon in the bottom tab bar's search affordance if the team prefers a more discoverable entry point than a single Home-page link.

```
┌─────────────────────────────┐
│ [A] Header (title + location) │
├─────────────────────────────┤
│ [B] Category filter chips      │
├─────────────────────────────┤
│ [C] Radius control (compact)   │
├─────────────────────────────┤
│ [D] Results list                │  ← scrolls, paginated
├─────────────────────────────┤
│ [E] Empty state                 │  ← replaces D when no results
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, back-chevron left (returns to Home), title + a small location indicator on the second line.
**Copy:** "Nearby Shops" (H1) + "Searching near **[formattedAddress]**" (Caption, Text-secondary) directly below — reuses whatever location the customer already has resolved from Home (Home page spec's location pill), so this screen doesn't ask the customer to re-confirm location on entry; a "Change" text-link (Caption, Primary orange) next to it opens the same Location-Confirm screen already built for the broadcast flow — **direct reuse**, not a new location-picker.
**Background:** White, 1px Border-gray bottom border on scroll.
**Animation:** none.

### [B] Category filter chips
**Purpose:** the primary filtering tool, per Goal 1 — location-search UI pattern research is explicit that categories work best as a first-level, always-visible filter row rather than buried in a secondary filter panel, exactly the treatment this section gives them.
**Layout:** horizontal-scroll row of rounded-full pills, 36px height, directly below the header — **visually and behaviorally identical to the Home page's Quick Category chip row** (same chip set: Groceries · Medicines · Electronics · Hardware · Stationery · Mobile Accessories · Bakery · Household, plus a leading **"All"** chip, selected by default).
**Key behavioral distinction from the Home page's chips:** on Home, tapping a category chip pre-fills the New Broadcast screen's text field and launches a broadcast flow (Home spec, Section [D]). **On this screen, tapping a chip filters the results list in place** — same visual component, different underlying behavior appropriate to browsing versus broadcasting; this is worth flagging explicitly for implementation since a shared chip component with two different tap-handlers (by screen context) is the efficient way to build this, not two visually distinct chip styles.
**Color:** selected chip filled Primary orange with White text; unselected Gray-100 background, Text-primary text — identical to every other chip-selection instance across this series.
**Animation:** background transition on selection, `duration-instant`, `ease-standard` — identical to the Home/Shortlist chip patterns already established.

### [C] Radius control (compact)
**Purpose:** the second required query parameter (`radiusMeters`), presented in a lighter-weight form than the New Broadcast screen's full slider treatment, since this screen doesn't need the same prominence for radius that a broadcast does — browsing is more exploratory, adjusting radius here is a secondary refinement, not a primary decision.
**Layout:** single compact row below the category chips — a small "Within **[X]**" text (Body, Text-primary) with an inline chevron, tapping opens a lightweight bottom-sheet with the same quick-pick chips (500m · 1km · 2km · 5km) used on the New Broadcast screen's radius selector — **reused component**, presented via a sheet here rather than inline, since this screen has more vertical content (a whole results list) competing for space than the focused New Broadcast form did.
**Default:** 1km, matching the New Broadcast screen's own default, for consistency of expectation across the app.
**Animation:** bottom-sheet slide-up (`duration-standard`, 240ms, `ease-out-soft`) on tap, same sheet-transition pattern used for the New Broadcast screen's manual-address expansion.

### [D] Results list
**Purpose:** the main content — a distance-sorted, category-filtered list of currently-open, in-hours, approved shops.
**Card component:** **near-total reuse of the Shortlist screen's shop card** (Shortlist spec, Section 3[D]) — same anatomy: photo/initials-avatar, name, category tag + open-status (always "Open now" here, per Section 0's server-side filtering guarantee — a "Closed" state literally cannot appear in this list's results, unlike the Shortlist screen, which is a meaningful simplification worth noting), distance, star rating (or "New to NEARBY"), reliability badge.
**The one card difference from the Shortlist screen:** **no "Select this shop" button, and no "Top pick" badge.** Neither applies here — there's no active request to select a shop into, and there's no server-side ranking-by-suitability the way the Shortlist's rating→reliability sort represents (this list is sorted purely by distance, a navigational fact, not a recommendation ranking) — badging a "top pick" on a pure-distance sort would misleadingly imply a judgment the sort itself doesn't make. Tapping the card body opens the **Shop Profile screen** (already built) — but see the note below on that screen's own adjustment when reached from here.
**Shop Profile screen adjustment when reached from Direct Search:** the Shop Profile spec's Section [H] ("Select this shop" button + confirmation modal) is **omitted entirely** in this entry path — there's no request for the customer to select a shop into. The screen still shows everything else (photo, rating, reliability, address, hours, [G] Call) unchanged; only the bottom fixed-action area differs, and in this context it simply isn't rendered, letting the Contact row's `tel:` link and the Address card's "Get directions" link (already present in that screen's Sections [E]/[G]) serve as this path's natural terminal actions instead.
**Pagination:** infinite-scroll, same `?page=&limit=20` convention as every paginated list in this product (§5.2) — server-guaranteed distance sort is preserved across pages, so no client-side re-sort is ever needed here, same principle already established for the Shortlist screen's server-determined ordering.
**Animation:** cards fade in + translateY(10px→0) over `duration-standard` with `stagger-step` (40ms) on initial load/filter-change — same list-reveal pattern reused from the Shortlist and Request History screens; re-fires on category/radius changes (a fresh filtered result set counts as a new "initial load" for animation purposes), but not on subsequent infinite-scroll pages, consistent with the Request History screen's identical rule.

### [E] Empty state *(replaces [D] when the filtered query returns zero shops)*
**Copy:** "No shops found" (H2, Text-primary) + "Try a different category or a wider radius." (Body, Text-secondary) — directly actionable, per the empty-state principle already applied twice in this series (never a bare "no results").
**Icon:** the same outline search/radar glyph used on the Request History screen's true-empty state (Text-secondary, 64×64) — visually consistent across the two screens that share a "nothing matched" meaning.
**Action:** a small "Widen radius" text-button (Caption, Primary orange) that opens the same radius bottom-sheet as Section [C], pre-highlighting the next tier up — a direct, one-tap fix for the single most common cause of an empty result in a small pilot locality (narrow radius + a less-common category).
**Animation:** icon fades in (`duration-standard`, `ease-out-soft`) — same restrained treatment as every empty state in this series.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | Selected category chip, "Change" location link, "Widen radius" action |
| Success | `#00B8A9` | "Open now" status text on every card (guaranteed present, per Section 0), "Usually reliable" badge |
| Text primary | `#14213D` | Header title, radius-control text, shop names |
| Text secondary | `#5B6472` | Location subtext, unselected chip text, distance/rating text, empty-state copy |
| Border | `#D8DCE3` | Card outlines, header scroll-divider |
| Surface | `#F4F5F7` / `#FFFFFF` | Unselected chip background, page background (Gray 100) / cards, header (White) |

No new colors and no Danger/Warning tokens anywhere on this screen — there's nothing cautionary or erroneous to represent; every result shown is, by construction, currently visitable.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Location subtext / "Change" link | Caption / 13px | 400 Regular |
| Category chip labels | Caption / 13px | 400 Regular |
| Radius-control text | Body / 15px | 400 Regular |
| Card shop name | Body / 15px | 600 SemiBold |
| Card category/status/distance/rating text | Caption / 13px | 400 Regular |
| Empty-state headline | H2 / 18px | 600 SemiBold |
| Empty-state body/action | Body / 15px / Button / 15px | 400 Regular / 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Chip selection, tap-scale feedback |
| `duration-standard` | 240ms | Radius bottom-sheet slide-up, results-list entrance stagger (on load/filter change), empty-state icon fade-in |
| `stagger-step` | 40ms | Delay between result cards on load/filter-change |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Chip transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Sheet slide-up, card entrance, empty-state entrance |

**No looping animation anywhere on this screen** — same reasoning as the Shortlist and Request History screens: this is browsable, settled information, not a live/real-time state.
**Reduced motion:** covered by the app-wide query, same degradation pattern (instant appearance, no stagger/slide) as every other screen in this series.

---

## 7. Images, icons & video — asset list

**No video.** Shop photos appear here exactly as they do on the Shortlist screen (thumbnail-scale, shop-supplied).

| Asset | Format/size | Source |
|---|---|---|
| Shop photo thumbnail / initials-avatar fallback | 56×56, rounded 8px | Reused, identical to the Shortlist card asset |
| Star-rating icon | 14×14, gold/amber tone | Reused from the Shortlist/Shop-Profile spec exception |
| Search/radar glyph (empty state) | Outline icon, 64×64, Text-secondary | Reused from the Request History screen's true-empty state |
| Back-chevron icon | Line icon, 24×24 | Icon library |
| Chevron (radius-control disclosure) | Line icon, 16×16, Text-secondary | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [A] Location | Carried over from Home's already-resolved location; "Change" reuses the existing Location-Confirm screen |
| [B]/[C]/[D] | `GET /api/v1/shops/nearby?lat=&lng=&radiusMeters=&category=&page=&limit=20` (Public — no auth header strictly required, though the app can still send one if the customer is signed in, with no behavioral difference) — category omitted entirely from the query when "All" is selected |
| [D] tap-through | Navigates to the Shop Profile screen with the omission noted in Section 3[D] |
| [E] "Widen radius" | Opens the same radius bottom-sheet as [C], client-side only |

**Loading state:** initial load and every filter/radius change shows 3–4 skeleton cards (gray placeholder blocks matching the real card's dimensions, shimmer) while the request resolves; infinite-scroll pagination shows a small inline spinner at the list's bottom edge rather than a full skeleton reset — identical pattern to the Request History screen.
**Error state:** a failed fetch uses the shared toast/error-boundary pattern (§6.1) with retry; since this is a Public endpoint, an auth-related error should never occur here, simplifying this screen's error surface relative to most others in the product.

---

## 9. Responsive & platform notes

- **Viewport:** standard vertical list, max-width 480px centered, consistent with every screen in this series; category chips scroll horizontally rather than wrapping, same treatment as the Request History screen's filter row.
- **No map view in v1** — per Section 0's NN/g-backed rationale; if the team later wants a map toggle, it would sit as a small icon-button in the header (Section [A]) switching the results area to a Leaflet/OSM map (reusing the same free/keyless mapping stack already committed to for the Location-Confirm screen) with simple, unclustered pins — explicitly deferred rather than built now, since the research doesn't support it being a priority for launch.
- **Pilot-scale expectation:** with 20–50 hand-onboarded shops total in a single locality (Master Build Doc §1), most category/radius combinations will return a short, easily-scannable list — infinite scroll is more than sufficient, no virtualization needed, consistent with every other list screen's sizing note in this series.
- **PWA standalone mode:** no bottom tab bar shown if this screen is reached as a drill-through from Home (matches the "focused task" treatment used for New Broadcast); if the team instead makes this a permanent tab-bar destination (an alternative entry-point design worth considering given how distinct this browsing mode is from the rest of the app), it would carry the tab bar the same way the Request History screen does — this is a product decision the team should make deliberately rather than defaulting silently.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 2.7, 5.2, 5.3, FR-1.15, UC-20) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). This screen is built almost entirely from components already specced elsewhere in this series (the Shortlist card, the Shop Profile screen, the radius quick-picks, the category chips, the Request History empty state) with the differences called out explicitly rather than silently — the goal is a screen that feels native to the rest of NEARBY, not a bolted-on separate "search" product inside it.*
