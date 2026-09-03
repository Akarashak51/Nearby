# NEARBY — Request-in-Progress (Live Count) Screen Build Specification
### Everything needed to build the `open`-state view of the request-detail screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1, 3.9A, 5.3, 6.3), NEARBY Brand/PWA Guide v1.0 (§1.4, 9.2), and live-order-tracking / countdown-timer UX research (Just Eat Takeaway's order-tracking redesign, Zomato Live Activities, CSS countdown-timer zero-state handling)

---

## 0. Scope — what this screen is, and the one rule everything else follows from

This is the screen shown the instant a broadcast is created, for as long as `requests.status === 'open'` (Master Build Doc §5, §6.3). It is the direct product of the New Broadcast screen's submit action, and it hands off to exactly one of three next screens depending on which Socket.IO event arrives: **Shortlist** (`request:ready`), **Expired** (`request:expired`), or **Cancelled** (`request:cancelled`, if the customer cancels it themselves).

**The one rule that shapes every design decision below:** while the window is open, the customer's client receives `response:update` events with **a live count only** — individual shop names, distances, and ratings are explicitly withheld until the window closes (§3.1, "Real-time events" table). This is a deliberate anti-gaming design in your own spec, not an oversight, and it means **this screen must never show a shop list, shop card, or shop name of any kind** — no matter how tempting it is to reuse the Shortlist screen's card component early. The entire visual vocabulary of this screen is built around *a number changing*, not *a list populating*.

This is functionally NEARBY's version of a delivery app's live-tracking screen — but simpler by necessity: there's no map, no moving pin, no ETA-ticking-down-as-a-courier-approaches, because there's no courier. What research on delivery-app tracking screens (Just Eat's redesign writeup, Zomato's Live Activities work) confirms transfers directly, though, is the *state-machine-driven-transitions* pattern and the *countdown-that-never-looks-broken* pattern — both specced in detail below.

---

## 1. Page goals, in priority order

1. Make it unambiguous, at a glance, that the broadcast is live and working — the single biggest anxiety point on any "waiting" screen (per the delivery-tracking research reviewed) is a customer wondering "is this actually doing anything?"
2. Show the live Yes-count updating in real time, without ever leaking which shops those Yes's belong to.
3. Show a countdown to `expiresAt` that degrades gracefully at zero — never freezes on "0:00" or drifts into a broken-looking negative value (directly informed by the CSS-countdown-timer research: "a courier who is two minutes late should not make the interface look broken").
4. Let the customer cancel cleanly, and let them leave/return without losing state — this is a passive-wait screen, not a modal the customer is trapped on.
5. Transition to the correct next screen the moment the server says to, with **zero customer action required** — this screen's entire job ends the instant `request:ready`/`request:expired`/`request:cancelled` arrives.

---

## 2. Layout overview

Single screen, no bottom tab bar (this is a focused, modal-style state — same presentation pattern as the New Broadcast screen it was launched from), centered vertical composition since there's genuinely one thing to look at.

```
┌─────────────────────────────┐
│ [A] Header (title only)      │
├─────────────────────────────┤
│ [B] Reconnect/offline banner │  ← conditional, top of content
├─────────────────────────────┤
│ [C] Broadcast pulse + count   │  ← the visual centerpiece
├─────────────────────────────┤
│ [D] Countdown timer           │
├─────────────────────────────┤
│ [E] Request recap card         │
├─────────────────────────────┤
│ [F] Reassurance copy           │
├─────────────────────────────┤
│ [G] Cancel button              │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, no back-chevron (per the "not trapped, but not accidentally dismissible" balance — see [G] for the actual exit path), centered title.
**Copy:** "Broadcasting your search"
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Reconnect/offline banner *(shown only when the Socket.IO connection has dropped and the client has fallen back to REST polling)*
**Purpose:** per §6.3, the client polls `GET /api/v1/requests/:id` every 5s if the socket connection drops (covers Render's free-tier cold-start reconnect lag) — the customer should know they're on the fallback path, not be left wondering why the count feels laggier.
**Copy:** "Reconnecting… updates may be a few seconds delayed"
**Layout:** thin full-width bar directly below the header, Gray 100 background, small spinner icon (12×12) + Caption text, Text-secondary color.
**Behavior:** disappears the instant the socket reconnects (no need for the customer to do anything); does not interrupt or reset the countdown/count shown in [C]/[D] — those keep their last-known values and simply resume live updates once reconnected, or receive one authoritative refresh from the next poll.
**Animation:** slides down from height 0 on appearance, slides back up on disconnect-resolved (`duration-quick`, 180ms, `ease-standard`) — a persistent-until-resolved banner, not a toast that auto-dismisses on a timer (its dismissal is tied to the actual connection state, not a fixed duration).

### [C] Broadcast pulse + live count — the centerpiece
**Purpose:** this is the one element on the whole page that has to communicate "this is alive and working right now" — everything else on the screen is supporting text.
**Layout:** large centered composition, roughly 200×200px visual footprint, positioned in the upper-middle of the screen where a hero image would sit on a marketing page — except here it's built entirely from the brand's existing pulse motif, not photography.
**Visual construction:** the NEARBY logomark's three radiating arcs (Brand Guide §9.2's broadcast-pulse pattern), scaled up from the small icon-sized version used on the Home page's Active Request banner to a full centerpiece size here — same animation, different scale, so the visual language stays consistent across the product rather than inventing a second "live" motif.
**Live count, inside the pulse:** a large number (Display size, 32px/700 Bold, Text-primary) centered inside the arcs, with the label directly beneath it in Caption size: "shops have responded" (singular "shop has responded" at count = 1; "No responses yet" at count = 0, not "0 shops have responded" — small copy detail that avoids the slightly cold read of a bare zero).
**Update behavior:** on each `response:update` event, the number eases from its old value to the new one over `duration-quick` (180ms) — a genuine count-up/count-down tween, not an instant swap, matching the same live-count-easing rule already established for the Home page's Active Request banner (reused here, not reinvented, since both are literally the same underlying data).
**Color:** pulse arcs in Success teal (#00B8A9) — not Primary orange — reusing the exact color choice already established for the Home page's Active Request banner pulse (an `open`-status request is Success-tinted there too); the number itself in Text-primary, the label in Text-secondary.
**Animation:** the pulse loops continuously (~1.6s cycle, opacity fade, per Brand Guide §9.2) for as long as this screen is showing an `open` request — this is the *one* place in the entire product where a long-running looping animation is correct, precisely because it's signaling a genuinely ongoing real-time process, not decoration. Respects `prefers-reduced-motion` (Brand Guide §9.5): reduced-motion users see a static version of the arcs with no loop, and rely on the count-up number and countdown timer for "is this working" reassurance instead.

### [D] Countdown timer
**Purpose:** shows time remaining until `expiresAt` — the response window's hard deadline.
**Layout:** centered below [C], Body-size (15px) but bumped to 600 SemiBold weight (a numeric read-out the customer is actively watching, same weight-bump logic used for the radius-slider's live value in the New Broadcast spec), format `mm:ss`, counting down.
**Copy (label above the number):** "Time remaining"
**Zero-state handling — the detail most implementations get wrong** (directly informed by the countdown-timer research reviewed: an ETA "should not make the interface look broken" when it hits its edge case): this timer must **never show a negative value or freeze visibly at 00:00 while nothing happens.** The correct sequence is:
1. Counts down normally from the window's full duration (typically 120s, per §3.2 — read the actual `expiresAt` minus now, never hard-code 120).
2. At 00:10 remaining, the numeral's color shifts from Text-primary to Warning amber (#FFB627) — a gentle, non-alarming "wrapping up soon" cue, not a jarring red flash (this is a calm-tone product, per Brand Guide §1.4 — no fake urgency).
3. At 00:00, the label text swaps to **"Wrapping up…"** and the numeral is replaced by a small inline spinner — this is a brief, expected gap (server-side window-close processing, shortlist building, or the zero-Yes expiry check) rather than a stuck timer. This state should resolve within a few seconds as the corresponding `request:ready`/`request:expired` event arrives and this screen transitions away entirely — it is never a long-term state a customer sits on.
**Color:** default Text-primary, Warning amber in the final 10 seconds (as above).
**Animation:** the numeral itself ticks per-second with no easing (a countdown should feel exactly accurate, not smoothed); the color shift at 00:10 cross-fades over `duration-instant` (100ms).

### [E] Request recap card
**Purpose:** a quiet reminder of exactly what was broadcast, since the customer may return to this screen after switching apps for a minute and want to reconfirm what they asked for.
**Layout:** simple card, Surface white background, 1px Border-gray outline, rounded 12px, sits below the timer.
**Content:** three rows, each a small icon + label:
- 🔍 Product query text (e.g. "AA batteries")
- 📍 Radius ("Within 1 km")
- 📌 Location (`formattedAddress`, truncated)
**Color:** icons Text-secondary, labels Text-primary, all Body/Caption size — this card is reference material, not something competing for attention with [C]/[D].
**Animation:** none — static, informational.

### [F] Reassurance copy
**Purpose:** manages the "is anything actually happening" anxiety directly in words, as a backstop to the pulse animation.
**Copy:** "We've notified open shops within your search radius. You'll be able to pick one as soon as they respond."
**Layout:** centered, Caption size, Text-secondary color, single paragraph, sits between the recap card and the cancel button.
**Animation:** none.

### [G] Cancel button
**Purpose:** the customer's one available action on this screen — per §5, cancellation is available any time while `status` is `open` (`POST /api/v1/requests/:id/cancel`), and is *not* available once the window closes into `awaiting_selection`, so this button only ever needs to exist in this specific screen/state.
**Layout:** fixed to the bottom of the screen, full-width minus 16px margins, 48px height, 12px radius — but styled as a **secondary/outline button**, not a filled Primary button: White background, 1px Danger-red outline, Danger-red text. This is a deliberate visual demotion relative to every other primary CTA in the product (Home's broadcast entry, New Broadcast's submit button, Sign-up's submit button) — cancelling is the one action on this screen that should visually read as "available, but not what we're hoping you'll do," without resorting to hiding it or making it hard to find.
**Copy:** "Cancel search"
**Behavior:** tapping opens a lightweight inline confirmation (not a full second screen) — "Cancel this search? Shops that were notified will be told it's no longer needed." with **Keep searching** / **Yes, cancel** actions — a one-tap-away safety check, since cancellation can't be undone and shops have already been notified.
**On confirm:** `POST /api/v1/requests/:id/cancel` → `status` becomes `cancelled`, `request:cancelled` fires to every shop with a pending response, and the customer's screen transitions to the Cancelled screen (page-architecture item 1.10), which distinguishes this customer-initiated case from an auto-cancel-on-suspension case in its own copy.
**Color:** Danger red (#E5484D) outline/text, as above; confirmation dialog's "Yes, cancel" action in filled Danger red, "Keep searching" as the visually primary/default-focused action (Primary orange, filled) — the safer, non-destructive choice gets the more prominent button styling, a standard confirmation-dialog convention.
**Animation:** confirmation dialog fades+scales in (`duration-quick`, 180ms) as a standard modal-overlay pattern; on confirm, this screen cross-fades to the Cancelled screen (`duration-standard`, 240ms, `ease-out-soft`) — same handoff pattern used everywhere else in the product.

---

## 4. Screen-exit transitions (brief — full specs belong to their own documents)

| Event received | Destination | Transition |
|---|---|---|
| `request:ready` (≥1 Yes, window closed) | Shortlist / Awaiting Selection screen | Cross-fade, `duration-standard`, `ease-out-soft` |
| `request:expired`, `yesCount: 0` | Expired screen, "no shops responded" copy variant | Same cross-fade pattern |
| `request:expired`, `yesCount ≥ 1` (selection-timeout edge case, §3.1.3) | Expired screen, "shops responded but window closed" copy variant, no radius suggestion | Same cross-fade pattern |
| `request:cancelled` (customer-confirmed via [G]) | Cancelled screen, customer-initiated copy | Same cross-fade pattern |
| `request:cancelled` (auto, e.g. shop-suspension side effect, §3.8) | Cancelled screen, auto-cancel copy | Same cross-fade pattern |

All five transitions use the identical cross-fade — consistent with every other screen handoff specced across this product's build documents so far, reinforcing that "moving to the next state" always feels the same regardless of which state it is.

---

## 5. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Success | `#00B8A9` | Broadcast pulse arcs, count-up number's implicit "alive" framing (color carried by the pulse, not the number text itself) |
| Warning | `#FFB627` | Countdown numeral in the final 10 seconds |
| Danger | `#E5484D` | Cancel button outline/text, "Yes, cancel" confirmation action |
| Text primary | `#14213D` | Header title, live count number, countdown default color, recap-card labels |
| Text secondary | `#5B6472` | Live-count sub-label, reassurance copy, recap-card icons, reconnect-banner text |
| Border | `#D8DCE3` | Recap card outline |
| Surface | `#F4F5F7` / `#FFFFFF` | Reconnect banner background (Gray 100) / header, recap card, cancel-confirmation dialog backgrounds (White) |

---

## 6. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Live count number | Display / 32px | 700 Bold |
| Live count sub-label | Caption / 13px | 400 Regular |
| Countdown numeral | Body / 15px | 600 SemiBold |
| Countdown label ("Time remaining" / "Wrapping up…") | Caption / 13px | 400 Regular |
| Recap-card labels | Body / 15px | 400 Regular |
| Reassurance copy | Caption / 13px | 400 Regular |
| Cancel button label | Button / 15px | 600 SemiBold |
| Reconnect-banner text | Caption / 13px | 400 Regular |

---

## 7. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Countdown color shift at 00:10, tap-scale on cancel button |
| `duration-quick` | 180ms | Live-count number tween, reconnect-banner slide, cancel-confirmation dialog fade-in |
| `duration-standard` | 240ms | Screen-exit cross-fades (all five, per Section 4) |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Reconnect-banner slide |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Screen-exit transitions |
| Broadcast pulse | ~1.6s loop, opacity fade | [C], continuously, for as long as `status === 'open'` |

**This is the one screen in the whole product where a continuous looping animation is correct** — every other spec in this series has been careful to reserve the pulse for genuinely live states and keep everything else static; this screen *is* that live state, full stop.
**Reduced motion:** the pulse becomes a static rendering of the same arcs (Section 3[C] above); the countdown and live-count-number tweens are numeric-value changes, not decorative motion, so they're left running even under `prefers-reduced-motion` — a number needs to change to convey information here, unlike a purely cosmetic animation.

---

## 8. Images, icons & video — asset list

**No video, no photography.**

| Asset | Format/size | Source |
|---|---|---|
| Broadcast pulse arcs | Scaled-up rendering of `nearby-mark.svg`'s arc paths (same source asset as the Home page's small pulse icon, just larger) | Reused, no new asset |
| Reconnect spinner icon | 12×12, Text-secondary | Icon library / CSS spinner |
| Recap-card icons (🔍 📍 📌) | Emoji, or matching 16×16 line icons if the team wants full visual consistency with icon-library choices elsewhere | Emoji recommended for v1, consistent with the Home/Sign-up specs' asset decisions |
| Cancel-confirmation dialog: no icon needed | — | Text-only dialog is sufficient |

---

## 9. Data & state requirements

| Section | Data / event |
|---|---|
| [B] Reconnect banner | Socket.IO connection-state (client-side `socket.on('disconnect')`/`('connect')`), triggers the 5s `GET /api/v1/requests/:id` poll fallback per §6.3 |
| [C] Live count | `response:update` Socket.IO event (server → `request:<requestId>` room, joined per the JWT-verified room-assignment rules in §3.9A) → count only, no shop-identifying fields, ever, at this stage |
| [D] Countdown | Computed client-side from the request document's `expiresAt` (fetched once on screen mount via `GET /api/v1/requests/:id`, or carried over directly from the New Broadcast screen's successful submit response) — re-synced on every REST poll if the socket has dropped, so a long-disconnected client doesn't drift from the server's actual deadline |
| [E] Recap card | Same initial fetch — `productQuery`, `radiusMeters`, `formattedAddress` from the request document, static for the life of this screen |
| [G] Cancel | `POST /api/v1/requests/:id/cancel` — only reachable/renderable while `status === 'open'`; the button and its confirmation dialog simply don't exist once status has moved on (which also means this screen itself is about to be replaced by one of the Section 4 destinations) |
| Screen-exit | `request:ready` / `request:expired` / `request:cancelled` Socket.IO events (or their REST-poll equivalents, read from the request document's `status` field changing) drive the transitions in Section 4 |

**Loading state:** on first mount, before the initial `GET /api/v1/requests/:id` resolves, show the pulse animation immediately (it doesn't depend on any fetched data — it's purely "this is broadcasting") with the count starting at 0/"No responses yet" and the countdown showing a brief skeleton-style placeholder (a static "--:--" or a subtle shimmer) until `expiresAt` is known — never delay showing the pulse itself waiting on network, since the pulse is the reassurance element and reassurance is most needed at the exact moment this screen first appears.
**Error state:** if the initial fetch fails entirely (not just a socket drop, but the REST call itself erroring), use the shared toast/error-boundary pattern (Master Build Doc §6.1) and offer a retry — this is the one genuine failure mode this screen needs to handle distinctly from a normal socket disconnect, which is already covered gracefully by [B].

---

## 10. Responsive & platform notes

- **Viewport:** centered single-column composition works unchanged from 360px up through tablet/desktop widths (max-width 480px centered, consistent with every other screen in this series) — nothing here needs a different layout at wider viewports, since it's fundamentally a status display, not a data-dense interface.
- **Backgrounding/app-switching:** if the customer backgrounds the PWA and returns, the pulse/countdown must resync against the server's actual `expiresAt` and current count on resume (a re-fetch on visibility-change, not just resuming a client-side timer that may have drifted or missed events entirely while backgrounded) — this matters more here than on most screens, since a stale countdown showing time that's already passed would directly trigger the "does this look broken" problem the zero-state handling in Section 3[D] is designed to avoid.
- **Push notification tie-in:** per the Master Build Doc's note that the customer-side push subscription's first real consumer is dispatch on `awaiting_selection`/`expired` (§ "Fixes since v2.6" list) — if the customer has backgrounded or closed the app entirely, the push notification is what brings them back, at which point they should land directly on whichever destination screen (Section 4) matches the current status, not back on this live-count screen showing stale data.
- **PWA standalone mode:** no bottom tab bar on this screen (Section 2) — same modal-style presentation as the New Broadcast screen, reinforcing that this is a focused, temporary state, not a "place" in the app's normal navigation structure.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 3.9A, 5.3, 6.3) and NEARBY Brand/PWA Guide v1.0 (Sections 1.4, 9.2, 9.5). The single hardest constraint carried through every section above is §3.1's explicit withholding of shop-identifying information during the open window — this screen shows a number and a countdown, and nothing that could be reverse-engineered into "which shops are considering this," by design.*
