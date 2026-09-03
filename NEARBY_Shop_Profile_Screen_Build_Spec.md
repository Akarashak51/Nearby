# NEARBY — Shop Profile (from Shortlist) Screen Build Specification
### Everything needed to build the shop-detail screen reached from a shortlist card — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1.1, 3.1.2, 3.4, 4.2–4.5), NEARBY Brand/PWA Guide v1.0, and restaurant/store detail-page pattern research (Zomato's product-page UX analysis, restaurant-website above-the-fold conventions)

---

## 0. Scope — what this screen is for, and the one thing it must never become

This is the "tap the rest of the card" destination from the Shortlist screen's dual-tap-target design (previous spec, Section 3[D]) — a customer who wants more than a card's surface-level info before committing taps through to here. It's NEARBY's equivalent of Zomato's restaurant page, but scoped down hard: **Zomato's own UX research flags restaurant pages as commonly over-cluttered** ("restaurant pages are often cluttered with excessive information, making it hard to scan"), and NEARBY has a much smaller, much more decision-focused job than a food-ordering product does — there's no menu to browse, no cart, no delivery-fee disclosure. This screen answers exactly one question: **"is this the shop I want to visit?"** — and every section below exists only in service of that question.

The universal above-the-fold convention from restaurant-website research — **photo, name/rating, and address+hours visible without scrolling** — is followed directly, but trimmed to what NEARBY actually needs, not padded out with menu/ordering content that doesn't apply here.

**Relationship to the Shortlist screen:** this screen doesn't duplicate the Select flow — it reuses the exact same "Select this shop" action and confirmation-modal pattern already specced for the Shortlist screen's cards, just presented at full-detail scale. A customer can select a shop from either screen; the behavior and copy are identical either way.

---

## 1. Page goals, in priority order

1. Confirm identity and trust signals fast — photo, name, rating, reliability — above the fold, per the restaurant-page research's "four things above the fold" rule (image, key info, primary action, location/hours).
2. Answer "can I actually get there and will they be open" — address, distance, hours, phone — without requiring a tap-through to a separate map screen.
3. Offer the same "Select this shop" action available from the Shortlist card, so this screen is a genuine alternative path, not a dead-end detour.
4. Stay short — this is a decision-support screen, not a content hub; if it doesn't help answer Goal 1's question, it doesn't belong here.

---

## 2. Layout overview

Single scrollable screen, photo-forward header (the one deliberate departure from every other screen's photo-free treatment in this series, and it's earned here — Zomato's own UX analysis specifically credits the featured-image-at-top layout for "catching the user's eye" on exactly this kind of detail page).

```
┌─────────────────────────────┐
│ [A] Header (back, floats over photo) │
├─────────────────────────────┤
│ [B] Shop photo (hero)         │
├─────────────────────────────┤
│ [C] Name, category, status row │
├─────────────────────────────┤
│ [D] Rating & reliability row  │
├─────────────────────────────┤
│ [E] Address & distance card   │
├─────────────────────────────┤
│ [F] Opening hours card        │
├─────────────────────────────┤
│ [G] Contact row (call)        │
├─────────────────────────────┤
│ [H] Select this shop button   │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header (floating over the photo)
**Layout:** transparent-background row, 56px height, floats over the top of the hero photo rather than sitting in its own White bar — back-chevron icon only, no title text (the shop name lives in [C] below, not duplicated here).
**Icon treatment:** the back-chevron sits inside a small circular White-background chip (36×36, subtle shadow) so it stays legibly tappable regardless of what's underneath it in the photo — a standard treatment for controls floating over photography, since a plain icon directly on an image can disappear against a busy background.
**Animation:** none — as the customer scrolls past the photo, this header can optionally solidify into a normal White bar with the shop name appearing (a common photo-detail-page pattern), cross-fading in over `duration-quick` (180ms) once scroll position passes the photo's bottom edge; this is a nice-to-have polish detail, not a hard requirement — a static floating header is an entirely acceptable v1 fallback if the transition adds complexity the team doesn't want yet.

### [B] Shop photo (hero)
**Content:** the shop's `photoUrl` (Cloudinary-hosted, uploaded at onboarding, §3/§4.2), full-width, 16:9 aspect ratio, no crop-to-square — a shop's storefront photo deserves its natural framing, not a forced square thumbnail like the Shortlist card's compact treatment.
**Fallback (no photo on file — a shop that submitted a proof-of-business document instead of a photo, §3):** a full-width solid-color panel (Gray 100 background) with the shop name's first letter centered in large Text-secondary type — the same initials-avatar logic as the Shortlist card, scaled up, rather than a generic "no image" placeholder graphic (keeps the fallback informative rather than apologetic).
**Color:** photo as supplied; fallback panel per above.
**Animation:** simple fade-in (`duration-standard`, 240ms) as the image loads — no parallax, no ambitious scroll-driven effects (matches the calm-tone rule; a restaurant-website hero video/parallax pattern from the research reviewed is explicitly *not* appropriate here — NEARBY has no cost budget for that kind of asset weight, and it would misrepresent a small hand-onboarded shop as something more produced than it is).

### [C] Name, category, status row
**Layout:** sits directly below the photo, White background, generous padding (16px).
**Content:**
- Shop name — H1 size (24px), 600 SemiBold, Text-primary, first line.
- Second line: category tag (rounded-full pill, Gray 100 background, Caption text) + open/closed status inline ("Open now" Success teal with solid-dot prefix, or "Closed" Text-secondary) — identical treatment to the Shortlist card's second line, for visual continuity between the two screens.
**Animation:** none.

### [D] Rating & reliability row
**Layout:** single row below [C], two halves.
**Left half:** star-rating icon row (gold/amber tone, same brand-guide exception noted in the Shortlist spec) + numeric average, e.g. "★ 4.6" — **or**, if `avgRating` is null, "New to NEARBY" text pill (never fabricated empty stars, same rule as the Shortlist card).
**Right half:** reliability badge — "Usually reliable" (Success-teal-tinted pill, score ≥ 80) / "Sometimes reliable" (Gray-100 pill, score 50–79) / no badge (score exactly 50 with zero rated visits) — identical logic and copy to the Shortlist card's badge, deliberately not re-explained differently here (a customer who saw the badge on the card shouldn't see conflicting language on the detail page).
**Optional expansion (nice-to-have, not required for v1):** tapping this row could expand a one-line rating-breakdown note, e.g. "Based on **[foundCount + notFoundCount]** rated visits" — gives the number underlying the badge without exposing the raw formula weights, useful context for a customer who wants slightly more than the badge alone but still shouldn't see the internal 0.7/0.3 weighting.
**Animation:** none beyond the optional expansion's height-transition (`duration-quick`, `ease-out-soft`) if implemented.

### [E] Address & distance card
**Purpose:** directly answers "can I get there" — the research reviewed is unanimous that address, and ideally a map, belongs above the fold or very near it; NEARBY's version skips embedding a full interactive map on this screen (that's already the dedicated Location-Confirm screen's job elsewhere in this product, and duplicating a Leaflet instance here for a single static point is unnecessary weight) in favor of a lightweight static treatment.
**Layout:** card, Surface white background, 1px Border-gray outline, rounded 12px, pin icon (16×16, Primary orange) + the shop's `formattedAddress` (from its geocoded location, §3/§4.2) on one line, distance from the customer's request location on the line below ("0.4 km away," Caption, Text-secondary — the exact same distance value already shown on the Shortlist card, carried through consistently).
**"Get directions" action:** small inline text-button (Caption, Primary orange), right-aligned within the card — opens the device's native maps app via a standard `geo:`/Google-Maps-URL deep link pre-filled with the shop's coordinates, rather than rendering an in-app map; this is the pragmatic, zero-additional-cost choice (no second Leaflet instance, no extra tile-server load) and matches how a customer would actually want to navigate there once they've decided — out to their phone's real navigation app, not a small in-app preview.
**Animation:** none — static reference card.

### [F] Opening hours card
**Purpose:** directly surfaces the shop's `openingHours` schedule (§3.4) — when a customer can expect to visit, beyond just the current open/closed snapshot already shown in [C].
**Layout:** card, same styling as [E], clock-outline icon + a compact 7-row day/hours list (Mon–Sun, each row: day abbreviation + hours or "Closed" for that day), current day's row highlighted with a subtle Gray-100 background band to make "today" easy to find at a glance.
**Fallback (shop has no schedule set, §3.4 — "a shop with no schedule set is eligible whenever `isOpen` is true"):** replace the 7-row list with a single line: "Hours vary — currently **[Open now / Closed]**" — never render an empty/misleading schedule grid for a shop that simply hasn't set one.
**Timezone note:** hours are already evaluated and stored relative to the shop's own IANA timezone (§3.4) — display them as the shop's local hours (which, for a single-locality pilot, will typically match the customer's own local time anyway, but the display logic should read the shop's stored timezone rather than assuming, so this remains correct if/when a second locality is ever onboarded, §3.11).
**Animation:** none.

### [G] Contact row
**Purpose:** the direct "call the shop" action — Zomato's own UX analysis specifically calls out a prominent, color-highlighted "call" CTA as the product page's key action alongside the main order button; NEARBY's version keeps this as a clearly available but secondary action, since **Select this shop** (Section [H]) is this screen's actual primary conversion action, not calling.
**Layout:** single row, phone-icon (16×16, Primary orange) + the shop's contact phone number displayed as tappable text (`tel:` link), Body size, Text-primary — sits below the hours card, above the fixed bottom button.
**Copy:** the phone number itself, e.g. "+91 XXXXX XXXXX" — no extra label needed, the icon communicates "this is how to call them."
**Animation:** none beyond standard tap-scale feedback on the tappable link.

### [H] Select this shop button
**Layout:** fixed to the bottom of the screen, full-width minus 16px margins, 48px height, 12px radius, sits above the safe-area inset — visually and behaviorally identical to the Shortlist card's per-card button, just at screen-CTA scale.
**Copy:** "Select this shop"
**Behavior:** identical to the Shortlist screen's Section [E] — opens the same select-confirmation modal ("Select **[Shop Name]**? The other shops that said yes will be notified they weren't picked — this won't affect their rating."), same Cancel/Confirm actions, same `POST /api/v1/requests/:id/select` call on confirm, same cross-fade transition to the Selection Confirmation screen. **This is a deliberate reuse, not a re-specification** — building two different confirmation flows for the same action reachable from two different screens would only create an opportunity for the copy or behavior to drift apart over time.
**States/animation:** identical to the Shortlist card's button spec (Primary orange default, Primary-pressed on press, tap-scale-to-0.97).

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | "Select this shop" button, pin/phone/clock icons, "Get directions" link |
| Success | `#00B8A9` | "Open now" status, "Usually reliable" badge |
| Text primary | `#14213D` | Shop name, address text, phone number |
| Text secondary | `#5B6472` | Category tag, "Closed" status, distance text, hours-list text, "Sometimes reliable" badge |
| Border | `#D8DCE3` | Address/hours card outlines |
| Surface | `#F4F5F7` / `#FFFFFF` | Fallback photo panel, category pill, "today" row highlight (Gray 100) / header chip, cards, page background (White) |

**Same gold/amber star-icon exception** carried over from the Shortlist spec (Section 4 there) — applies identically here.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Shop name | H1 / 24px | 600 SemiBold |
| Category tag / status | Caption / 13px | 400 Regular |
| Rating numeral / reliability badge | Caption / 13px | 400 Regular (rating numeral may bump to 600 SemiBold to match the Shortlist card's treatment) |
| Address text | Body / 15px | 400 Regular |
| Distance / "Get directions" | Caption / 13px | 400 Regular |
| Hours-card day/time rows | Body / 15px | 400 Regular ("today" row bumped to 600 SemiBold) |
| Phone number | Body / 15px | 400 Regular |
| "Select this shop" button | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Tap-scale feedback (button, phone link, directions link) |
| `duration-quick` | 180ms | Optional floating-header solidify-on-scroll, optional rating-breakdown expansion |
| `duration-standard` | 240ms | Hero photo fade-in on load, select-confirmation modal handoff (shared with Shortlist spec) |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Photo fade-in, screen-exit on confirmed selection |

**No broadcast pulse, no looping animation anywhere on this screen** — same reasoning as the Shortlist screen: this represents settled, browsable information about a specific shop, not a live/real-time state.
**No parallax, no scroll-driven hero effects** — explicitly ruled out in Section 3[B] above, despite being common on restaurant marketing websites; NEARBY's brand register and zero-cost-asset philosophy both point away from that treatment here.

---

## 7. Images, icons & video — asset list

**No video.** This is the second (and last, in this screen series so far) screen where real shop photography appears — deliberately, and only the shop's own supplied image.

| Asset | Format/size | Source |
|---|---|---|
| Shop hero photo | Full-width, 16:9, shop-supplied at onboarding (Cloudinary) | User-generated |
| Fallback initials panel | CSS-drawn, no image asset | — |
| Back-chevron icon (in floating chip) | Line icon, 24×24 | Icon library |
| Star-rating icon | 14×14, gold/amber tone | Icon library (same as Shortlist spec) |
| Pin icon (address card) | Line icon, 16×16, Primary orange | Icon library |
| Clock-outline icon (hours card) | Line icon, 16×16, Primary orange | Icon library |
| Phone icon (contact row) | Line icon, 16×16, Primary orange | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B]–[G] | All fields delivered as part of the shortlist payload the customer already has client-side from `request:ready` (name, photoUrl, category, distance, avgRating, reliabilityScore-derived badge, address, openingHours, phone, timezone) — this screen should **not** need a fresh network call if reached directly from the Shortlist screen's already-loaded data; if reached via a deep link or a stale client cache, fall back to a fresh `GET /api/v1/shops/:id` (or equivalent) fetch |
| [E] "Get directions" | Client-side deep link construction only (`geo:` URI scheme or a Google Maps web-intent URL using the shop's stored lat/lng) — no server call |
| [H] Select | `POST /api/v1/requests/:id/select` with `{ shopId }` — identical to the Shortlist screen's Section [E] |

**Loading state:** if a fresh fetch is needed (deep-link case above), show a simple skeleton: gray photo-placeholder block + 3 gray text-line placeholders, shimmer, swapped for real content over `duration-quick`.
**Error state:** a failed fetch uses the shared toast/error-boundary pattern with retry (§6.1); a failed selection follows the identical inline-modal-error handling already specced for the Shortlist screen.

---

## 9. Responsive & platform notes

- **Viewport:** single column, max-width 480px centered, consistent with every screen in this series; the hero photo scales to fill the available width at any breakpoint while keeping its 16:9 ratio (no cropping adjustments needed across widths).
- **Photo loading performance:** since this is the one screen carrying meaningful image weight in the whole customer app, use a reasonably compressed/responsive Cloudinary transformation (Cloudinary supports on-the-fly resizing via URL parameters) rather than serving the shop's original uploaded resolution — keeps this screen's load time in line with the zero-latency-loading principle established everywhere else in the product, without needing any custom image-processing code beyond a Cloudinary URL parameter.
- **PWA standalone mode:** reached via the Shortlist screen's normal (non-modal) navigation, so standard back-gesture/back-button returns to the Shortlist screen with its scroll position preserved — this is a drill-through detail view within an already-open "place" in the app, not a new modal task.
- **Deep-linkability:** since a shop's profile could plausibly be reached from a push notification or a shared link in a future iteration, this screen's data requirements (Section 8) are deliberately written to support a cold, fresh-fetch entry path, not just the common warm-navigation case from the Shortlist screen.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1.1, 3.1.2, 3.4, 4.2–4.5) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). The "Select this shop" action and its confirmation modal are deliberately specified as a direct reuse of the Shortlist screen's own Section [E], not a re-specified duplicate — the same copy, the same endpoint, the same transition, regardless of which of the two screens a customer selects from.*
