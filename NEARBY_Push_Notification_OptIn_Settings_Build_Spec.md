# NEARBY — Push Notification Opt-In / Settings Build Specification
### Everything needed to build both the contextual soft-ask prompt and the permanent Settings screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1, 4.6, endpoint table §5, FR-6.3, UC-19), NEARBY Brand/PWA Guide v1.0, and web-push permission-UX research (web.dev's Permission UX guidance, the "soft ask"/pre-permission pattern, Chrome's permanent-block-after-repeated-denial behavior)

---

## 0. Scope — two connected surfaces, and one real gap in the source document

This document covers **two things that have to work together**, not one screen:

1. **The contextual soft-ask** — a custom, in-app prompt shown *before* the native browser permission dialog ever fires, at the one moment your own doc makes this feature's value obvious: right after a customer's first broadcast goes live.
2. **The permanent Settings screen** — reachable from Profile (page-architecture item 1.17), where a customer can see their current status and manage it afterward.

The reason both are required, not just one: permission-UX research is unanimous that **a native browser denial is sticky and effectively permanent** — most browsers won't let a site re-trigger `Notification.requestPermission()` after a decline, and Chrome specifically suppresses the prompt entirely after repeated dismissals. So the soft-ask exists to protect that one native prompt from being burned on a bad first impression, and the Settings screen exists because, after that one shot, "management" can only mean "toggle within what's still possible" plus "explain how to fix it in browser/OS settings if it's blocked" — not a repeatable in-app control.

**The one real gap, named plainly (the third in this build series, after the feedback-trigger and request-history gaps):** your endpoint table has `POST /push/subscribe` (upsert-on-endpoint, §4.6) but **no corresponding unsubscribe/deactivate endpoint.** A customer who enables notifications and later wants to turn them off in-app has no documented server-side action to call. This document proposes the minimal, pattern-consistent fix:

> **Proposed: `DELETE /api/v1/push/subscribe`** — Bearer (customer), body `{ endpoint }` (the same `PushSubscription.endpoint` value used to upsert it), removes or marks-inactive the matching `pushSubscriptions` document. A direct structural counterpart to the existing `POST`, adding no new collection or field — just the missing other half of a create/remove pair.

---

## 1. Page goals, in priority order

1. Get the soft-ask shown at the right moment, worded honestly, and never on page load or at signup — per the research, an out-of-context prompt is the single biggest driver of reflexive denial.
2. Protect the native browser prompt — only trigger `Notification.requestPermission()` after the soft-ask's own "Enable" tap, never automatically.
3. Give the customer an honest, always-available way to check and adjust their notification status afterward, including the one case (browser-level block) the product genuinely cannot fix programmatically.
4. Keep the copy scoped to what NEARBY actually sends — this product has exactly one notification stream (transactional: "shops responded" / "window closed"), never marketing — and the copy throughout should say so plainly, since research flags stream-scoping as a real driver of trust.

---

## 2. Layout overview — the soft-ask (contextual, appears inline)

Not a full screen — a compact card, since the research is explicit this shouldn't feel like an interruption blocking the customer's actual task (which, at this moment, is watching their live broadcast).

```
┌─────────────────────────────┐
│ [Soft-ask card]                │
│  icon + headline + body        │
│  [Enable notifications] [Not now] │
└─────────────────────────────┘
```

## 2b. Layout overview — the Settings screen (permanent, in Profile)

```
┌─────────────────────────────┐
│ [A] Header (back + title)      │
├─────────────────────────────┤
│ [B] Current status card        │
├─────────────────────────────┤
│ [C] What you'll receive        │
├─────────────────────────────┤
│ [D] Action row (variant-specific) │
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### Soft-ask card (contextual trigger)
**Where it appears:** as a dismissible card at the top of the **Live-Count screen** (Section [B]/[C] area of that spec, above the broadcast-pulse centerpiece), shown the **first time** a customer reaches that screen with `Notification.permission === 'default'` (i.e., never asked, never blocked) — this is the exact intent-tied moment the research recommends: the customer has just started a real, live 2-minute window and the value of "we'll tell you when it resolves, even if you leave" is self-evident without any persuasion needed.
**Never shown:** on signup, on first Home-page load, or at any point before the customer has actually created a broadcast — per the explicit "never on page load, no context" anti-pattern from the research reviewed.
**Layout:** card, Surface white background, 1px Border-gray outline, rounded 12px, small bell-outline icon (20×20, Primary orange) + headline + body + two actions, dismissible via a small X in the corner (treated identically to tapping "Not now").
**Copy:**
- Headline: "Never miss a response" (Body, 600 SemiBold, Text-primary)
- Body: "We'll let you know the moment shops answer or your search window closes — even if you've closed the app. That's the only thing we'll ever notify you about." (Caption, Text-secondary) — the last sentence directly implements Goal 4, scoping expectations honestly up front.
**Actions:** "Enable notifications" (filled Primary orange, small, ~36px height) and "Not now" (text-only, Text-secondary) side by side.
**Behavior:**
- **"Enable notifications" tapped** → calls `Notification.requestPermission()` for the first and only time this session (this is the one and only trigger for the native dialog anywhere in the product, per Goal 2). If granted: registers a `PushSubscription` via the service worker + the app's VAPID public key, then `POST /api/v1/push/subscribe` with `{ endpoint, keys: { p256dh, auth } }` (§4.6) — card transitions to a brief inline confirmation ("Notifications on" with a small checkmark) before dismissing itself after ~1.5s. If the native dialog is denied: the card dismisses immediately with no error state (a denial isn't a failure to handle, it's a valid choice) and `Notification.permission` is now `'denied'` — the soft-ask must never appear again this session or in future sessions once this state is detected (checked on every Live-Count screen mount), since re-showing a soft-ask for a permission that's already permanently blocked would be pointless and mildly annoying.
- **"Not now" or X tapped** → dismisses the card for this broadcast only; the soft-ask is eligible to appear again on the customer's *next* broadcast (since no native permission was consumed by declining the soft-ask itself — this is the entire point of the two-step pattern, and it's explicitly allowed to ask again here, unlike after an actual native denial).
**Color:** icon Primary orange; headline Text-primary; body Text-secondary; "Enable" button filled Primary orange; "Not now" Text-secondary.
**Animation:** card slides down + fades in (`duration-standard`, 240ms, `ease-out-soft`) when it first appears on the Live-Count screen (a slight delay, ~500ms after the screen itself has settled, so it doesn't compete with the broadcast-pulse centerpiece's own entrance); dismisses via the reverse (slide up + fade, `duration-quick`, 180ms) whether dismissed manually or auto-dismissing after a successful opt-in.

### [A] Settings screen header
**Layout:** 56px row, back-chevron left (returns to Profile), centered title.
**Copy:** "Notifications"
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Current status card
**Purpose:** the one thing this screen most needs to communicate plainly — what state is the customer actually in.
**Layout:** card, Surface white background, 1px Border-gray outline, rounded 12px, icon + status line + one-line explanation.
**Three states, read directly from `Notification.permission`:**
1. **`granted` (and a subscription exists server-side):** bell-filled icon, Success teal — "Notifications are on" (Body, 600 SemiBold, Text-primary) + "We'll notify you when shops respond or your search window closes." (Caption, Text-secondary).
2. **`default` (never asked):** bell-outline icon, Text-secondary — "Notifications are off" (Body, 600 SemiBold, Text-primary) + "Turn them on so you don't have to keep the app open while you wait." (Caption, Text-secondary).
3. **`denied` (blocked at the browser level):** bell-slash icon, Text-secondary (not Danger — a blocked permission is a customer choice already made, not an error state, consistent with the "never frame a valid choice as a problem" principle applied throughout this series) — "Notifications are blocked" (Body, 600 SemiBold, Text-primary) + "You'll need to enable them in your browser settings — see below." (Caption, Text-secondary).
**Animation:** none — static status display, re-evaluated fresh on every screen mount (permission state can change outside the app, e.g. via browser settings, so this should never rely on stale cached state).

### [C] What you'll receive
**Purpose:** the same honest scoping promise from the soft-ask card, restated here for a customer who never saw that moment (e.g. they enabled it from Settings directly, or want a reminder of what they signed up for).
**Layout:** simple list, two rows, each a small icon + one line — not a card, just plain content, since there's nothing interactive about it.
**Copy:**
- 🔔 "When shops respond to your search"
- ⏰ "When your search window closes"
**Closing line below the list:** "That's it — no marketing, no promotions." (Caption, Text-secondary) — directly and plainly addressing the single biggest source of push-notification distrust identified in the research (unbounded, unpredictable notification volume).
**Animation:** none.

### [D] Action row — variant-specific by current status

**If `granted`:** a single "Turn off notifications" button, outline style (White background, 1px Border-gray outline, Text-primary text — deliberately neutral-toned rather than Danger red, since disabling notifications is a normal preference change, not a destructive/alarming action). Tapping it calls the proposed `DELETE /api/v1/push/subscribe` (Section 0) and updates the status card to reflect `default` — note that this **cannot** revoke the browser-level permission itself (only the app's own web standard doesn't allow that), so a customer who re-enables later will see the soft-ask/settings "Enable" flow work instantly without a fresh native prompt, since `Notification.permission` is still `granted` at the browser level; this in-app toggle only controls whether NEARBY actually holds and uses a subscription, not the underlying OS-level permission.

**If `default`:** a single "Enable notifications" button, filled Primary orange — identical behavior to the soft-ask card's own button (same `requestPermission()` → subscribe flow), just reachable proactively from Settings rather than waiting for the contextual moment.

**If `denied`:** **no button at all** — instead, a short numbered set of plain-language instructions for re-enabling via the browser (e.g. "Tap the lock icon in your address bar → Site settings → Notifications → Allow" as a generic pattern, ideally detected/adapted per browser if feasible, or presented as a general "look for site permissions in your browser's settings" instruction if per-browser detection isn't worth the engineering effort for a pilot) — this is the second deliberate "no action button" case in this entire build series (after the Cancelled screen's Variant B), for the same honest reason: the product genuinely cannot fix this from inside itself, and offering a button that does nothing would be worse than admitting the limit plainly.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where |
|---|---|---|
| Primary | `#FF5A36` | Soft-ask "Enable" button, Settings "Enable notifications" button |
| Success | `#00B8A9` | Status card's bell-filled icon (`granted` state) |
| Text primary | `#14213D` | Headlines, status-card main line, "Turn off" button text |
| Text secondary | `#5B6472` | Body copy throughout, `default`/`denied` icon tone, "What you'll receive" list text |
| Border | `#D8DCE3` | Card outlines, "Turn off" button outline |
| Surface | `#F4F5F7` / `#FFFFFF` | Page background (Gray 100) / cards, header (White) |

**No Danger red anywhere in this entire spec** — a blocked or disabled notification state is never treated as an error, consistent with the same principle already applied to the Feedback screen's "Not found" option and the Cancelled screen's Variant B.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Soft-ask headline | Body / 15px | 600 SemiBold |
| Soft-ask body | Caption / 13px | 400 Regular |
| Soft-ask button labels | Button / 15px (scaled down for the compact card) | 600 SemiBold |
| Settings header title | H1 / 24px | 600 SemiBold |
| Status-card main line | Body / 15px | 600 SemiBold |
| Status-card explanation | Caption / 13px | 400 Regular |
| "What you'll receive" list | Body / 15px | 400 Regular |
| Closing scoping line | Caption / 13px | 400 Regular |
| Action-row button labels | Button / 15px | 600 SemiBold |
| Browser-instructions text (`denied` state) | Body / 15px | 400 Regular |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Button tap-scale feedback |
| `duration-quick` | 180ms | Soft-ask card dismiss (manual or auto-after-success), inline confirmation swap |
| `duration-standard` | 240ms | Soft-ask card entrance on the Live-Count screen |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Soft-ask entrance/exit |

**No looping animation anywhere in this spec** — both surfaces are settled, informational/decision UI, not live states.
**Reduced motion:** the soft-ask's slide transitions degrade to a simple opacity fade, covered by the app-wide query.

---

## 7. Images, icons & video — asset list

**No video, no photography.**

| Asset | Format/size | Source |
|---|---|---|
| Bell-outline icon | Line icon, 20×20 (soft-ask) / 24×24 (status card) | Icon library |
| Bell-filled icon (`granted` state) | Line icon, 24×24, Success teal | Icon library |
| Bell-slash icon (`denied` state) | Line icon, 24×24, Text-secondary | Icon library |
| Small checkmark (soft-ask success confirmation) | Line icon, 16×16 | Icon library |
| 🔔 / ⏰ (What you'll receive list) | Emoji, consistent with every other small-list icon decision in this series | — |
| Back-chevron icon | Line icon, 24×24 | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| Soft-ask "Enable" | `Notification.requestPermission()` (browser API) → on grant, service-worker `pushManager.subscribe()` with the app's VAPID public key → `POST /api/v1/push/subscribe` with `{ endpoint, keys }` (§4.6) |
| Settings [B] status | Read directly from `Notification.permission` (`'granted'` / `'default'` / `'denied'`) client-side; cross-checked against whether an active subscription still exists server-side if the team wants full accuracy (e.g. a subscription could exist server-side while the browser permission has since been revoked externally) — for v1, trusting `Notification.permission` as the source of truth is a reasonable simplification |
| Settings [D] "Enable" | Identical flow to the soft-ask's own button |
| Settings [D] "Turn off" | Proposed `DELETE /api/v1/push/subscribe` (Section 0) with `{ endpoint }` |

**Loading state:** the brief moment between tapping "Enable" and the native dialog resolving needs no special loading UI — this is a fast, synchronous-feeling browser interaction; the `POST /push/subscribe` call itself can show a small inline spinner on the button if the team wants extra polish, but it's a fast call and likely unnecessary.
**Error state:** a failed `POST`/`DELETE` call (network issue, not a permission issue) uses the shared toast/error-boundary pattern (§6.1) — but a native permission *denial* is never treated as an error, per Section 3's explicit handling.

---

## 9. Responsive & platform notes

- **iOS Safari PWA note:** web push support in installed PWAs varies by iOS version — the soft-ask and Settings screen should both check `'serviceWorker' in navigator && 'PushManager' in window` before rendering any of this at all, and simply omit the feature entirely (no soft-ask card, no Settings entry) on a platform where it isn't supported, rather than showing a broken or permanently-disabled-looking control.
- **Viewport:** the soft-ask card fits within the Live-Count screen's existing layout without pushing the broadcast-pulse centerpiece off-screen on a standard reference viewport; the Settings screen follows the same centered, max-480px layout as every other screen in this series.
- **PWA standalone mode:** the Settings screen carries the app's normal back-navigation (returns to Profile) but no bottom tab bar, consistent with how sub-pages within Profile are treated elsewhere in this product's structure.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 4.6, FR-6.3, UC-19) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). The soft-ask/native-prompt separation and the "never re-ask after a real denial" rule are not stylistic choices — they follow directly from how browser permission APIs actually behave, and building around that behavior incorrectly would risk permanently losing the ability to ever ask a customer again. The missing unsubscribe endpoint (Section 0) is flagged the same way the Request History screen's missing list endpoint was — a gap to raise with the team, not something to quietly work around.*
