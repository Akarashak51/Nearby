# NEARBY — Incoming Request Card / Detail Build Specification
### Everything needed to build the `request:new` card component and its expanded detail view — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1, 4.3, 4.4, 5.3, 6.3), NEARBY Brand/PWA Guide v1.0 (§1.4, 3.3), and the Zomato Restaurant Partner accept/reject pattern already established as this component's inspiration in this series' page-architecture document

---

## 0. Scope — two views of one object, and one privacy decision worth stating explicitly

This document specs two connected things: the **compact card** already introduced as the Live Dashboard's core content (previous spec, Section [E]), and its **detail view** — what opens when a shopkeeper taps the card body itself rather than one of its two buttons directly. The compact card exists for fast, confident decisions; the detail view exists for the moments a shopkeeper wants a beat longer, more room, or just a bigger, more comfortable pair of buttons before committing.

**One deliberate design decision, stated plainly since nothing in the source document forces it either way:** neither view shows the customer's exact location on a map. NEARBY's model has the *customer* traveling to the *shop*, never the reverse — so there is no operational reason for a shop to ever see precisely where a customer was standing when they broadcast. Showing that pin anyway would reveal something about a stranger's real-time location for no functional benefit. **Distance is the only spatial information a shop needs, and it's the only spatial information either view of this component shows.** This mirrors a privacy principle already established elsewhere in this series (the customer-side Live-Count screen withholding shop identity during the open window) applied in the opposite direction.

**A related, out-of-scope note:** the original page-architecture document flagged a possible "rush-hour" toggle (temporarily pausing new incoming requests during a busy patch) as an idea inspired by the Zomato Partner app — it is **not** a documented feature in your Master Build Document, and this specification does not build it. It's mentioned briefly at the end of this document as a flagged future idea, not specced in detail, so as not to imply it's already part of the confirmed scope.

---

## 1. Page goals, in priority order

1. Make the compact card's Yes/No decision fast, confident, and hard to misclick — most responses should never need the detail view at all.
2. Give the detail view real, distinct value for the moments it's used — bigger context, bigger buttons, not just a zoomed-in copy of the same card.
3. Never expose more about the customer than operationally necessary — distance, not location; a product query, not an identity.
4. Keep both views perfectly in sync with the same underlying countdown and the same real-time removal behavior already established at the dashboard level.

---

## 2. Layout overview — compact card (as it appears in the dashboard feed)

```
┌─────────────────────────────┐
│ Product query text             │
│ Distance · Countdown            │
│ ┌───────────┐ ┌───────────┐   │
│ │    Yes     │ │     No     │   │
│ └───────────┘ └───────────┘   │
└─────────────────────────────┘
```

## 2b. Layout overview — detail view (tap-through from the card body)

```
┌─────────────────────────────┐
│ [A] Header (X + title)         │
├─────────────────────────────┤
│ [B] Large countdown centerpiece │
├─────────────────────────────┤
│ [C] Request details card        │
├─────────────────────────────┤
│ [D] Yes / No buttons (large)    │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### Compact card (full spec, consolidating and finalizing what the Dashboard document introduced)
**Layout:** full-width card, Surface white background, 1px Border-gray outline, rounded 12px, comfortable internal padding (16px), sits in the dashboard's incoming-requests feed.
**Content:**
- **Product query** (`productQuery`, Body, 600 SemiBold, Text-primary) — the single most important line, always first, never truncated to fewer than 2 lines before ellipsis (a shopkeeper needs to actually read what's being asked for, not guess from a cut-off fragment).
- **Distance + countdown, one row** — distance (`distanceMeters`, Caption, Text-secondary, e.g. "0.6 km away") on the left, countdown (`mm:ss`, Body, 600 SemiBold, Text-primary — Warning amber in the final 10 seconds, identical rule to the customer-side Live-Count screen's timer) right-aligned.
- **Yes / No buttons, full-width split, 52px height:** "Yes" filled **Primary orange**, White text; "No" outline style, White background, 1px Border-gray outline, Text-primary text — this exact pairing is, per the Dashboard spec's Section 0, the **only** place orange appears anywhere in the shop dashboard.
**Tap targets:** the Yes/No buttons answer immediately on tap (Dashboard spec, Section [E]) — tapping anywhere else on the card body (the product text, the distance/countdown row) opens the **detail view** below, for a shopkeeper who wants a moment longer or simply prefers a larger, more deliberate confirmation step. Both paths are equally valid; neither is presented as the "correct" way to respond.
**Animation:** entrance/exit exactly as specced in the Dashboard document (fade+translateY on arrival, fade+collapse on removal) — not repeated here in full to avoid duplicating an already-final spec.

### [A] Detail view header
**Layout:** 56px row, an **X (close)** icon on the left (returns to the dashboard feed without answering — this view is optional context, not a forced step), centered title.
**Copy:** "Incoming request"
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Large countdown centerpiece
**Purpose:** the detail view's clearest value-add over the compact card — a genuinely larger, easier-to-read version of the same countdown, for a shopkeeper who wants more visual room around the number they're watching.
**Layout:** centered composition, upper third of the screen, the countdown numeral rendered at Display size (32px, 700 Bold) rather than the compact card's Body-size treatment — a direct scale-up, not a redesign.
**Label above:** "Time to respond" (Caption, Text-secondary).
**Color:** Text-primary default, **Warning amber** in the final 10 seconds — identical rule and threshold to every other countdown in this product (the customer-side Live-Count screen, the compact card above).
**Animation:** the numeral ticks per-second with no easing (same "should feel exactly accurate" principle as every other countdown in this series); the color shift at 00:10 cross-fades over `duration-instant` (100ms). If the window closes while this view is open (countdown reaches its end without a response), the view auto-dismisses back to the dashboard, where the card has already been removed by the same `request:expired`/`request:ready` handling specced at the dashboard level — this view never shows its own separate "expired" state, it simply closes, since the dashboard is the single source of truth for which requests are still answerable.

### [C] Request details card
**Purpose:** the actual added context — everything worth knowing before answering, laid out with more breathing room than the compact card's single dense row allows.
**Layout:** card, Surface white background, 1px Border-gray outline, rounded 12px, simple label/value rows:
- **What they need:** `productQuery`, shown in full (Body, Text-primary) — repeated here even though it's already visible above the fold on the compact card, since the detail view should be readable as a complete, self-contained summary on its own.
- **Distance:** `distanceMeters` (Body, Text-secondary), e.g. "0.6 km away" — exactly the same figure as the compact card, never recalculated or shown differently between the two views.
- **Requested:** a relative timestamp of when the broadcast was made (`requests.createdAt`, e.g. "Just now" / "1 minute ago," Body, Text-secondary) — a small piece of context the compact card omits for space, useful for a shopkeeper checking their dashboard slightly after a request first arrived.
**No customer name, no photo, no exact location, no map** — per Section 0's stated privacy decision, this card is deliberately as complete as it should ever get; there is nothing more to show, not because of a technical limitation, but because nothing more is appropriate to show.
**Animation:** none — static reference content, same treatment as every recap-style card in this series.

### [D] Yes / No buttons (large)
**Layout:** fixed to the bottom of the screen, two equal-width buttons side by side, **60px height** (larger than the compact card's already-large 52px, since the entire premise of opening this view is wanting a more comfortable, deliberate tap target), 12px radius each, sits above the safe-area inset.
**Copy:** "Yes" / "No" — identical wording to the compact card, no elaboration needed at this size.
**Color:** identical to the compact card — "Yes" filled **Primary orange**, White text; "No" outline, Border-gray, Text-primary text.
**Behavior:** identical `POST /api/v1/requests/:id/respond` call to the compact card's own buttons — this view is not a different decision path, just a roomier one; tapping either button here answers exactly as if the equivalent compact-card button had been tapped, and the screen closes back to the dashboard the instant the response is recorded.
**Animation:** tap-scale-to-0.97 on either button; screen-exit cross-fade (`duration-standard`, 240ms, `ease-out-soft`) back to the dashboard feed, where the card has already been removed.

---

## 4. Color usage summary (Brand Guide §3.1/3.3 — no new hex introduced)

| Token | Hex | Used where |
|---|---|---|
| **Primary orange** | `#FF5A36` | "Yes" button, both compact card and detail view — the only element carrying this color in either view |
| Warning | `#FFB627` | Countdown numeral in the final 10 seconds, both views |
| Text primary | `#14213D` | Product-query text, "No" button text, detail-card labels/values |
| Text secondary | `#5B6472` | Distance text, relative-timestamp text, countdown label |
| Border | `#D8DCE3` | Card outlines, "No" button outline |
| Surface | `#F4F5F7` / `#FFFFFF` | Page background (Gray 100) / cards, header, button backgrounds (White) |

**Zero Signal Teal anywhere on either the compact card or the detail view** — worth stating explicitly, since it's the shop surface's dominant accent everywhere else: this specific component is the one place in the whole shop dashboard where the *only* meaningful accent color is orange, precisely because this is the one action Brand Guide §3.3 reserves orange for. Introducing Teal here, even decoratively, would dilute that reservation.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Compact card: product-query text | Body / 15px | 600 SemiBold |
| Compact card: distance | Caption / 13px | 400 Regular |
| Compact card: countdown | Body / 15px | 600 SemiBold |
| Compact card: Yes/No labels | Button / 15px | 600 SemiBold |
| Detail header title | H1 / 24px | 600 SemiBold |
| Detail: countdown numeral | Display / 32px | 700 Bold |
| Detail: countdown label | Caption / 13px | 400 Regular |
| Detail: request-card labels/values | Body / 15px | 400 Regular |
| Detail: Yes/No labels | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Countdown color-shift at 00:10 (both views), tap-scale on all buttons |
| `duration-standard` | 240ms | Compact-card entrance/removal (per the Dashboard spec), detail-view screen-exit on response |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Detail-view screen-exit |

**No looping animation on either view** — same reasoning as the Dashboard screen itself: the countdown numeral's own ticking is the "this is time-sensitive" signal, and it needs no additional ambient motion layered on top.
**Reduced motion:** covered by the app-wide `prefers-reduced-motion` query.

---

## 7. Images, icons & video — asset list

**No video, no photography, no map, no customer imagery of any kind** — per Section 0's privacy decision, this component shows text and a countdown only.

| Asset | Format/size | Source |
|---|---|---|
| Close (X) icon (detail header) | Line icon, 24×24 | Icon library |

*(No other new assets — both views are built entirely from typography, color, and the standard button/card components already established throughout this series.)*

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| Compact card | Populated directly from the `request:new` Socket.IO payload (or a `GET /shops/me/requests` entry, on the 5s-poll fallback) — `productQuery`, `distanceMeters` (or the raw request/shop locations if the client computes distance itself), `createdAt` |
| Detail view | Reuses the exact same in-memory data as the compact card it was opened from — no separate fetch, since nothing shown in the detail view is information the compact card's underlying object doesn't already contain |
| Both views' Yes/No | `POST /api/v1/requests/:id/respond` with `{ responseValue: 'yes' | 'no' }` (rate-limited 60/shop/hour, §5.3) |
| Card/detail removal | `request:expired`, `request:ready`, or `request:cancelled` — if the detail view is open when one of these arrives for its request, it auto-dismisses (Section 3[B]) rather than showing its own separate terminal state |

**Loading state:** neither view has one — both render immediately from already-available data (either a live socket payload or the dashboard's already-fetched feed), with no additional network dependency of their own.
**Error state:** a failed `POST .../respond` call from either view uses the shared toast/error-boundary pattern (§6.1); critically, the card must **not** be removed from the feed (or the detail view auto-dismissed) until a response is actually confirmed successful — identical reasoning to the Dashboard spec's own note on this exact failure mode.

---

## 9. Responsive & platform notes

- **Viewport:** the compact card sits within the Dashboard's existing feed layout (max-width 480px, per that spec); the detail view is a full-screen modal-style presentation, same treatment as the customer-side New Broadcast screen — a focused task, not a standing "place."
- **Multiple open detail views:** not applicable — only one incoming request's detail view can reasonably be open at a time, since it's launched from a specific card tap and closes back to the feed before another can be opened.
- **PWA standalone mode:** the detail view carries no bottom tab bar (a focused, modal-style presentation, consistent with every other task-screen treatment in this series) — its close (X) or a successful Yes/No response both return directly to the Dashboard, tab bar and all.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 4.3, 4.4, 5.3, 6.3) and NEARBY Brand/PWA Guide v1.0 (Sections 1.4, 3.3). The decision to withhold the customer's exact location from both views — showing distance only — is a deliberate privacy choice consistent with this product's broadcast-and-visit model (the customer travels to the shop, never the reverse), not a limitation of the available data; the request document already contains a full location, this component simply never surfaces it. The "rush-hour" toggle flagged in this series' original page-architecture document remains an unbuilt, inspired-by idea, not a documented requirement, and is intentionally excluded from this specification.*
