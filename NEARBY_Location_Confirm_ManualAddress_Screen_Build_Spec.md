# NEARBY — Location Confirm / Manual Address Screen Build Specification
### Everything needed to build the manual-location fallback screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1, 5.4, 6.2), NEARBY Brand/PWA Guide v1.0, Uber/Google's fixed-pin map-confirmation pattern research, and free/keyless mapping stack research (Leaflet + OpenStreetMap, matching Nominatim's own free-tier)

---

## 0. Scope — where this screen fits and what makes it different from every competitor's version

This screen exists **only because §6.2 of the Master Build Document requires it** — it's the fallback path when `navigator.geolocation.getCurrentPosition()` is denied, times out, fails, or is unavailable. It has one job: turn free-text into a confirmed `{latitude, longitude}` pair the customer has actually looked at and approved, via `POST /api/v1/geo/lookup` (backed by Nominatim, rate-limited 10/IP/hour, §5.4).

Three things make this different from Uber/Swiggy's equivalent screen, and the spec below is built around them:
1. **No live-tracking, no driver, no delivery ETA** — this is a one-time location capture for a broadcast, not an ongoing pickup/drop-off point. The screen should feel *lighter* than a ride-hailing pin-confirm screen, not equally weighted.
2. **Free geocoding only (Nominatim), no Google Maps/Mapbox key** — the Master Build Doc's whole stack is chosen for zero cost (§3, throughout). This screen's map rendering must follow that same constraint: **Leaflet + OpenStreetMap tiles**, not Google Maps — both are free/keyless and match Nominatim's own no-API-key philosophy exactly.
3. **This is a fallback, not the primary path** — §6.2 is explicit that this screen only appears on denial/failure, or when the customer explicitly asks to change a successfully-auto-resolved location. It should never be the default first thing a customer sees.

**Relationship to the New Broadcast screen:** the prior build spec's Section [E] describes this content as an *inline expansion* within the broadcast screen for the common case. This document specifies the **full, dedicated version** of that same flow — used either as that inline expansion's fully-expanded state, or as its own presented screen/modal if the team prefers a cleaner separation. Either implementation path uses the exact same sections, copy, and behavior below; only the container differs.

---

## 1. Page goals, in priority order

1. Let the customer type an approximate address in the least effortful way possible and get a geocoded result back.
2. Show that result **on a map, with a pin**, so the customer visually confirms it's the right spot — never accept a geocoded address blind (§6.2 requires explicit confirmation, not silent acceptance).
3. Let the customer fine-tune the exact point if the geocoded address is close but not quite right (matches the "drag pin" affordance that Uber, Google, and Mapbox all default to for this exact kind of screen).
4. Fail informatively, never silently fall back to IP-based location or any other guess (§6.2's explicit prohibition).

---

## 2. Layout overview

Presented as a full-height view (either its own screen or the broadcast modal's expanded state — Section 0), map-forward layout since the whole point of this screen is visual confirmation, not more form-filling.

```
┌─────────────────────────────┐
│ [A] Header (back + title)    │
├─────────────────────────────┤
│ [B] Address search input     │  ← sticky top, over the map
├─────────────────────────────┤
│ [C] Map with fixed center pin │  ← fills remaining space
├─────────────────────────────┤
│ [D] Address confirmation card │  ← floats over map, bottom
├─────────────────────────────┤
│ [E] Confirm button             │  ← inside card D
└─────────────────────────────┘
```

**Pin interaction model — fixed-pin, moving-map** (not a draggable-pin-on-a-static-map): per the pattern research above, Uber, Google, and Mapbox all default to keeping the pin visually fixed at the screen's center and letting the *map* pan underneath it. This is the better choice for NEARBY too — it's the simpler implementation (no drag-gesture/hit-testing on the marker itself, just standard map-pan handling) and it's the pattern customers are most likely to already know from Uber/food-delivery apps.

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, back-chevron left (24×24, Text-primary), centered title.
**Copy:** "Confirm your location"
**Background:** White, sits above the map (not transparent-over-map — keeps the back action reliably tappable regardless of map content underneath).
**Animation:** none.

### [B] Address search input
**Purpose:** entry point for the free-text address (this is the field wired to `POST /api/v1/geo/lookup`).
**Copy (placeholder):** "Street, area, or landmark"
**Layout:** full-width rounded input, 48px height, 12px radius, White background, 1px Border-gray outline, search icon left, sits in its own sticky bar directly below the header (distinct from the map below it — a 1px Border-gray divider separates the two).
**Behavior:** submits on keyboard "search"/enter, or via an inline trailing "Search" text-button (Caption, Primary orange) inside the field — not a separate full-width button, since this is a single-field lookup, not a form with multiple inputs.
**Loading state:** while the geocode request is in flight, the trailing action swaps to a small spinner; the field itself stays editable (it's a quick lookup, not a blocking multi-second process).
**Color:** default border Border-gray; focus border Primary orange 2px.
**Animation:** border-color transition on focus, `duration-instant` (100ms), `ease-standard`.

### [C] Map with fixed center pin
**Library/tiles:** **Leaflet + OpenStreetMap tile layer** (`https://tile.openstreetmap.org/{z}/{x}/{y}.png`, with the required OSM attribution string rendered in a corner per OSM's tile usage policy) — free and keyless, the only mapping choice consistent with the rest of the Master Build Document's zero-cost stack (Nominatim, Netlify, Render free tiers, etc.). React-Leaflet is the recommended wrapper for a React/Vite codebase.
**Initial center:** the geocoded result's `{latitude, longitude}` after a successful [B] search, at zoom level 16 (close enough to distinguish individual streets/buildings, per Leaflet's standard "neighborhood" zoom convention) — or, if this screen was reached directly from a failed auto-geolocation attempt with no search yet performed, centers on the pilot locality's default coordinate (a sensible town-center fallback, never a random/arbitrary point).
**Pin:** a single static marker rendered at the exact visual center of the map viewport (CSS-positioned, not a Leaflet `Marker` object tied to a lat/lng — since the map pans underneath it, the pin itself never moves on screen). Uses the NEARBY logomark's pin motif (Brand Guide §2.1 — "the pin is the shop being found," reused here for "the pin is where you are") rendered in Primary orange, with a small drop-shadow beneath its point for ground-contact depth (the *second* deliberate shadow use in the whole product, after the radius-slider thumb in the New Broadcast spec — both are for the same reason: an object that visually needs to read as "sitting on/pointing at" something below it).
**Interaction:** standard pan (drag) and pinch-zoom, exactly as Leaflet provides by default; every pan-end triggers a **reverse-geocode-on-settle** — after the map stops moving, silently call `POST /api/v1/geo/lookup` in reverse (coordinates → address) to update the confirmation card's text, so the address shown always matches where the pin is actually pointing, not just the last text search.
**"Recenter to my location" button:** small circular floating action button, bottom-right of the map (above the confirmation card), compass/crosshair icon, White background with subtle shadow — re-attempts `navigator.geolocation.getCurrentPosition()` on tap, offering one more chance to succeed even though the customer arrived here via a failure (a transient GPS glitch is common; this button costs nothing to include and can skip the rest of this screen entirely if it succeeds).
**Color:** map tiles render as-is (OSM's standard cartography — not recolored, since retinting map tiles is both a licensing gray area and unnecessary effort); pin in Primary orange; recenter-button icon in Text-primary.
**Animation:** map pan/zoom uses Leaflet's own built-in easing (no custom override needed — reinventing this would fight the library, not complement it); the pin has **no bounce/drop animation on initial load** — it should simply be there, present and static, matching the calm-tone rule (Brand Guide §1.4) rather than the more playful drop-and-bounce pin animations common in consumer map apps.

### [D] Address confirmation card
**Purpose:** the actual text the customer is confirming — the map is context, this card is the decision.
**Layout:** floats above the bottom of the map, full-width minus 16px side margins, White background, rounded-top-only 16px radius (visually "docked" to the bottom edge), subtle top shadow to lift it off the map, sits above the safe-area inset.
**Content:**
- Pin icon (16×16, Primary orange) + the resolved `formattedAddress` from the most recent geocode (forward or reverse), Body size, max 2 lines with ellipsis overflow.
- Directly below, Caption-size secondary line: "Drag the map to adjust" — a persistent hint, not a one-time tooltip, since this is the core interaction the whole screen exists to support and it should never disappear.
**Color:** Text-primary for the address, Text-secondary for the hint line.
**Animation:** the address text cross-fades (`duration-quick`, 180ms) whenever it updates from a pan-settle reverse-geocode — never a hard swap, since this text can change without an explicit user tap (map settling triggers it), and an abrupt change would be easy to miss or, worse, jarring.

### [E] Confirm button
**Layout:** inside card [D], full-width, 48px height, 12px radius, sits directly below the hint line.
**Copy:** "Use this location"
**States:** identical pattern to every other primary CTA in this product — Primary orange default, Primary-pressed on press, Gray 300/disabled only in the edge case where no address has resolved yet at all (e.g., the very first paint before any search or GPS attempt has completed) and tap-scale-to-0.97 on interaction.
**Behavior:** on tap, the current pin-center coordinates (whatever the map has most recently reverse-geocoded, whether from a text search or a manual pan) are locked in as the request's `location`, and the customer is returned to the New Broadcast screen (or, if this screen was reached from the Home page's "Change" location link, back to wherever that link was tapped from) with the confirmed address now shown in that screen's Location status row.
**Animation:** on confirm, cross-fades the screen away (`duration-standard`, 240ms, `ease-out-soft`) back to the calling screen — same handoff pattern used consistently across every screen transition in this product.

### No-match / error state (replaces [C]/[D] content, doesn't add a new section)
**Trigger:** [B]'s search returns no geocode result from Nominatim.
**Copy (shown inline below the search field, map area shows a neutral empty-map state at the locality's default center rather than a broken/blank view):** "Couldn't find that address — try adding more detail, like a nearby landmark or area name."
**Behavior:** input remains focused and editable for another attempt; [E]'s confirm button is disabled until a valid geocode (forward or reverse) exists. **Never** silently substitutes an IP-based or default location — this is a hard requirement carried over from §6.2 of the Master Build Document, and the UI should make failure to resolve a dead-end that requires another explicit attempt, not something that quietly proceeds.
**Rate-limit case** (10/geo-lookup calls/IP/hour exceeded, §5.4): shown with the same generic copy pattern as the Sign-Up page's password-reset rate limit (prior build spec) — never expose the specific limit number or "rate limited" language to the customer; instead: "We're having trouble looking that up right now — try again in a few minutes." (a truthful but non-technical framing that doesn't invite the customer to immediately hammer retry).

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | Map center pin, focus border on search input, confirm button, pin icon in confirmation card |
| Primary-pressed | `#E24521` | Confirm button pressed state |
| Text primary | `#14213D` | Header title, confirmed address text, recenter-button icon |
| Text secondary | `#5B6472` | "Drag the map to adjust" hint, no-match error copy |
| Border | `#D8DCE3` | Search input outline, header/map divider |
| Surface | `#F4F5F7` / `#FFFFFF` | Empty-map background fallback (Gray 100) / header, input, confirmation-card, recenter-button backgrounds (White) |

**No Danger-red used here**, deliberately — the no-match state above uses Text-secondary, not Danger, because a not-yet-found address is a normal, expected part of using a free-text geocoder (Nominatim can miss on ambiguous input), not an error condition worth the same visual alarm as a form-validation failure elsewhere in the product.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Search input text/placeholder | Body / 15px | 400 Regular |
| Confirmed address (card D) | Body / 15px | 400 Regular |
| "Drag the map to adjust" hint | Caption / 13px | 400 Regular |
| No-match error copy | Caption / 13px | 400 Regular |
| Confirm button label | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Search-input focus border, confirm-button tap-scale |
| `duration-quick` | 180ms | Confirmation-card address text cross-fade on pan-settle update |
| `duration-standard` | 240ms | Screen-transition handoff back to the calling screen on confirm |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Focus-state transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Post-confirm screen handoff |
| *(Leaflet's built-in pan/zoom easing)* | library default | Map movement — deliberately not overridden |

**Explicitly no motion on:** the pin itself (no drop/bounce on load, no idle animation) — matches the calm-tone rule and distinguishes it from the broadcast-pulse animation elsewhere in the app, which specifically signals "something live is happening"; this pin is static by design, since nothing here is a real-time event.
**Reduced motion:** the app-wide `prefers-reduced-motion` query (§9.5) covers the confirmation-card cross-fade and screen-transition; Leaflet's own pan/zoom easing is a direct-manipulation response to touch/drag input, not a decorative animation, so it isn't gated by this query (consistent with how the radius slider's drag was treated in the New Broadcast screen spec).

---

## 7. Images, icons & video — asset list

**No video, no photography.**

| Asset | Format/size | Source |
|---|---|---|
| Map tiles | OpenStreetMap raster tiles via Leaflet's `TileLayer`, `https://tile.openstreetmap.org/{z}/{x}/{y}.png` | Free, keyless, per OSM's tile usage policy (attribution required and rendered on-map, standard Leaflet behavior) |
| Center pin | Reuses the NEARBY logomark's pin shape (Brand Guide §2.1), rendered ~32×40px, Primary orange, with a small drop-shadow | Derived from `nearby-mark.svg`'s pin/base shape — no new source file needed, extract just the pin geometry |
| Back-chevron icon | Line icon, 24×24 | Icon library (same one used app-wide) |
| Search icon | Line icon, 20×20 | Icon library |
| Recenter/crosshair icon | Line icon, 20×20, Text-primary | Icon library |
| Pin icon (in confirmation card, small) | Line icon, 16×16, Primary orange | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B] Address search (forward geocode) | `POST /api/v1/geo/lookup` with `{ address: <text> }` → `{ latitude, longitude, formattedAddress }` (Public, rate-limited 10/IP/hour, §5.4) |
| [C] Pan-settle (reverse geocode) | Same `POST /api/v1/geo/lookup` endpoint, reverse mode (`{ latitude, longitude }` → `{ formattedAddress }`) if the backend supports it as a single endpoint with dual direction, or a documented reverse-specific variant — **counts against the same 10/IP/hour limit**, so debounce pan-settle calls (e.g., only fire ~600ms after the map stops moving, not on every frame of a drag) to avoid burning the rate limit on ordinary map exploration |
| [C] Recenter button | `navigator.geolocation.getCurrentPosition()`, same as the primary path in §6.2 — a successful result here can skip the rest of this screen and return the customer directly to the calling screen, same as if geolocation had succeeded the first time |
| [E] Confirm | No separate endpoint — the coordinates are held in client state and passed back to whichever screen invoked this one (New Broadcast, or Home's location-pill "Change" link) |

**Loading state:** [B]'s inline spinner (Section 3) covers the forward-geocode call; the map itself has no loading skeleton (Leaflet's tile loading is handled natively by the library — gray tile placeholders while imagery loads in, standard behavior, no custom treatment needed).
**Error state:** the no-match and rate-limit copy (Section 3, "No-match / error state") are the only error states this screen needs — both handled inline, never via the shared toast pattern, since address resolution is this screen's entire purpose, not an incidental background task.

---

## 9. Responsive & platform notes

- **Viewport:** the map fills all available vertical space between the sticky search bar [B] and the floating confirmation card [D] — this is the one screen in the product where a genuinely full-bleed, edge-to-edge layout is correct, since a cramped map is actively harder to use for panning/zooming.
- **Permissions:** if the customer taps "Recenter to my location" and geolocation is denied *again* (they already declined once to reach this screen), don't re-prompt the OS permission dialog on every tap — after one repeat denial in the same session, disable the recenter button with a brief inline note ("Location access is off — enable it in your browser/device settings to use this") rather than repeatedly triggering a permission prompt the customer has already said no to.
- **OSM tile usage policy:** since this is a live pilot product (not a prototype), the standard `tile.openstreetmap.org` endpoint is fine at pilot scale (20–50 shops, single locality) but its usage policy technically expects moderate, non-bulk traffic — worth a Section-9-style monitoring note (consistent with how the Master Build Doc already flags Cloudinary/Render caps in its own Section 9) if usage ever scales meaningfully beyond the pilot.
- **PWA standalone mode:** this screen's header [A] is its only chrome (no bottom tab bar, since it's a focused sub-task reached from either the New Broadcast modal or Home's location pill) — back-gesture/back-button should return to whichever of those two contexts launched it, discarding any unconfirmed map position (nothing here is auto-saved, consistent with the no-persistence rule already established for the New Broadcast screen's inputs).

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 5.4, 6.2) and NEARBY Brand/PWA Guide v1.0 (Sections 2, 3–5, 9). The Leaflet/OpenStreetMap mapping choice is not incidental — it's the only option consistent with the zero-cost, keyless-geocoding stack (Nominatim) already committed to elsewhere in the Master Build Document; introducing a Google Maps or Mapbox dependency here would contradict that stack decision even though neither source document names a mapping library explicitly.*
