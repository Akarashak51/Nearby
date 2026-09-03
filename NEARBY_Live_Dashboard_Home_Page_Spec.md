# NEARBY — Shop Dashboard: Live Dashboard / Home
### Full Page Build Specification — Content, Layout, Visual Design, Copy, Motion & Data

**Version 1.0**
**Companion to:** NEARBY Master Build Document v2.6.2 (§3.1, §3.1.2, §3.2, §3.4, §5.3, §6.1, §6.3) · NEARBY Brand & PWA Guide v1.0 · NEARBY Page Architecture & Competitor Analysis (§2.4)
**Surface:** `/shop/*` — Teal-forward · **Role:** shop

---

## 0. Scope & How to Use This Document

This document specifies the Live Dashboard / Home screen, page 2.4 in the Page Architecture & Competitor Analysis document — the screen every other page in this series is described as living "alongside," "embedded in," or "reachable from." Rush-Hour Toggle sits as a card on this page (that spec, §3); the Open/Closed switch mirrored on Outlet Management (that spec, §4.1) originates here; the reliability score summary that expands into the full Reliability & Ratings view (that spec, §3) is shown here in compact form. This document is where all of those threads are tied together into the one screen a shop owner opens first, most often, and for the shortest attention span.

Pattern-sourced from the Zomato Restaurant Partner app's "live order management" home tab and Zepto Franchise's "performance visibility" KPI tiles (Page Architecture doc §2.4, pattern source) — reused here for answering broadcast availability rather than fulfilling orders, since NEARBY has no order or delivery leg (Master Build Document §1).

Nothing in this document contradicts the Master Build Document v2.6.2 or the Brand & PWA Guide v1.0, and unlike several pages in this series (Rush-Hour Toggle, Staff Management, Leave Scheduler), this page introduces **no new data fields or endpoints** — every piece of content on it is already fully defined by `GET /shops/me` (§5.3) and `GET /shops/me/requests` (§5.3, NEW v2.6.1).

---

## 1. Overview & Purpose

### 1.1 — What it is

The shop owner's control tower: a single screen combining the one switch that matters most (Open/Closed), a live feed of requests currently awaiting this shop's Yes/No answer, and a glance at how the shop is doing overall (today's counts, current reliability score). It is deliberately not a dashboard of many equally-weighted widgets — it has one dominant action (respond to incoming requests) and everything else arranged in clear support of that.

### 1.2 — Why this page decides the whole app's first impression

Master Build Document §6.1 confirms this is a decision point, not just a landing page: on load, the shop surface calls `GET /shops/me` specifically **to decide whether to show profile creation (page 2.1) or this Live Dashboard** — meaning this page's own logic already has to handle "does this shop even exist yet" before it can render anything. This document assumes that decision has already resolved to "yes, an approved shop exists" — the onboarding, pending, and rejected states are out of scope here and covered by their own pages (2.1–2.3).

### 1.3 — The real-time-with-a-safety-net design constraint

This is the one page in the entire Shop Dashboard where being *wrong for even a few seconds* has a direct commercial cost — a shop that doesn't see an incoming request in time misses its one chance to answer inside the response window (default 120s, §3.2). This shapes nearly every decision in this document: the socket-first, 5-second-poll-fallback architecture (§2.3, already fully defined in Master Build Document §6.3) isn't optional infrastructure to consider adding later, it's the reason this page can be trusted at all on Render's free-tier cold-start conditions (§6.1).

### 1.4 — User story

> *"As a shop owner, I want to open the app and immediately know two things: am I marked open right now, and is there a request waiting for me to answer — without having to dig for either."*

### 1.5 — Pattern source

Inspired by: Zomato Restaurant Partner's "live order management" home tab (real-time incoming orders, accept/reject) and Zepto Franchise's "performance visibility" KPI dashboard (Page Architecture doc §2.4, pattern source). NEARBY's version has a shorter list of KPIs than Zepto Franchise's own dashboard, since NEARBY has one composite reliability metric rather than several operational ones (see the Reliability & Ratings specification in this series, §1.5, for why).

---

## 2. Functional Specification

### 2.1 — Data this page reads (all read-only at the page level; writes happen via specific actions)

| Source | Field(s) | Shown on this page as |
|---|---|---|
| `GET /shops/me` (§5.3) | `isOpen` | The Open/Closed switch's current state |
| `GET /shops/me` (§5.3) | `reliabilityScore` | Compact score summary (§4.4), links to the full Reliability & Ratings view |
| `GET /shops/me` (§5.3) | `rushHourActive`, `rushHourEndsAt` | Rush-Hour card state (fully specified in that document's own §4.1/§4.4 — referenced, not redefined, here) |
| `GET /shops/me/requests` (§5.3, NEW v2.6.1) | Every currently-`open` `shopResponses` entry for this shop, paginated | The incoming-request feed (§4.3) |
| Derived, computed for display only | Today's Yes/No counts | Computed client-side by filtering the feed's own history against "today," or via a lightweight aggregate if the feed payload already includes a same-day count — no new endpoint required either way |

### 2.2 — Real-time events this page listens for (existing, no changes)

Per the Master Build Document's "Real-time events" table (§5) and §6.1's description of the shop dashboard's socket behavior:

| Event | This page's reaction |
|---|---|
| `request:new` | A new card animates into the top of the incoming-request feed (§4.3, §7) |
| `request:cancelled` | If the cancelled request has a pending card in this shop's feed, that card is removed with the standard row-collapse treatment (§7) — matching how a customer's own cancellation is handled system-wide (§3.8) |
| (implicit) countdown expiry on a card the shop never answered | The card is removed once its own response window closes, exactly as it would resolve into "Expired" history on the Request History page (that spec, §2.3) |

### 2.3 — Socket-drop fallback (existing, fully defined — restated for this page's context)

Per Master Build Document §6.3 (FIXED v2.6.1): if the shop's socket connection drops, the client polls `GET /api/v1/shops/me/requests` every 5 seconds so the incoming-request feed never silently goes stale — this closes the specific gap named in that section, where a Render free-tier cold start could otherwise cause a shop to lose visibility into a request it should still be able to answer. This page must visually surface *when* it's in fallback-polling mode (§4.6), not just handle it silently, so a shop owner understands why the feed might feel slightly less instant than usual rather than assuming something is broken.

### 2.4 — Actions available directly from this page

- **Toggle `isOpen`** — calls `PATCH /shops/me` (§5.3), the same endpoint and field the Outlet Management page's mirrored switch uses (that spec, §2.1) — both surfaces write to the identical field, so toggling here and toggling there are the same action with two entry points, not two separate states to keep in sync.
- **Answer Yes/No on an incoming request card** — calls `POST /requests/:id/respond` (existing endpoint, referenced from the Incoming Request card specification, page 2.5, not redefined here since that page owns the full card interaction detail).
- **Rush-Hour on/off** — fully specified in the Rush-Hour Toggle document; this page hosts that card but does not redefine its behavior.

### 2.5 — What this page explicitly does not do

- It does not let the owner edit shop details, hours, or staff from here — those live on their own dedicated pages (2.6, 2.7, 2.10), reachable via nav, not duplicated here.
- It does not show request history (page 2.9 owns that) — this page shows only what is *currently* awaiting an answer, nothing resolved.
- It does not show any competitive information (how many other shops were also notified, etc.) — consistent with this series' repeated note that NEARBY isn't a bidding or competitive-visibility marketplace (see the Request History specification, §1.3).

---

## 3. Where It Lives

- This **is** the shop's default landing screen — the first thing rendered after `GET /shops/me` on `/shop/*` load resolves to an approved shop (§1.2, Master Build Document §6.1).
- Every other Shop Dashboard nav tab (Outlet, Reliability, History, Staff, Notifications, Help, Planned Closures) sits alongside this one as a peer, but this is the tab a shop owner returns to by default when the app is reopened.

---

## 4. Section-by-Section Content & Layout

### 4.1 — Header & Open/Closed switch

The most prominent element on the page — full-width, sits above everything else, including the Rush-Hour card and the incoming feed.

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Shop name | e.g. "Sharma General Store" | H1 | Gray 900 — personalizes the page immediately, reused directly from `shops.shopName` |
| Open/Closed switch | Label: "Open for requests" + large switch | Body / Medium (label), — (switch) | Signal Teal track when on, Gray 300 when off — the single largest, most prominent interactive element on the page, reflecting its status as the one switch that matters most (§1.1) |
| Status caption | "You're open and visible to customers nearby." (on) / "You're closed — customers won't see your shop right now." (off) | Caption | Gray 600 — states the real-world consequence of the switch's position in plain words, not just "On/Off" |

### 4.2 — Rush-Hour card

Embedded directly below the Open/Closed switch, exactly as specified in the Rush-Hour Toggle document's own §3 ("Where It Lives") and §4.1/§4.4 (default and active card states) — this document does not redefine that card's content, only its position on the page: **immediately below §4.1**, above the incoming-request feed, per that document's stated rationale that this is where a shop owner's attention already is.

### 4.3 — Incoming-request feed

The page's main content area, taking up the majority of vertical space — a live-updating list of requests currently awaiting this shop's answer.

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Section header | "Waiting for your answer" | H2 | Gray 900, shown only when at least one card is present |
| Request card (×N) | Full interaction detail owned by the Incoming Request card specification (Page Architecture page 2.5) — this document positions it, doesn't redefine it | — | Each card includes the product query, distance, and a live countdown to the response window's close, per that page's own spec |
| Empty feed state | See §4.7 | — | — |

### 4.4 — Today's snapshot (compact stats row)

A slim, secondary row beneath the incoming-request feed's section header or above it (whichever reads better once built) — three short stats, not full cards, since these are glance-value context, not the page's main draw.

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Stat — Yes today | "{N} Yes today" | Body / Medium numeral, Caption label | Gray 900 numeral, Gray 600 label |
| Stat — No today | "{N} No today" | Body / Medium numeral, Caption label | Gray 900 numeral, Gray 600 label |
| Stat — Reliability score | "{score}/100" + a "View details" link | Body / Medium numeral, Caption link | Numeral color-banded exactly per the Reliability & Ratings specification's own §5.1 rule (Teal 80–100, neutral Gray 50–79, Warning below 50) — this page's compact numeral must never contradict that document's color logic, so it's specified here as a direct reuse, not a new convention |

### 4.5 — Quick links row

A row of small shortcut chips beneath the snapshot stats, for the handful of actions a shop owner might want to jump to without using the main nav — this is a convenience layer, not a replacement for the nav.

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Chip | "Edit hours" | Caption / link chip | Routes to Opening Hours (page 2.7, reached via Outlet Management's tab set) |
| Chip | "Planned closures" | Caption / link chip | Routes to the Leave/Closure Scheduler (page 2.13) |
| Chip | "Help" | Caption / link chip | Routes to Help Centre (page 2.12) |

### 4.6 — Fallback-polling indicator

Per §2.3 — a small, unobtrusive banner shown only while the client has fallen back to 5-second REST polling because the socket connection has dropped:

> **Copy:** *"Reconnecting — your requests are still updating, just a little slower right now."*

Styled as a thin, dismissible-feeling (though not actually dismissible, since it should disappear on its own once reconnected) strip at the top of the incoming-request feed section — never styled as an error, since this is expected, handled behavior (§2.3), not a failure.

### 4.7 — Empty feed state

Shown when there are currently no pending requests awaiting this shop's answer:

> **Copy:** *"No requests waiting right now. We'll notify you the moment a new one comes in."*

A single static instance of the logomark's pin (Brand Guide §9.4 empty-state pattern), consistent with the treatment used across this series' other empty states — no looping animation, since a looping empty state on the page a shop owner checks most often would read as the app being stuck exactly at the moment it matters most to trust that it isn't.

### 4.8 — First-time nudge integration point (cross-referenced from the Opening Hours specification)

Per that document's own §4.7, a dismissible one-time card appears here for a newly-approved shop with no schedule set, prompting "Set your hours" — this page hosts that nudge but does not redefine its content or dismissal logic, which belongs entirely to the Opening Hours document.

---

## 5. Visual Design System Application

### 5.1 — Color mapping

| Token | Used for in this feature | Rationale |
|---|---|---|
| Signal Teal #00B8A9 | Open/Closed switch (on state), "View details" link, quick-link chips | Shop surface's dominant accent (Brand Guide §3.3) — every routine navigational and status action on this page uses it consistently |
| Warning #FFB627 | Rush-Hour card active state (inherited from that specification, not redefined here); below-50 reliability score numeral (inherited from that specification) | Both inherited exactly as their owning documents define them — this page must not introduce a competing meaning for Warning |
| Gray 900 / Gray 600 | Shop name, status captions, stat numerals/labels, empty-state and fallback-banner text | Standard body-text accessibility rule (§3.4), applied consistently across this whole documentation series |
| Gray 300 | Open/Closed switch track (off state), section dividers | Border/inactive-state token (§3.2) |
| Surface / White | Page and card backgrounds | Standard page/card tokens (§3.2) |

| Signal Teal #00B8A9 | Warning #FFB627 | Gray 900 #14213D | Gray 600 #5B6472 | Gray 300 #D8DCE3 |
|---|---|---|---|---|

Orange is not used anywhere on this page, for the same reason as every other Shop Dashboard spec in this series — reserved exclusively for Yes/No response buttons, which live on the Incoming Request card (page 2.5), not on this hosting page itself.

### 5.2 — Typography mapping

| Role (Brand Guide §4.2) | Size / line-height / weight | Used here for |
|---|---|---|
| H1 | 24px / 32px, 600 SemiBold | Shop name header |
| H2 | 18px / 26px, 600 SemiBold | "Waiting for your answer" section header |
| Body | 15px / 22px, 400 Regular (Medium for the switch label and stat numerals) | Open/Closed switch label, snapshot stat numerals |
| Caption | 13px / 18px, 400 Regular | Status captions, stat labels, quick-link chip text, fallback-polling banner, empty-state copy |

### 5.3 — Spacing, grid & component styling

- Header + switch block: 24px top padding (more generous than standard, since this is the page's focal point), full-width, no card border — this reads as the page's own header, not a card floating on it.
- Rush-Hour card: exactly as specified in that document's own §5.3 — 16px corner radius, 16px padding, positioned with 16px vertical gap above and below relative to the switch block and the feed section.
- Snapshot stats row: three equal-width columns, no dividers between them, 16px vertical padding above and below the row.
- Quick-link chips: horizontally laid out, 8px gap, pill-shaped, Gray 100 fill, Gray 900 text — a lighter, more casual treatment than a primary button, since these are shortcuts, not the page's main actions.
- Incoming-request feed: full-width cards (each card's own internal styling owned by the Incoming Request card specification), 12px vertical gap between cards.
- Fallback-polling banner: thin strip, 8px vertical padding, Gray 100 background, no border, sits directly above the feed section header.

### 5.4 — Iconography

- No new custom iconography introduced by this page specifically — the Open/Closed switch, quick-link chips, and stats row all use standard system-style affordances (switch control, chip shape, plain numerals) rather than custom glyphs.
- The logomark pin (existing brand asset) appears once, statically, in the empty feed state (§4.7), consistent with its use across the Reliability & Ratings, Request History, Staff Management, and Leave Scheduler specifications in this series.

---

## 6. Full Copy Deck

| State / Location | Exact copy |
|---|---|
| Open switch label | Open for requests |
| Status — open | You're open and visible to customers nearby. |
| Status — closed | You're closed — customers won't see your shop right now. |
| Feed section header | Waiting for your answer |
| Stat — Yes today | {N} Yes today |
| Stat — No today | {N} No today |
| Stat — reliability | {score}/100 |
| Stat — reliability link | View details |
| Quick link — hours | Edit hours |
| Quick link — closures | Planned closures |
| Quick link — help | Help |
| Fallback-polling banner | Reconnecting — your requests are still updating, just a little slower right now. |
| Empty feed state | No requests waiting right now. We'll notify you the moment a new one comes in. |
| Load error | Couldn't load your dashboard. Pull down to try again. |

---

## 7. Motion & Animation Specification

Every token below is reused from the existing Motion Token table (Brand Guide §9.1) — nothing new is introduced. This is a real-time page, so motion carries genuine informational weight here (a new card appearing *is* the notification, for anyone looking at the screen when it happens) — but per the calm-tone rule (Brand Guide §1.4), it must still never read as manufactured urgency.

| Interaction | Token(s) used | Behavior |
|---|---|---|
| Open/Closed switch flip | duration-instant (100ms) | Standard switch-flip animation, identical mechanics to every other toggle across this series |
| New incoming-request card arrives (`request:new`) | duration-standard (240ms), ease-out-soft | Card slides and fades in at the top of the feed — a clear, noticeable-but-not-alarming entrance, since this is the single most important live moment on this page |
| Request card removed (cancelled, expired, or answered) | duration-quick (180ms), ease-standard | Card fades out and collapses its height smoothly, remaining cards shift up to fill the gap |
| Fallback-polling banner appear/disappear | duration-quick (180ms), ease-out-soft | Slides down from above the feed section on socket drop, slides back up on reconnect — same easing family as toasts across this series |
| Reliability score numeral (snapshot stat) | No animation — instant value swap | Same explicit rule as the Reliability & Ratings view's own headline numeral (that spec, §7) — never an animated count-up, for the same reason: dramatizing movement could feel punitive or falsely celebratory |
| Rush-Hour card state changes | Fully owned by that specification's own §7 — not redefined here |
| Reduced motion | Single global override (§9.5) | All of the above collapse to instant state changes — no page-specific reduced-motion work required |

---

## 8. Imagery, Graphics, Icons & Video

No photography or video anywhere on this page. The only illustrative asset is the existing logomark pin, used once, statically, in the empty feed state (§4.7, §5.4) — matching the same treatment established across the Reliability & Ratings, Request History, Staff Management, and Leave Scheduler specifications, so every "nothing here right now" moment in the Shop Dashboard feels like one consistent visual language rather than five different empty-state treatments.

No skeleton-loader design beyond the standard shimmer pattern already defined in the Brand Guide (§9.4) — applied to the switch block, snapshot stats, and feed section independently, so each part of the page can render as soon as its own data resolves rather than blocking on the slowest piece.

---

## 9. States & Edge Cases

| Case | Behavior |
|---|---|
| Brand-new approved shop, first ever visit to this page | Open/Closed switch defaults to whatever `isOpen` value was set at onboarding (owned by the Onboarding specification, page 2.1, not redefined here); empty feed state shown (§4.7); snapshot stats show 0/0 and the default reliability score of 50, exactly as that value is explained on the Reliability & Ratings view (that spec, §4.1) |
| Socket connects successfully | No fallback banner shown; feed updates arrive via real-time events (§2.2) with no visible polling behavior |
| Socket drops mid-session | Fallback banner appears (§4.6); feed continues updating via 5-second polling (§2.3) — functionally correct, just slightly less instant, and the banner exists specifically so the owner doesn't mistake this for the app being broken |
| Shop toggles Closed while a request card is still showing in the feed | The card remains visible and answerable until its own window closes or the shop answers it — going Closed stops *new* matches (§3.4) but does not retract a request the shop was already notified about, consistent with how Rush-Hour handles the same distinction (that spec, §2.1) |
| Shop status becomes blocked while viewing this page | Per the Master Build Document's blocking behavior (§3.5, §3.8), the shop is immediately excluded from all future matching; this page should show a persistent, non-dismissible banner — "Your shop has been blocked. Contact support for details." — and disable the Open/Closed switch, consistent with how the Outlet Management specification handles the same scenario (that spec, §9) |
| `GET /shops/me` or `GET /shops/me/requests` fails on load (network/5xx) | Skeleton placeholders remain, replaced by an inline error state: "Couldn't load your dashboard. Pull down to try again." (§6), with pull-to-refresh still available |
| A request in the feed expires while the owner is actively looking at the screen | The card animates out per §7's "Request card removed" row at the exact moment its countdown reaches zero — no separate confirmation or toast needed, since the countdown itself was the warning |

---

## 10. Accessibility

- The Open/Closed switch has a clear, programmatically-associated label ("Open for requests") and announces its current state (`aria-checked`) to screen readers.
- The fallback-polling banner is announced via `aria-live="polite"` when it appears or disappears, so a screen-reader user understands why the feed's update cadence might feel different, without an intrusive interruption.
- New incoming-request cards are announced via `aria-live="polite"` on the feed container, so a screen-reader user is notified when a new request arrives without needing to actively re-check the page — critical given the short response window (§1.3).
- The reliability score's color-banding is never the only signal — the numeral itself always carries the same information in text, consistent with that specification's own accessibility rule (§10 of that document).
- Minimum 44×44px touch target on the switch, every quick-link chip, and the "View details" link.
- Reduced motion: fully covered by the existing global override (§7) — no page-specific work required.

---

## 11. Admin Visibility & Data Management

- This page has no direct admin-facing counterpart — it is the shop owner's own operational view, and everything shown on it (isOpen, reliability score, today's counts) is already visible to admin through the All Shops directory (Page Architecture page 3.4) at the underlying data level.
- No interaction with the Reports queue (page 3.6) — this is an operational dashboard, not an action surface that could generate a report.
- The blocked-shop banner behavior (§9) is the one place this page's content is directly shaped by an admin action (`PATCH /admin/shops/:id/block`, Master Build Document §3.5) — but the page itself has no admin controls; it only reflects that state once set elsewhere.

---

## 12. Component Inventory — Developer Handoff

| Component | Type | States | Key tokens |
|---|---|---|---|
| ShopHeaderSwitch | Composite (name + switch + status caption) | open, closed, blocked (disabled) | H1, Body, Caption, Signal Teal / Gray 300, duration-instant |
| RushHourCard | Reused from the Rush-Hour Toggle specification | default, active | Owned entirely by that document |
| IncomingRequestFeed | List container | populated, empty, fallback-polling | H2, duration-standard (new card), duration-quick (removed card) |
| RequestCard | Reused from the Incoming Request card specification (page 2.5) | Owned entirely by that page's own spec | — |
| TodaySnapshotRow | Composite (3 stats) | populated | Body, Caption, reliability color-band inherited from that spec |
| QuickLinksRow | Chip row | populated (fixed 3 chips) | Caption, Gray 100 |
| FallbackPollingBanner | Inline banner | hidden, visible | Caption, Gray 100, duration-quick |
| DashboardEmptyState | Composite (logomark + copy) | visible only when feed is empty | Body, logomark pin (static) |
| Toast (reused) | Existing shared component | load-error | duration-quick, ease-out-soft |

Six page-specific composites, two directly reused components owned by their own specifications (Rush-Hour Toggle, Incoming Request card), and one reused shared component — no new design-system primitive is introduced by this page; its entire job is arranging and hosting pieces this series has already fully specified elsewhere.

---

## Appendix A — Quick Reference: What's New vs. Reused

| New for this feature | Reused as-is from existing docs |
|---|---|
| Nothing — no new fields, endpoints, or business logic | Every color token (§3.2, Brand Guide) |
| Page layout and composition (§4 of this document) — arranging existing pieces | Every type-scale role (§4.2, Brand Guide) |
| Copy deck (§6 of this document) | `GET /shops/me` and `GET /shops/me/requests` endpoints, unchanged (§5.3, Master Build Document) |
| Today's-snapshot stat computation (§2.1) — display-only, no new data | Socket-drop 5-second polling fallback, unchanged (§6.3, Master Build Document) |
| | Rush-Hour Toggle card and Incoming Request card, both fully owned by their own specifications in this series |
| | Reliability score color-banding rule, unchanged (Reliability & Ratings specification, §5.1) |
| | Empty-state pattern using the static logomark pin (§9.4, Brand Guide) |
| | Reduced-motion global override (§9.5, Brand Guide) |

---

*— End of specification —*
