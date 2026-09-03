# NEARBY — Shop Live Dashboard / Home Screen Build Specification
### Everything needed to build `/shop/*`'s main screen for an `approved` shop — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1, 3.1.2, 3.4, 4.2, 4.4, 5.3, 6.1, 6.3), NEARBY Brand/PWA Guide v1.0 (§1.4, 3.3), and the Zomato Restaurant Partner / Zepto Franchise app pattern research already established in this series' page-architecture document

---

## 0. Scope — the screen this entire build series has been building toward, and the one hard rule that governs it

Every customer-side screen specced so far exists to produce exactly one event this screen has to answer: `request:new`, fired to the shop's own `shop:<shopId>` room the instant a broadcast lands in range, in-hours, and in-category (§5, "Real-time events"). This is the shop's mirror image of the customer's Live-Count screen — except here, the shop isn't watching a countdown resolve passively, it's the one deciding what happens next.

**The one hard color rule from Brand Guide §3.3 that shapes this entire screen more than anything else:** *"Signal Teal replaces orange as the dominant accent on nav/active states, keeping orange reserved for the Yes/No response buttons only, so the one action that matters is never visually competing with chrome."* This is the first screen in the series where that reservation actually has something to apply to — every other Teal-forward screen so far (Onboarding, Pending, Rejected) had no Yes/No action at all. Here, **orange appears exactly once, on exactly one component** (the incoming-request card's Yes/No buttons), and nowhere else on the page — not on the toggle, not on the stats, not on navigation. That restraint is the entire point of the rule, and it's worth being disciplined about it in a way none of the prior shop-side screens needed to be.

---

## 1. Page goals, in priority order

1. Make the incoming-request feed and its Yes/No decision the unmistakable center of gravity on this screen — everything else here is context or secondary management, per the "Zomato Partner live order management" pattern already identified in this series' original page-architecture research.
2. Put the open/closed toggle front and center, above the fold, one tap to change — per the Zepto Franchise app's own KPI-visibility pattern, a shop's availability status is the single most consequential thing a shopkeeper needs to control quickly.
3. Never let a shop lose visibility into a request it should still be answering — the socket-drop/5s-poll fallback (§6.3's v2.6.1 fix) is a hard requirement, not a nice-to-have, since a missed Yes/No decision has real consequences for both the shop's reliability score and the customer waiting on the other end.
4. Show reliability plainly enough to be useful, without exposing the underlying formula weights — same principle already applied to how the customer-side Shortlist/Shop-Profile screens display reliability, mirrored here from the shop's own point of view.

---

## 2. Layout overview

Single scrollable screen, reached as the default landing screen once `GET /shops/me` (§6.1) returns `status: 'approved'` — this is the first screen in the shop-dashboard series to carry a bottom tab bar, since it's the first genuinely standing "place" this account has.

```
┌─────────────────────────────┐
│ [A] Header (shop name + settings) │
├─────────────────────────────┤
│ [B] Open/Closed toggle          │
├─────────────────────────────┤
│ [C] Reliability & stats row      │
├─────────────────────────────┤
│ [D] Reconnect/offline banner     │  ← conditional
├─────────────────────────────┤
│ [E] Incoming requests feed        │  ← the main content
├─────────────────────────────┤
│ [F] Empty state                   │  ← replaces E when no pending requests
├─────────────────────────────┤
│ [G] Bottom tab bar                 │  ← fixed
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, shop name (left, H2, Text-primary, truncated if long) + a small settings/gear icon (right, 24×24, opens Outlet Management, page-architecture item 2.6).
**Background:** White, 1px Border-gray bottom border on scroll.
**Animation:** none.

### [B] Open/Closed toggle — front and center
**Purpose:** Goal 2, directly — the Zepto Franchise app's own dashboard leads with exactly this kind of availability control, and this product's own FR set treats `isOpen` as a first-class, frequently-changed field (§3.4).
**Layout:** a large, full-width toggle card, Surface white background, 1px Border-gray outline, rounded 12px, sits directly below the header — not a small switch tucked into a settings row, but a prominent, unmistakable control given its own dedicated space.
**Content:** a large label — **"Open"** or **"Closed"** (H1, 24px, 600 SemiBold) — with a real switch control beside it (48×28, standard toggle-switch proportions), plus a small caption beneath: "Customers can see you're open" (when on) or "You won't appear in nearby searches" (when off) — directly and honestly describing the real consequence of the toggle's state, not just labeling it.
**Behavior:** tapping the switch calls `PATCH /shops/me` with `{ isOpen: !current }` immediately — no confirmation dialog, since this is meant to be a fast, frequent action a shopkeeper flips throughout the day (opening in the morning, stepping out briefly, closing at night), and adding friction here would work directly against how this control is actually used.
**Color:** **Signal Teal** fill when "Open" (the switch track and the "Open" label both use Teal — this is the shop surface's dominant accent, and this toggle is the single most prominent place it appears); Gray-300 fill with Text-secondary label when "Closed" — deliberately not Danger red for "Closed," since being closed is a completely normal, expected state, not a problem.
**Note on `openingHours` interaction:** per §3.4, a shop with a configured schedule is only eligible for broadcast-matching during its scheduled hours regardless of this toggle — the caption beneath the toggle should reflect this when relevant, e.g. if the shop has hours configured and it's currently outside them: "Your hours say you're closed right now" as a small additional Caption-size note, so a shopkeeper who manually flips this toggle "Open" outside their configured schedule understands why customers still might not see them.
**Animation:** switch-track color transition on toggle, `duration-instant` (100ms), `ease-standard`; the label text swap ("Open"/"Closed") cross-fades rather than hard-swapping (`duration-quick`, 180ms).

### [C] Reliability & stats row
**Purpose:** Goal 4 — a compact, glanceable summary of the numbers that actually matter to a shopkeeper's business on this platform.
**Layout:** a row of 2–3 small stat cards, equal width, Surface white background, 1px Border-gray outline, rounded 12px each, sits below the toggle.
**Card 1 — Reliability:** reliabilityScore rendered as a friendly label rather than a bare 0–100 number — "Excellent" (≥80), "Good" (60–79), "Building up" (<60 or still at the default 50 with no rated visits yet) — Caption label above, the friendly term below in H2 size, Signal Teal for "Excellent"/"Good," Text-secondary for "Building up" (never Danger, even at the low end — a new or still-recovering shop isn't being penalized visually, just accurately described). Tapping this card opens a short explainer (a simple info sheet, reusing the same plain-language framing already established for the customer-side reliability badges: "based on whether customers who visited you found what they needed, and how quickly you respond").
**Card 2 — Rating:** `avgRating` shown as a star + number (e.g. "★ 4.6"), or "New" if null — identical star-icon gold/amber-tone exception already established on the customer side, reused here.
**Card 3 (optional, if space allows on wider viewports; can be a second row on narrow ones) — Today's responses:** a simple "Yes: X · No: Y" count for the current day, giving the shopkeeper a quick sense of their own activity without needing to open a full history screen.
**Animation:** none — static, glanceable summary content, no live-updating values here (these are point-in-time reads on screen load, not sockets-driven).

### [D] Reconnect/offline banner *(conditional)*
**Purpose:** identical in function to the customer-side Live-Count screen's own reconnect banner — this is the shop-side application of the exact same v2.6.1 fix (§6.3): "if the shop's socket drops, the client polls `GET /shops/me/requests` every 5s so a shop never loses visibility into requests it should currently be answering."
**Copy, layout, color, animation:** identical to the customer-side spec's Section [B] — "Reconnecting… updates may be a few seconds delayed," thin Gray-100 bar, small spinner, slides in/out on connection-state change. Reused directly, not redesigned, since the underlying mechanism and message are the same on both sides of the product.

### [E] Incoming requests feed — the main content, and the one place orange appears
**Purpose:** Goal 1 and Goal 3 combined — this is the actual work of running a shop on NEARBY.
**Card anatomy, one per pending `shopResponses` entry (i.e. every `request:new` the shop hasn't yet answered or that hasn't yet expired):**
- **Top line:** the requested product/item text (`productQuery`, Body, 600 SemiBold, Text-primary) — this is what a customer is asking for, the single most important piece of information on the card.
- **Second line:** distance from the shop (`distanceMeters`, computed at response time but estimable/shown from the request's location at receipt too — Caption, Text-secondary), e.g. "0.6 km away."
- **Countdown:** a small `mm:ss` countdown to the window's close (`requests.createdAt` + the locality's `responseWindowSeconds`, default 120s per §3.11 — the same window the customer is simultaneously watching on their own Live-Count screen), Body size, 600 SemiBold — using the **exact same zero-state handling already specced for the customer-side Live-Count screen's timer** (color shift to Warning amber in the final 10 seconds, no visible freeze at 00:00): reused behavior, not reinvented, since it's functionally the same clock both parties are watching.
- **Two large buttons, side by side, full-width split:** **"Yes"** (filled **Primary orange**, White text) and **"No"** (outline style, White background, 1px Border-gray outline, Text-primary text) — this is the **one and only place Primary orange appears anywhere on this entire screen**, per Section 0's hard rule. The size and prominence of these two buttons should make them impossible to miss — larger than a typical button in this product (52px height), since this is the single action this whole screen exists to enable.
**Behavior:** tapping either button immediately calls `POST /api/v1/requests/:id/respond` with `{ responseValue: 'yes' | 'no' }` (rate-limited 60/shop/hour, §5.3 — a limit generous enough that it should never realistically constrain normal use, so no client-side warning about it is needed) and the card is removed from the feed the instant a response is recorded — no confirmation dialog, since this mirrors the toggle's own "fast, frequent action" philosophy and a shopkeeper needs to move through a feed of possibly-several simultaneous requests quickly.
**Card removal without a response (window closes, or the request is cancelled):** the card disappears from the feed automatically on `request:expired`, `request:ready` (if the shop didn't respond in time, meaning it missed its own window — treated identically to an expiry from this shop's point of view), or `request:cancelled` — the endpoint's own `POST /respond` requiring `status: 'approved'` and an in-window request means a stale card left un-tapped simply becomes non-actionable once the underlying event fires, and the card should reflect that by disappearing rather than sitting there indefinitely as a dead, un-tappable relic.
**Sort order:** newest-first (most recently received `request:new` at the top) — a shopkeeper working through a live feed generally wants to see and act on what's freshest, and this also naturally surfaces the requests with the most time still remaining near the top.
**Animation:** new cards entering the feed (`request:new` arriving) fade in + translateY(10px→0) over `duration-standard` (240ms, `ease-out-soft`) — same list-reveal pattern as every other card-list in this product; a card being removed (responded to, or expired) fades out + collapses its height over `duration-quick` (180ms), rather than snapping away instantly, so the feed doesn't visually jump.

### [F] Empty state *(replaces [E] when there are zero pending requests)*
**Copy:** "No requests right now" (H2, Text-primary) + "We'll notify you the moment a nearby customer needs something you might have." (Body, Text-secondary) — reassuring, not apologetic, matching the calm, non-alarmist tone already established across every empty state in this series.
**Icon:** a simple outline shop/storefront glyph, 64×64, Text-secondary — deliberately not the broadcast-pulse motif (that's reserved for an active, live customer-side broadcast; this shop isn't broadcasting anything, it's waiting to receive).
**Animation:** icon fades in (`duration-standard`, `ease-out-soft`) — same restrained, one-time entrance as every empty state in this series.

### [G] Bottom tab bar
**Tabs:** Home (active) · Requests (history, page-architecture item 2.9) · Reliability (item 2.8, or folded into Profile) · Profile/Settings — the shop-side equivalent of the customer app's own tab structure, adapted to what a shop actually needs day-to-day.
**Active state:** **Signal Teal** icon + label (per the Teal-forward accent rule); inactive: Text-secondary.
**Layout/animation:** identical structural treatment to the customer-side Home page's own tab bar (Section [I] of that spec), just recolored for this surface.

---

## 4. Color usage summary (Brand Guide §3.1/3.3 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| **Signal Teal** | `#00B8A9` | Open-toggle's "Open" state, reliability card's "Excellent"/"Good" labels, active tab-bar icon |
| **Primary orange** | `#FF5A36` | **Exclusively** the incoming-request feed's "Yes" button — nowhere else on this screen |
| Warning | `#FFB627` | Countdown timer's final-10-seconds color shift (reused from the customer-side Live-Count screen's identical rule) |
| Text primary | `#14213D` | Shop name, toggle label, card product text |
| Text secondary | `#5B6472` | Toggle captions, stat-card sub-labels, distance text, empty-state copy |
| Border | `#D8DCE3` | Card outlines, "No" button outline |
| Surface | `#F4F5F7` / `#FFFFFF` | Reconnect-banner background (Gray 100) / cards, header, page background (White) |

**This table is the clearest illustration in the whole build series of Brand Guide §3.3's exact rule in practice** — one accent (Teal) owns the page's chrome and navigation, one accent (Orange) is deliberately restricted to a single, unmistakable action, and neither ever appears where the other belongs.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Shop name (header) | H2 / 18px | 600 SemiBold |
| Toggle label ("Open"/"Closed") | H1 / 24px | 600 SemiBold |
| Toggle caption | Caption / 13px | 400 Regular |
| Stat-card labels | Caption / 13px | 400 Regular |
| Stat-card values | H2 / 18px | 600 SemiBold |
| Request-card product text | Body / 15px | 600 SemiBold |
| Request-card distance/countdown | Caption / 13px / Body / 15px | 400 Regular / 600 SemiBold (countdown numeral bumped, matching the customer-side Live-Count screen's identical treatment) |
| Yes/No button labels | Button / 15px | 600 SemiBold |
| Empty-state headline/body | H2 / 18px / Body / 15px | 600 SemiBold / 400 Regular |
| Reconnect-banner text | Caption / 13px | 400 Regular |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Toggle switch-track transition, Yes/No button tap-scale |
| `duration-quick` | 180ms | Toggle label cross-fade, reconnect-banner slide, card-removal fade-out |
| `duration-standard` | 240ms | New request-card entrance, empty-state icon fade-in |
| `stagger-step` | 40ms | Delay between multiple simultaneous incoming request cards, if several arrive at once |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Toggle transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Card entrance/exit, empty-state entrance |

**No broadcast-pulse animation anywhere on this screen** — that motif belongs exclusively to the customer's own live-broadcast experience; the shop side's equivalent "something is live" signal is the countdown timer itself on each request card, not a looping ambient animation, since a shop dashboard with several simultaneous incoming requests would look visually chaotic with multiple independent pulses running at once.
**Reduced motion:** covered by the app-wide `prefers-reduced-motion` query, same degradation pattern as every other screen.

---

## 7. Images, icons & video — asset list

**No video, no photography** — this is a working dashboard, not a browsing screen; nothing here calls for imagery beyond icons.

| Asset | Format/size | Source |
|---|---|---|
| Settings/gear icon | Line icon, 24×24 | Icon library |
| Toggle switch | Standard switch component, CSS-drawn | — |
| Star-rating icon | 14×14, gold/amber tone | Reused from the customer-side spec's exception |
| Storefront/shop-outline icon (empty state) | Outline icon, 64×64, Text-secondary | Icon library |
| Reconnect spinner icon | 12×12 | Reused from the customer-side Live-Count screen |
| Bottom tab-bar icons | Line icons, 24×24 each | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [A]/[B]/[C] | `GET /api/v1/shops/me` (§6.1) — `shopName`, `isOpen`, `openingHours`, `reliabilityScore`, `avgRating`; loaded once on mount, [B]'s toggle updates optimistically on tap |
| [B] Toggle | `PATCH /api/v1/shops/me` with `{ isOpen }` |
| [D]/[E] | `request:new` Socket.IO event (server → `shop:<shopId>` room) appends a card to the feed; `GET /api/v1/shops/me/requests` (§6.1/6.3) is the REST fallback, polled every 5s while the socket connection is down |
| [E] Yes/No | `POST /api/v1/requests/:id/respond` with `{ responseValue: 'yes' | 'no' }` (rate-limited 60/shop/hour, §5.3) |
| [E] card removal | `request:expired`, `request:ready`, or `request:cancelled` events targeting a request currently shown in the feed |

**Loading state:** brief skeleton (gray placeholder blocks for the toggle card and stat row, plus 1–2 skeleton request cards) while the initial `GET /shops/me` and `GET /shops/me/requests` resolve on first mount.
**Error state:** a failed `PATCH /shops/me` (toggle) reverts the optimistic UI change and shows the shared toast/error-boundary pattern (§6.1) with a retry; a failed `POST .../respond` keeps the card in the feed (rather than removing it as if answered) and shows the same shared error pattern, since silently dropping a request the shop meant to answer would be a real functional problem, not just a cosmetic one.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with the rest of this product; the stat-card row (Section [C]) may wrap to two rows on very narrow viewports rather than compressing three cards uncomfortably into one.
- **Multiple simultaneous requests:** in the pilot's 20–50-shop, single-locality context, a shop is unlikely to face many truly simultaneous incoming requests, but the feed and its stagger-entrance animation (Section 6) are built to handle a handful gracefully regardless — no pagination needed at this scale, consistent with every other pilot-scale list-sizing note in this series.
- **PWA standalone mode:** this is the first shop-dashboard screen to carry the bottom tab bar — an approved shop's account finally has a normal, standing "place" in the product, unlike the Onboarding/Pending/Rejected screens that preceded it.
- **Background/app-switching:** since the countdown timers on incoming-request cards are time-sensitive in the same way the customer-side Live-Count screen's countdown is, this screen should re-sync against server-provided `createdAt` values on app-resume (not trust a client-side timer that may have drifted while backgrounded) — identical reasoning to the customer-side spec's own note on this exact issue.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 3.1.2, 3.4, 4.2, 4.4, 5.3, 6.1, 6.3) and NEARBY Brand/PWA Guide v1.0 (Sections 1.4, 3.3). This is the first screen in the build series where Brand Guide §3.3's orange-reservation rule actually governs a real design decision rather than simply being absent by default — every prior Teal-forward shop screen had no Yes/No action to reserve orange for, and this screen is built with explicit discipline to keep that reservation intact: orange appears on exactly one component, nowhere else.*
