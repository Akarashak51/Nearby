# NEARBY — Push Notification Settings
### Full Page Build Specification — Content, Layout, Visual Design, Copy, Motion & Data

**Version 1.0**
**Companion to:** NEARBY Master Build Document v2.6.2 (§3.1, §4.6, §5.3, FR-6.3/UC-19) · NEARBY Brand & PWA Guide v1.0 · NEARBY Page Architecture & Competitor Analysis (§1.14, §2.11)
**Surfaces:** `/app/*` (Orange-forward, page 1.14) **and** `/shop/*` (Teal-forward, page 2.11) — one shared spec, two accent themes
**Roles:** customer, shop

---

## 0. Scope & How to Use This Document

This document specifies Push Notification Settings, which appears **twice** in the Page Architecture & Competitor Analysis document — page 1.14 on the customer app and page 2.11 on the shop dashboard — described identically in both listings as managing "the push subscription that now has a real consumer." This is deliberately written as **one shared specification** rather than two separate documents, because the underlying mechanism, data model, and content structure are identical on both surfaces; only the accent color (Brand Guide §3.3) and a handful of copy strings differ. Every section below calls out the two or three places customer and shop diverge.

Unlike Staff Management, this page is **fully buildable from the Master Build Document as it stands today** — `pushSubscriptions` (§4.6) already exists as a schema, `POST /api/v1/push/subscribe` already exists as an endpoint (§5.3), and `services/push.js` already dispatches to both shop and customer subscriptions on the relevant real-time events (§3.1, FIXED v2.6.2). This page's entire job is to give the shop or customer a legible on/off control and status view over a mechanism that already runs end-to-end — no new backend logic is proposed here beyond one small, explicitly-marked addition for unsubscribing cleanly (§2.4).

Pattern-sourced from the standard browser/app notification-permission prompt pattern, generalized (Page Architecture doc §1.14, pattern source).

---

## 1. Overview & Purpose

### 1.1 — What it is

A settings screen where a customer or shop owner can see whether push notifications are currently enabled on their device, turn them on (triggering the browser's native permission prompt) or off, and understand — in plain language — exactly what they'll be notified about. It is the human-facing control panel over the `pushSubscriptions` collection (§4.6) and the two real-time moments `services/push.js` actually fires on (§1.2 below).

### 1.2 — What a push notification means on each surface (the core content problem this page solves)

Push notifications mean genuinely different things to a customer and a shop owner, and this page's copy must reflect that rather than using one generic "get notified" description on both surfaces:

- **Customer:** notified when their **broadcast's response window closes** — either because the window expired with no selection made, or (per the FIXED v2.6.2 correction) the instant their request reaches `awaiting_selection`, so they know it's time to come back and pick a shop even if they weren't watching the live-count screen (page 1.5).
- **Shop:** notified the instant a **new broadcast request matches them** (`request:new`, §3.1) — this is the higher-stakes case of the two, since a shop has a short response window (default 120s, §3.2) and a missed push could mean a missed opportunity to answer at all.

### 1.3 — Why this page needs to exist at all, given browsers already have their own permission UI

The browser-native permission prompt only ever asks "allow notifications from this site: yes/no" — it has no concept of NEARBY's two specific notification types, no way to explain *why* the person is being asked, and no in-app way to check current status or turn things back on after a decline without digging through browser site-settings. This page exists to wrap that blunt browser primitive in NEARBY's own context, exactly as the Page Architecture doc's pattern-source note describes: "standard app notification-permission prompt, generalized" — the generalization *is* this page.

### 1.4 — User story (both surfaces)

> **Customer:** *"I want to know the moment shops start responding to my request, without having to keep the app open and stare at the countdown."*
>
> **Shop:** *"I don't want to miss a request because I had the app in the background — I need my phone to buzz the second someone's looking for what I sell."*

### 1.5 — Pattern source

Inspired by: the standard browser/PWA notification-permission-prompt pattern, generalized into an in-app settings screen (Page Architecture doc §1.14, §2.11, pattern source) — neither Blinkit, Zepto, nor Zomato's public-facing screenshots were the specific inspiration here, since this is closer to a platform-level (browser) UX convention than a competitor-specific feature; NEARBY's contribution is purely the plain-language wrapping described in §1.3.

---

## 2. Functional Specification

### 2.1 — Data this page reads and writes

| Source | Field(s) | Role on this page |
|---|---|---|
| `pushSubscriptions` (§4.6) | `endpoint`, `keys.p256dh`, `keys.auth` | Written via `POST /push/subscribe` (§5.3) when the person enables push — the client obtains these from the browser's own `PushManager.subscribe()` call, this page never constructs them itself |
| `pushSubscriptions` (§4.6) | `userId` | Implicit — resolved from the caller's JWT, exactly like every other user-scoped write in the Master Build Document (§5.3 pattern) |
| Browser `Notification.permission` (not a NEARBY field — a native browser API) | `default` / `granted` / `denied` | Read on page load to determine which of the three states in §4 to render — this is the actual source of truth for whether push *can* work at all, independent of whether a `pushSubscriptions` row exists yet |

### 2.2 — The three states this page must distinguish (critical to get right)

A person's notification status is the combination of two independent things — the **browser's own permission** and whether NEARBY has actually **stored a subscription** for this device — and conflating them is the most common way this kind of settings page goes wrong:

| Browser permission | Subscription stored? | What this means | What the page shows |
|---|---|---|---|
| `default` (never asked) | No | Person hasn't been asked yet | **Off** state, with an enable action that triggers the browser prompt |
| `granted` | Yes | Fully working | **On** state |
| `granted` | No | Rare but possible — e.g. permission was granted once but the subscribe call failed or the endpoint was later invalidated by the browser | **Off** state, but the enable action here **skips the browser prompt** (permission is already granted) and goes straight to `POST /push/subscribe` |
| `denied` | Irrelevant | Person actively declined at the OS/browser level; NEARBY cannot re-prompt programmatically — this is a hard browser restriction, not a NEARBY limitation | **Blocked** state (§4.3) — this page cannot turn it on directly and must instead explain how to re-enable it in browser/device settings |

### 2.3 — API contract (existing, no changes needed for enabling)

`POST /api/v1/push/subscribe` — Bearer — `{ endpoint, keys: { p256dh, auth } }` — stores/updates this device's `pushSubscriptions` entry (§5.3, NEW v2.6). This page's "turn on" action is a thin UI layer over: (1) call the browser's `PushManager.subscribe()`, which triggers the native permission prompt if not already granted, then (2) send the resulting subscription object to this existing endpoint.

### 2.4 — Turning off (small, explicitly-marked addition)

The Master Build Document defines subscribing but not unsubscribing. This page needs a **DELETE /api/v1/push/subscribe** endpoint — **NEW**, minor addition, following the exact same auth and body pattern as its sibling:

| Endpoint | Auth | Body / Notes |
|---|---|---|
| `DELETE /api/v1/push/subscribe` | Bearer | `{ endpoint }` — removes this device's `pushSubscriptions` entry — **NEW**, not in the Master Build Document's current endpoint table, but a natural, small counterpart to the existing `POST /push/subscribe` row (§5.3) |

On the client, turning the toggle off also calls the browser's own `PushSubscription.unsubscribe()` so the device itself stops being subscribed at the browser level, not just removed from NEARBY's database — both halves matter, or the device could still receive pushes NEARBY no longer intends to send if the local unsubscribe is skipped.

### 2.5 — What this page does not do

- It does not let a person choose granular notification types (e.g. "only notify me about X, not Y") — each surface has exactly one meaningful notification type in the current product (§1.2), so a single on/off toggle is the entire control surface needed; a per-type matrix would be over-building for a product with one event per role.
- It does not show notification history or a log of past pushes — that's out of scope for a settings page; a customer's own request outcomes are already visible on Request History (customer page 1.12) and a shop's on Request History (shop page 2.9).
- It never silently re-prompts for permission — every enable action is a deliberate tap, never triggered automatically on page load or app open, respecting the browser's own anti-spam expectations around permission prompts.

---

## 3. Where It Lives

- **Customer:** Profile / Account settings (page 1.17) → "Notifications" row → this page.
- **Shop:** reachable from the Shop Dashboard nav's settings area, alongside Help Centre (page 2.12) — grouped with other account-level (not shop-operational) settings, distinct from the operational pages like Outlet Management or Rush-Hour.
- Both are a single, non-tabbed screen — there's exactly one setting to manage per surface (§2.5), so no sub-navigation is needed.

---

## 4. Section-by-Section Content & Layout

### 4.1 — Page header & explainer

| Element | Content / Copy (customer) | Content / Copy (shop) | Type role | Notes |
|---|---|---|---|---|
| Page header | "Notifications" | "Notifications" | H1 | Gray 900 — identical wording on both surfaces |
| Explainer | "Get notified the moment shops respond to your request, so you don't have to keep watching the countdown." | "Get notified the instant a request matches your shop, so you don't miss the window to respond." | Body | Gray 600 — this is the single most important string on the page, since it's the plain-language answer to "what am I actually turning on" (§1.2) |

### 4.2 — Main toggle card (On / Off states)

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Toggle label | "Push notifications" | Body / Medium | Gray 900 |
| Toggle switch | On/off switch | — | Surface accent color when on (Orange #FF5A36 on customer per Brand Guide §3.3, Signal Teal #00B8A9 on shop) — see §5.1 for the full rationale on why this is the one page in the series where Orange legitimately appears on the shop surface's sibling customer version |
| Status caption (on) | "You'll be notified on this device." | Caption | Gray 600 |
| Status caption (off, browser permission not yet requested) | "Turn this on to get notified." | Caption | Gray 600 |

### 4.3 — Blocked state (browser permission denied)

Shown instead of the toggle when `Notification.permission === 'denied'` (§2.2) — this state cannot be fixed by any in-app action, so the page must clearly redirect the person to the one place that *can* fix it.

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Status banner | "Notifications are blocked for NEARBY in your browser settings." | Body | Gray 900, on a Gray 100 background — informational, not styled as an error, since this is a browser-level state the person themselves chose at some point, not something NEARBY got wrong |
| Instructions | "To turn them back on, open your browser or phone's notification settings and allow notifications for NEARBY." | Caption | Gray 600 |
| Action | "Open settings" (where the platform supports deep-linking to notification settings; otherwise omitted and the instructions alone suffice) | Button / SemiBold, outline style | Surface accent color outline |

### 4.4 — Unsupported-environment state

Shown when the browser/device doesn't support the Push API at all (e.g. certain in-app browsers, unsupported older browsers):

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Status banner | "Push notifications aren't supported in this browser." | Body | Gray 900, Gray 100 background |
| Suggestion | "Try opening NEARBY in Chrome or Safari, or check back after installing NEARBY to your home screen." | Caption | Gray 600 — references the PWA install prompt (Page Architecture §4), since installing as a PWA resolves push support on some platforms that don't support it in a plain browser tab |

### 4.5 — "What you'll be notified about" detail (expandable, optional read-more)

A single collapsed row beneath the main toggle, expandable for the person who wants the full picture rather than just the one-line explainer in §4.1:

| Element | Content / Copy (customer) | Content / Copy (shop) | Type role |
|---|---|---|---|
| Row label | "What you'll be notified about" | "What you'll be notified about" | Caption / link |
| Expanded detail | "When shops finish responding to your request, or when time runs out to pick one." | "The moment a new request matches your shop and you have a limited time to answer." | Caption |

### 4.6 — Confirmation toasts

| Action | Copy |
|---|---|
| Turned on successfully | "Notifications turned on." |
| Turned off | "Notifications turned off." |
| Browser prompt declined | "You didn't allow notifications — you can try again anytime." |
| Enable failed (network/device error) | "Couldn't turn on notifications. Try again." |

---

## 5. Visual Design System Application

### 5.1 — Color mapping (two-surface table)

| Token | Customer surface (`/app/*`) | Shop surface (`/shop/*`) | Rationale |
|---|---|---|---|
| Surface accent (toggle-on color) | Orange #FF5A36 | Signal Teal #00B8A9 | Per Brand Guide §3.3 — each surface's dominant accent drives its primary interactive controls; this page's toggle is exactly that kind of control on each surface, so it correctly takes the surface's own accent rather than a fixed neutral color |
| Informational banners (§4.3, §4.4) | Gray 100 fill, Gray 900 text, on both surfaces | Same | Deliberately surface-neutral — a blocked/unsupported browser state isn't a customer- or shop-specific concept, so it doesn't need surface theming |
| Body / Caption text | Gray 900 / Gray 600, both surfaces | Same | Standard body-text accessibility rule (§3.4), applied identically on both surfaces per this entire documentation series' convention |

| Orange #FF5A36 (customer accent) | Signal Teal #00B8A9 (shop accent) | Gray 100 #F4F5F7 | Gray 900 #14213D | Gray 600 #5B6472 |
|---|---|---|---|---|

Danger and Warning tokens are not used anywhere on this page on either surface — even the blocked/unsupported states (§4.3, §4.4) are deliberately framed as neutral, informational states rather than errors, since neither is something the person did wrong.

### 5.2 — Typography mapping (identical on both surfaces)

| Role (Brand Guide §4.2) | Size / line-height / weight | Used here for |
|---|---|---|
| H1 | 24px / 32px, 600 SemiBold | Page header ("Notifications") |
| Body | 15px / 22px, 400 Regular (Medium for the toggle label) | Explainer text, toggle label, blocked/unsupported banner text |
| Caption | 13px / 18px, 400 Regular | Status captions, expandable detail row and its content, instructions text |
| Button | 15px / 20px, 600 SemiBold | "Open settings" action |

### 5.3 — Spacing, grid & component styling

- Toggle card: full-width, 16px padding, 16px corner radius, white surface with 1px Gray 300 border — matches the standard card language used across both surfaces in this series (e.g. the Rush-Hour Toggle card on shop, or any standard settings row on customer).
- Explainer text: sits directly under the H1 with 12px gap, full-width, wraps to 2 lines maximum before truncation is acceptable (though in practice both copy variants in §4.1 fit comfortably on 2 lines at standard mobile width).
- Blocked/unsupported banners: 16px padding, 8px corner radius, Gray 100 fill, no border — a lighter visual weight than the toggle card, since these are informational asides rather than the page's primary control.
- Expandable "what you'll be notified about" row: 44px minimum touch target height even though its content is short, consistent with the accessibility minimum applied throughout this series.

### 5.4 — Iconography

- Bell icon (outline when off, filled when on) — 24×24, leading the toggle card, colored to match the current state (Gray 600 when off, surface accent when on) — the one small piece of custom-feeling iconography on this page, though it's a standard, widely-recognized bell glyph rather than a custom brand asset.
- Chevron on the expandable "what you'll be notified about" row (16×16, Gray 600), rotating on expand — same reused glyph and rotation behavior as the Request History page's expand affordance (that spec, §5.4, §7).
- No new custom iconography is introduced.

---

## 6. Full Copy Deck

| State / Location | Customer copy | Shop copy |
|---|---|---|
| Page header | Notifications | Notifications |
| Explainer | Get notified the moment shops respond to your request, so you don't have to keep watching the countdown. | Get notified the instant a request matches your shop, so you don't miss the window to respond. |
| Toggle label | Push notifications | Push notifications |
| Status — on | You'll be notified on this device. | You'll be notified on this device. |
| Status — off (not yet requested) | Turn this on to get notified. | Turn this on to get notified. |
| Blocked banner | Notifications are blocked for NEARBY in your browser settings. | Notifications are blocked for NEARBY in your browser settings. |
| Blocked instructions | To turn them back on, open your browser or phone's notification settings and allow notifications for NEARBY. | Same |
| Blocked action | Open settings | Open settings |
| Unsupported banner | Push notifications aren't supported in this browser. | Same |
| Unsupported suggestion | Try opening NEARBY in Chrome or Safari, or check back after installing NEARBY to your home screen. | Same |
| Expandable row label | What you'll be notified about | Same |
| Expandable detail | When shops finish responding to your request, or when time runs out to pick one. | The moment a new request matches your shop and you have a limited time to answer. |
| Toast — enabled | Notifications turned on. | Same |
| Toast — disabled | Notifications turned off. | Same |
| Toast — prompt declined | You didn't allow notifications — you can try again anytime. | Same |
| Toast — enable failed | Couldn't turn on notifications. Try again. | Same |

---

## 7. Motion & Animation Specification

Every token below is reused from the existing Motion Token table (Brand Guide §9.1) — nothing new is introduced.

| Interaction | Token(s) used | Behavior |
|---|---|---|
| Toggle on/off | duration-instant (100ms) | Standard switch-flip animation, identical mechanics to every other toggle across this series (Rush-Hour's day toggles, Opening Hours' day-on switches) |
| Bell icon state change (outline ↔ filled) | duration-quick (180ms), crossfade | Icon crossfades between its outline and filled variant in sync with the toggle animation, no separate bounce or pop |
| Expandable detail row reveal | duration-quick (180ms), ease-out-soft | Row expands/fades in below the label, chevron rotates 180° in sync — identical pattern to the Request History row expand (that spec, §7) |
| Confirmation toast | duration-quick (180ms), ease-out-soft | Standard bottom-edge toast (§9.3) |
| Blocked/unsupported banner appearance (on page load, if applicable) | No animation — renders directly in its final state | These are page-load-time states, not something the person triggers mid-session, so there's nothing to animate into (consistent with the "no loading flicker to design for" reasoning used elsewhere in this series for similarly data-ready-on-load pages) |
| Reduced motion | Single global override (§9.5) | All of the above collapse to instant state changes — no page-specific reduced-motion work required |

---

## 8. Imagery, Graphics, Icons & Video

No photography, illustration, or video anywhere on this page, on either surface. The only icon is the standard bell glyph (§5.4) — a universally recognized system icon, not a custom brand asset, appropriate for a settings page whose whole job is quick comprehension rather than brand moment-making.

No skeleton loader is needed — this page's content depends only on a synchronous browser API read (`Notification.permission`) and, if a subscription exists, a value already available from the same session/profile data loaded elsewhere in the app; there's no network round-trip blocking the page's initial render.

---

## 9. States & Edge Cases

| Case | Behavior |
|---|---|
| Person taps "turn on," browser prompt appears, they tap Allow | `PushManager.subscribe()` resolves, client calls `POST /push/subscribe` (§2.3), toggle flips to on, success toast shown |
| Person taps "turn on," browser prompt appears, they tap Block | Browser permission becomes `denied`; toggle stays off; toast: "You didn't allow notifications — you can try again anytime" (§6) — note this specific toast is shown once, immediately, distinct from the persistent Blocked state (§4.3) which appears on every subsequent page visit until the browser-level permission changes |
| Person had notifications on, then revokes permission from browser/OS settings directly (outside the app entirely) | Next time this page loads, `Notification.permission` reads `denied`; page renders the Blocked state (§4.3) even though NEARBY's own `pushSubscriptions` record may still exist server-side — a stale subscription is harmless (pushes to a revoked endpoint simply fail silently on the server side, per how `services/push.js` / the underlying `web-push` library already handles delivery failures), but see the next row for graceful cleanup |
| Stale/invalid subscription discovered server-side (delivery fails permanently) | **Recommended (not a page-level concern, backend housekeeping):** `services/push.js` can call the new `DELETE /push/subscribe` internally when the push provider reports the endpoint as permanently gone, keeping the `pushSubscriptions` collection tidy — flagged here as a natural extension of §2.4's new endpoint, not required for this page's own UI to function correctly |
| Multiple devices for the same person | Each device has its own `endpoint` (§4.6 schema) and therefore its own subscription row — this page's toggle only ever reflects and controls *this device's* subscription state, never a person's notifications across all their devices at once; switching this off on a phone doesn't affect a subscription made from a desktop browser, and the page's copy doesn't need to explain this explicitly since "on this device" already appears in the on-state caption (§4.2) |
| Person uninstalls the PWA / clears browser data | The subscription becomes orphaned exactly as in the stale-subscription row above — no special handling needed on this page, since the person simply won't see it again unless they reinstall and revisit |
| DELETE call fails (network/5xx) while turning off | Toggle optimistically shows off in the UI, but a background retry (or a visible error toast, "Couldn't turn on notifications. Try again." reworded for the off case — recommend adding a matching off-failure toast at implementation time) should confirm the unsubscribe actually completed server-side, since a failed DELETE leaves a stale-but-still-active subscription that could still receive pushes unexpectedly |

---

## 10. Accessibility

- The toggle switch has a clear, programmatically-associated label ("Push notifications") and announces its current state (`aria-checked`) to screen readers, not just a bare visual switch.
- Bell icon state (outline/filled) is never the sole signal of on/off status — the toggle switch's own state and the status caption text both carry the same information redundantly.
- Blocked and unsupported banners use a real heading or strong text emphasis on their lead sentence, so screen-reader users navigating by content structure don't miss that the page is in a non-default state.
- The expandable detail row uses `aria-expanded` on its trigger, matching the pattern already established on the Request History page (that spec, §10).
- Minimum 44×44px touch target on the toggle switch itself, the expandable row, and the "Open settings" action.
- Reduced motion: fully covered by the existing global override (§7) — no page-specific work required.

---

## 11. Admin Visibility & Data Management

- No admin-facing surface is needed for this feature specifically — `pushSubscriptions` (§4.6) is an infrastructure-level collection (device tokens), not something an admin would ever need to inspect per-person the way they inspect shop profiles or reports.
- No interaction with the Reports queue (page 3.6) — a person managing their own notification preference cannot affect another person.
- **Analytics note (no new build):** if ever useful, a count of active `pushSubscriptions` could be added to `GET /admin/analytics` (§5.4) as a rough proxy for engagement depth — noted here as a possible future metric only, consistent with how this documentation series flags natural extensions without building them prematurely.

---

## 12. Component Inventory — Developer Handoff

| Component | Type | States | Key tokens |
|---|---|---|---|
| PushSettingsHeader | Static header + explainer | customer-copy, shop-copy | H1, Body |
| PushToggleCard | Composite (icon + label + switch + status caption) | off-unrequested, on, off-after-decline | Body, Caption, Orange (customer) / Signal Teal (shop), duration-instant |
| PushBlockedBanner | Informational banner | visible only when `Notification.permission === 'denied'` | Body, Caption, Gray 100, Button (outline) |
| PushUnsupportedBanner | Informational banner | visible only on unsupported browsers/devices | Body, Caption, Gray 100 |
| NotifiedAboutExpandable | Expandable detail row | collapsed, expanded | Caption, duration-quick |
| Toast (reused) | Existing shared component | enabled, disabled, declined, error | duration-quick, ease-out-soft |

Five page-specific composites, each theme-aware (reading the surface's own accent token rather than a hardcoded color) plus one reused shared component — no new design-system primitive is introduced, and the same five components serve both the customer and shop instances of this page with only copy and accent-color props differing.

---

## Appendix A — Quick Reference: What's New vs. Reused

| New for this feature | Reused as-is from existing docs |
|---|---|
| `DELETE /api/v1/push/subscribe` endpoint (§2.4) — small, explicitly-marked addition | Every color token (§3.2, Brand Guide), applied per-surface per §3.3 |
| Client-side three-state permission/subscription model (§2.2) — a UI-layer concept over existing data, no schema change | Every type-scale role (§4.2, Brand Guide) |
| Copy deck for both surfaces (§6 of this document) | `pushSubscriptions` collection, unchanged (§4.6, Master Build Document) |
| Blocked/unsupported informational states (§4.3, §4.4) | `POST /api/v1/push/subscribe` endpoint, unchanged (§5.3, Master Build Document) |
| | `services/push.js` dispatch logic for both shop (`request:new`) and customer (`awaiting_selection`/`expired`) events, unchanged (§3.1, Master Build Document) |
| | Expandable-row and toast interaction patterns (§9.3, Brand Guide — reused directly from the Request History specification in this series) |
| | Reduced-motion global override (§9.5, Brand Guide) |

---

*— End of specification —*
