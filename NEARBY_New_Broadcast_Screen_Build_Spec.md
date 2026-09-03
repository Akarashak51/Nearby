# NEARBY — New Broadcast (Search) Screen Build Specification
### Everything needed to build `/app/broadcast` (or equivalent route) — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1, 3.2, 3.11, 5.3 validation, 6.2), NEARBY Brand/PWA Guide v1.0, and radius/location-picker UI pattern research

---

## 0. Scope & what makes this screen different from a competitor's "search" screen

This is the single most important screen in the product — it's the one action the whole Home page (previous spec) is built to funnel a customer into within one tap. But it is **not** a product-search screen like Blinkit/Zepto's — there's no catalog to autocomplete against, no results grid to render inline. This screen does exactly one thing: **collect three inputs (product query, radius, location) and fire a single broadcast.** Everything on it should read as a short, confident form — not a browsing experience.

Three hard constraints from the Master Build Document govern every decision below:
1. **`radiusMeters` is required, validated server-side to 100–5000** (§5.3) — the radius control must only be able to produce values in that range; nothing in the UI should let a customer submit outside it.
2. **`productQuery` is free-text**, case-folded/trimmed for analytics (§5.4) — no structured category/SKU picker, no autocomplete against a live catalog (none exists).
3. **Location comes from one of exactly two paths** (§6.2): silent success via `navigator.geolocation.getCurrentPosition()`, or a manual-address fallback through `POST /geo/lookup` on denial/failure/timeout. The screen's layout must accommodate either path appearing, but never show both a location error *and* let the customer submit — submission is blocked until a location exists.

---

## 1. Page goals, in priority order

1. Get location resolved (silently, ideally) before the customer even finishes typing their product query — location resolution should run in parallel with typing, not block it.
2. Make the radius control fast to set and impossible to set wrong (only ever produces valid 100–5000m values).
3. Keep the whole screen a single scroll-free view on a standard phone viewport if at all possible — this is a "fire and go" screen, not a wizard.
4. Communicate, calmly, exactly what happens next ("shops nearby will be notified") so the customer isn't surprised by the live-count screen that follows — sets up the Home-page Active Request banner correctly.

---

## 2. Layout overview

Single screen, no internal scroll on standard viewports (360–428px width), modal-style presentation (slides up from Home, per Brand Guide §9.3 screen-transition pattern) rather than a full route replace — reinforces that this is a quick, focused action, not a new "place" in the app.

```
┌─────────────────────────────┐
│ [A] Header (back + title)    │
├─────────────────────────────┤
│ [B] Product query input      │
├─────────────────────────────┤
│ [C] Radius selector          │
├─────────────────────────────┤
│ [D] Location status row      │  ← state-dependent
├─────────────────────────────┤
│ [E] Manual address fallback  │  ← conditional
├─────────────────────────────┤
│ [F] What happens next strip  │
├─────────────────────────────┤
│ [G] Broadcast button          │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, left-aligned back-chevron icon (24×24, Text-primary), centered title, no right-side action.
**Copy:** "New search"
**Background:** White, no border/shadow (this is a modal sheet, its own rounded-top-corner container already separates it from Home visually).
**Animation:** none.

### [B] Product query input
**Purpose:** the one required creative-text field on this screen.
**Copy (label, floats above on focus):** "What do you need?"
**Placeholder (shown when empty, before focus):** "e.g. \"AA batteries\", \"paracetamol 500mg\", \"phone charger\""
*(Rationale for a multi-example placeholder rather than one item: since there's no catalog to browse, the placeholder is doing the job Blinkit's category grid does elsewhere — showing the breadth of things this field accepts, in three concrete examples spanning different shop categories, so a first-time user isn't left guessing whether this is only for groceries.)*
**Layout:** full-width input, 52px height, 12px radius, Surface white background, 1px Border-gray outline, auto-focused on screen open (keyboard opens immediately — this is the field that matters most, per the "get to the action fast" goal).
**Character limit:** soft cap ~80 characters, no hard block — a helper caption appears only past 60 characters: "Keep it short — shops scan this in seconds."
**Validation:** required; empty-submit attempt shakes the field horizontally (±4px, 2 cycles, 150ms total) and shows "Tell us what you're looking for" in Danger red beneath it — no silent-disable-only pattern, since a disabled button with no explanation is a common frustration point in form UX research.
**Color:** default border Border-gray; focus border Primary orange 2px; error border Danger red.
**Animation:** border-color transition `duration-instant`; error shake as specified above (one-time, not looping).

### [C] Radius selector
**Purpose:** bounded, fast, can't-go-wrong distance control — mapped 1:1 to `radiusMeters`, 100–5000m server validation (§5.3).
**Pattern chosen:** a **segmented quick-pick row + expandable slider**, not a slider-only or dropdown-only control — location-search UI pattern research shows sliders are the most common interactive pattern for bounded-range radius controls, but a slider alone forces every user to drag for even the most common values, so quick-pick chips for the most-likely distances sit above a slider for fine control.
**Layout:**
- Row of 4 quick-pick chips: **500m · 1km · 2km · 5km** — rounded-full pills, 36px height, tapping one sets the value immediately and updates the slider position to match.
- Below the chips, a horizontal slider (track 100–5000m, default thumb position at 1000m matching the "1km" chip), with the live numeric value displayed above the thumb as it drags (e.g. "1.2 km"), rounded to the nearest 100m for display and for the value actually sent, keeping the input tidy without pretending to false precision.
- Slider track: Gray 100 background, filled portion (0 to thumb) in Primary orange; thumb a white circle with a 2px Primary-orange border and subtle drop shadow (the *one* deliberate shadow use on this screen — a slider thumb needs a clear "this is draggable" affordance, unlike everything else in the brand system which stays flat).
**Copy (section label):** "Search radius"
**Helper caption below the control:** "Only shops within this distance will be notified" — sets accurate expectations, avoids the customer thinking a larger radius means "more results shown," when actually it means "more shops asked."
**Color:** filled track Primary orange (#FF5A36); unfilled track Gray 100 (#F4F5F7); active/selected quick-pick chip Primary-orange background with White text, unselected chips Gray 100 background with Text-primary text.
**Animation:** chip-selection background transition `duration-instant`; slider thumb drag is 1:1 with touch/pointer position (no easing — a slider must feel directly manipulated, easing here would feel laggy); numeric value label above the thumb updates instantly, no count-up easing (that pattern is reserved for the live response-count elsewhere, per Brand Guide §9.2 — using it here would blur its meaning as "something happening in real time" when this is just a static input read-out).

### [D] Location status row
**Purpose:** transparently shows what location will be used, without making the customer manage it in the common case.
**Layout:** single row, pin icon (16×16) + status text, Caption size, sits below the radius control.
**States:**
1. **Resolving** (0–10s, geolocation call in flight): pin icon in Text-secondary, text "Finding your location…" with a small inline shimmer/pulse on the icon only (not a full skeleton — this is one line of text, a shimmer block would be disproportionate).
2. **Resolved (auto)**: pin icon in Success green, text "Using your current location" + a small "Change" text-link (Caption, Primary orange) that opens section [E] manually even on the success path, in case the customer wants to search near a different spot than where they're standing.
3. **Denied/failed** (`PermissionDenied`, `Timeout`, `PositionUnavailable`, or API absent — §6.2): pin icon in Danger red, text "Couldn't get your location" — section [E] expands automatically directly below this row.
**Animation:** state transitions cross-fade (`duration-quick`, 180ms) rather than hard-swap, since this row's text changes without user action (the geolocation call resolving in the background) and an abrupt swap would be jarring mid-read.

### [E] Manual address fallback *(shown only on denial/failure, or when the customer taps "Change" from state 2 above)*
**Layout:** expands directly below [D] with a standard-transition height animation (not an instant snap — `duration-standard`, 240ms, `ease-out-soft`), full-width text input + inline "Confirm" button.
**Copy (label):** "Enter your address instead"
**Placeholder:** "Street, area, or landmark"
**Behavior:** on submit, calls `POST /api/v1/geo/lookup` (rate-limited 10/IP/hour per §5.4) → returns `{ latitude, longitude, formattedAddress }`. On success, the resolved `formattedAddress` replaces the input with a small confirmation chip ("📍 [formattedAddress] — looks right?") with Edit/Confirm actions, before it's locked in as the request's location — the customer confirms the geocoded result rather than it being silently accepted, per §6.2's explicit requirement.
**No-match state:** if Nominatim returns no result, inline error directly below the input: "Couldn't find that address — try adding more detail (area or landmark)" and the input remains editable for another attempt. **Does not** silently fall back to IP-based location (§6.2 is explicit that this must never happen) — the [G] broadcast button stays disabled until a location is actually confirmed by one of the two documented paths.
**Color:** input styled identically to [B]; Confirm button small, Primary orange, inline (not full-width — this is a secondary in-flow action, not the screen's main CTA).

### [F] What happens next strip
**Purpose:** manages expectations before submission — the one piece of pure explanatory copy on this screen, kept to a single line so it doesn't compete with the form.
**Layout:** thin full-width bar, Gray 100 background, small radiating-signal icon (reuses the logomark's arc motif, matching the Home page's How-It-Works icon choice) + one line of Caption-size text.
**Copy:** "Nearby open shops will be notified instantly — you'll see responses live for the next 2 minutes." *(the "2 minutes" reflects the default 120s response window, §3.2 — if a locality's admin-configured window ever differs from 120s per §3.11, this copy should read the actual configured value rather than a hard-coded "2 minutes"; treat the string as a template, not a literal.)*
**Color:** icon in Primary orange; text in Text-secondary.
**Animation:** none — static informational content.

### [G] Broadcast button
**Layout:** fixed to the bottom of the screen (not inline in the scroll flow), full-width minus 16px side margins, 52px height, 12px radius, sits above the safe-area inset on notched devices.
**Copy:** "Broadcast to nearby shops"
**States:**
- Disabled (missing product query OR no confirmed location yet): Gray 300 background, Text-secondary text, non-interactive. Tapping a disabled button still triggers the relevant field's validation shake (Section [B]) rather than doing nothing, so the customer always gets feedback.
- Enabled: Primary orange background, White Button-weight text.
- Pressed: Primary-pressed background, tap-scale-to-0.97.
- Loading (submit in flight — request document being created server-side): label replaced by centered spinner, button width unchanged.
**Animation:** on successful submission, the entire screen cross-fades into the Home page's now-populated Active Request banner state (`duration-standard`, `ease-out-soft`) — consistent with every other screen-to-screen handoff in the product (same pattern used on Sign-up success and `request:selected`, per the prior two build specs), rather than the modal simply "closing."

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | Radius slider fill + thumb border, active quick-pick chip, focus borders, broadcast button, "Change" link, "what happens next" icon |
| Primary-pressed | `#E24521` | Broadcast button pressed state |
| Success | `#00B8A9` | Location-resolved pin icon |
| Danger | `#E5484D` | Location-denied pin icon, validation error text/border, no-match address error |
| Text primary | `#14213D` | Header title, input text, unselected chip text |
| Text secondary | `#5B6472` | Placeholder text, helper captions, resolving-state text |
| Border | `#D8DCE3` | Input outlines |
| Surface | `#F4F5F7` / `#FFFFFF` | Unfilled slider track / unselected chips (Gray 100), input and header backgrounds (White) |

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Product query label/input | Body / 15px | 400 Regular |
| Radius section label | H2 / 18px | 600 SemiBold |
| Quick-pick chip labels | Caption / 13px | 400 Regular |
| Slider live value ("1.2 km") | Body / 15px | 600 SemiBold (bumped weight for this one read-out, since it's the number the customer is actively watching while dragging) |
| Location status text | Caption / 13px | 400 Regular |
| Manual address input | Body / 15px | 400 Regular |
| What-happens-next copy | Caption / 13px | 400 Regular |
| Broadcast button label | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Input focus border, chip selection, tap-scale feedback |
| `duration-quick` | 180ms | Location-status row cross-fade between states |
| `duration-standard` | 240ms | Manual-address section expand, screen-transition into Home on submit |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Focus/chip transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Manual-address expand, post-submit handoff |
| Validation shake | ±4px, 2 cycles, ~150ms total, one-time | Product-query field on empty-submit attempt |

**Explicitly not used here:** the broadcast-pulse looping animation (Brand Guide §9.2) — that's reserved for the *live, in-progress* request state on Home/Request-detail, not this pre-submission form. Using it here would signal "broadcasting" before anything has actually been broadcast, which would be misleading.
**Reduced motion:** covered by the app-wide `prefers-reduced-motion` query — the validation shake and slider-drag are the two things worth double-checking degrade sensibly (shake → instant color-only error state; slider drag is direct-manipulation and inherently has no "motion" to disable).

---

## 7. Images, icons & video — asset list

**No video, no photography** — consistent with every other screen spec in this product; this screen in particular has the least reason of any screen to carry visual weight, since its entire job is "collect three inputs and go."

| Asset | Format/size | Source |
|---|---|---|
| Back-chevron icon | Line icon, 24×24 | Icon library (same one used app-wide) |
| Location pin icon | Line icon, 16×16, recolorable (Text-secondary/Success/Danger per state) | Icon library |
| Radiating-signal icon ("what happens next" strip) | Reuses logomark's arc motif, ~20×20 | Derived from `nearby-mark.svg`, no new asset (same reuse decision as the Home page's How-It-Works icons) |
| Slider thumb | CSS-drawn circle (white fill, 2px orange border, subtle box-shadow) — no image asset needed | — |
| Confirm-address checkmark (optional, inline in the confirmation chip) | Line icon, 14×14 | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B] Product query | Client-side state only until submit; sent as `productQuery` (trimmed) |
| [C] Radius | Client-side state, default 1000m; sent as `radiusMeters`, always an integer 100–5000 (slider/chips physically cannot produce out-of-range values, so no client-side range error state is needed — this is enforced by construction, not validated after the fact) |
| [D]/[E] Location | `navigator.geolocation.getCurrentPosition()` (10s timeout, §6.2) on screen mount; on failure, `POST /api/v1/geo/lookup` (Public, 10/IP/hour) on manual submit |
| [G] Submit | `POST /api/v1/requests` with `{ productQuery, radiusMeters, location }` → on success, server creates the `requests` document (`status: 'open'`, `expiresAt = now + responseWindowSeconds`) and the client transitions to the Home page's Active Request banner / dedicated tracking state |

**Loading states:** [D]'s "Finding your location…" pulse (Section 3 above) and [G]'s in-flight spinner are the only two loading states this screen needs — no skeletons, since nothing here is list/card data.
**Error states:** geo-lookup failures are handled inline within [E] (Section 3 above), never via the shared toast pattern — a location error is central to this screen's own task, not a background/incidental failure. A `POST /api/v1/requests` submission failure (network/500) *does* use the shared toast/error-boundary pattern (Master Build Doc §6.1), since that's an incidental infrastructure failure, not a validation issue.

---

## 9. Responsive & platform notes

- **Viewport:** designed to fit without internal scroll on a 360×740px reference viewport (a common small-Android baseline); on genuinely small/older devices where it doesn't fit, the screen scrolls normally with [G] remaining fixed to the bottom via `position: sticky`/`fixed`, never scrolling out of reach.
- **Keyboard handling:** product-query input auto-focuses on screen open (Section 3[B]) — ensure the fixed-bottom broadcast button doesn't get obscured by the on-screen keyboard on iOS Safari's PWA mode (a common bug with `position: fixed` + virtual keyboards); test specifically for this since it's the screen's primary CTA.
- **Slider accessibility:** the radius slider must be operable via arrow keys when focused (standard `<input type="range">` behavior) and expose its current value to screen readers (`aria-valuenow`/`aria-valuetext` reading "1.2 kilometers," not a raw meter integer) — the quick-pick chips double as an accessible alternative to dragging for anyone who finds the slider harder to operate precisely.
- **PWA standalone mode:** presented as a modal sheet over Home (Section 2) rather than a full route push, so the back-chevron in [A] and the OS/gesture back-action should do the same thing — dismiss back to Home with whatever was typed so far discarded (no draft-saving in v1, matching the Home page spec's note that a broadcast's inputs aren't persisted across sessions).

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 3.2, 3.11, 5.3, 5.4, 6.2) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). No color, motion token, validation range, or location-handling behavior introduced here contradicts either source document — the 100–5000m radius bound and the two-path-only location flow are both enforced by the control designs themselves, not left to after-the-fact validation.*
