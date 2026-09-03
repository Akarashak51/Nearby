# NEARBY — Pending-Approval Screen Build Specification
### Everything needed to build the `status: 'pending'` shop-dashboard screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.3, 4.2, 5.1, 6.1, endpoint table §5), NEARBY Brand/PWA Guide v1.0 (§1.4, 3.3), and waiting-room/pending-state UX research (the same empty-state honesty principles applied throughout this series, extended to a genuinely indeterminate wait)

---

## 0. Scope — what this screen is the *only* screen for, and the one real-time gap worth naming

Once a shopkeeper submits the onboarding form (previous spec), `GET /api/v1/shops/me` (§ endpoint table) returns `status: 'pending'` — and per §6.1, that response is specifically designed so the dashboard can distinguish "no profile yet," "pending," "approved," "rejected," and "blocked" without guessing. This screen is what renders for the `pending` case.

**A structural fact worth stating plainly, since it shapes this screen's whole posture:** a user whose `role` was just promoted to `'shop'` (§3.3's v2.6.1 fix) no longer has `role: 'customer'`, and every customer-facing endpoint in this product (`POST /requests`, `GET /requests/:id`, etc.) requires `Bearer (customer)`. **This screen is very likely the only thing this account can see in the entire product until an admin acts on it** — there's no customer app to fall back into, and no other shop-dashboard screen exists yet since there's no approved shop to run a dashboard for. This is functionally a waiting room, not one screen among several equally-available options, and the design below treats it that way.

**The gap worth naming (the fifth in this build series):** there is no documented Socket.IO event for a shop-status change (approve/reject/block) anywhere in Section 5's "Real-time events" table — only request-lifecycle events are listed, even though a `shop:<shopId>` room already exists and is already joined by every shop socket (§5, room-membership rules). This document proposes the smallest possible addition:

> **Proposed: `shop:statusChanged`** — server → the shop's own `shop:<shopId>` room, fired from the same admin-action handlers that already exist for `PATCH /admin/shops/:id/approve`/`reject`/`block`. No new room, no new infrastructure — just one more event emitted from code that already runs.

**Why this gap matters less in practice than it might on paper:** §3.3's approval criteria include a **manual admin phone callback** to confirm the shop's phone number is reachable — which means, for most shops in this pilot, a real human phone call is very likely to be the actual moment a shopkeeper learns their status is about to change, well before any in-app mechanism would. This document specs a sensible polling fallback (Section 8) regardless, since relying entirely on an informal phone call as the *only* status-update mechanism would be fragile, but the phone call meaningfully lowers the stakes of this particular gap compared to, say, the customer-side push-notification gaps surfaced earlier in this series.

---

## 1. Page goals, in priority order

1. Confirm, plainly, that the submission went through and is genuinely being looked at — the single biggest anxiety in any "please wait" state is wondering if anything actually happened.
2. Set an honest expectation for what happens next, specifically naming the phone callback (§3.3 criterion a) so a shopkeeper isn't confused when their phone rings from an unfamiliar number asking about their shop.
3. Let the shopkeeper see exactly what they submitted, without needing to remember it themselves — useful if the wait is long enough that details fade from memory.
4. Give a low-effort way to check for an update without needing to fully exit and relaunch the app.

---

## 2. Layout overview

Single screen, centered composition — the whole-screen "status display" pattern already used for the customer-side Live-Count and outcome screens, applied here for the shop side's equivalent waiting state.

```
┌─────────────────────────────┐
│ [A] Header (title only)       │
├─────────────────────────────┤
│ [B] Status icon + headline     │
├─────────────────────────────┤
│ [C] What happens next          │
├─────────────────────────────┤
│ [D] Submitted details recap     │
├─────────────────────────────┤
│ [E] Check status button          │
├─────────────────────────────┤
│ [F] Sign out                      │
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, no back-chevron (per Section 0, there's genuinely nowhere else in the product for this account to go), centered title.
**Copy:** "Application status"
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Status icon + headline
**Layout:** centered composition, upper third of the screen.
**Icon:** a simple outline clock/hourglass glyph, 64×64, **Signal Teal** tone (`#00B8A9` — the shop surface's dominant accent, per Section 0's Teal-forward rule established in the Onboarding screen spec; this is the first "outcome-style" icon in the series to use a colored rather than neutral Text-secondary tone, a deliberate choice since Teal here reads as "in progress, on this surface" rather than needing the neutral treatment used for the customer-side Expired/Cancelled screens' icons — this is not a negative or ambiguous outcome, it's an active, expected step).
**Headline:** "Your shop is under review" (H1, Text-primary, 600 SemiBold).
**Animation:** icon fades in (`duration-standard`, 240ms, `ease-out-soft`) on mount — same restrained, one-time entrance as every other outcome-style icon in this series.

### [C] What happens next
**Purpose:** directly implements Goal 2 — the single most important explanatory content on this screen.
**Copy:** "We'll check your details and give you a call at **[phone number just submitted]** to confirm everything's correct. This usually takes a day or two." — naming the actual phone number back to the shopkeeper (pulled from their own submission) makes this concrete rather than abstract, and "a day or two" is an honest, deliberately soft timeframe rather than a specific promise the pilot's manual process can't reliably guarantee.
**Layout:** card, Surface white background, 1px Border-gray outline, rounded 12px, phone-outline icon (16×16, Text-secondary) + the copy above, sits below the headline.
**Animation:** none — static, informational, same treatment as every "what happens next" card in this series.

### [D] Submitted details recap
**Purpose:** Goal 3 — lets the shopkeeper confirm what's actually under review without having to remember it.
**Layout:** card, same visual treatment as [C], simple label/value rows:
- Shop name
- Category
- Address (the confirmed `formattedAddress` from the onboarding screen's map-confirm step)
- Phone number
- A small thumbnail of the uploaded photo, or a filename + document icon if a document was submitted instead — mirroring exactly which of the two the shopkeeper chose on the Onboarding screen.
**Animation:** none — static reference card.

### [E] Check status button
**Purpose:** Goal 4 — a low-effort manual refresh, given the real-time gap named in Section 0.
**Layout:** full-width minus 16px margins, 44px height, outline style (White background, 1px Signal Teal outline, Signal Teal text — not filled, since this is a supporting action rather than this screen's single primary CTA; there isn't really a "primary" action on a waiting screen, so keeping this outline-weight avoids implying urgency that doesn't exist).
**Copy:** "Check status"
**Behavior:** re-fetches `GET /api/v1/shops/me` on tap. If still `pending`, a brief inline confirmation appears beneath the button — "Still under review" (Caption, Text-secondary) — fading out after ~2 seconds, so tapping this doesn't feel like it did nothing even when nothing has changed. If the fetch returns `approved`, `rejected`, or `blocked`, the screen transitions immediately to the corresponding destination (Section 4).
**Animation:** tap-scale-to-0.97; the "Still under review" confirmation fades in/out (`duration-quick`, 180ms) rather than appearing/disappearing abruptly.

### [F] Sign out
**Layout:** plain text-only row, Danger red, centered, sits at the natural bottom of the screen — identical treatment and reasoning to the Profile screen's own Sign Out row (a genuinely consequential, singular action on an otherwise calm screen, worth the one deliberate use of Danger red).
**Copy:** "Sign out"
**Behavior:** same confirmation-then-clear-JWT pattern already specced for the Profile screen — included here specifically because, per Section 0, this account has no other standing "place" to navigate to, so leaving the app entirely (rather than sitting on this screen) is a completely reasonable thing for a shopkeeper to want to do while waiting, and it should be easy to find.
**Animation:** identical to the Profile screen's Sign Out row.

---

## 4. Color usage summary (Brand Guide §3.1/3.3 — no new hex introduced)

| Token | Hex | Used where |
|---|---|---|
| **Signal Teal** | `#00B8A9` | Status icon, "Check status" button outline/text |
| Danger | `#E5484D` | Sign-out row (the one deliberate exception, per Section 3[F]'s reasoning) |
| Text primary | `#14213D` | Headline, recap-card labels/values |
| Text secondary | `#5B6472` | "What happens next" body copy, recap-card icon, "Still under review" confirmation text |
| Border | `#D8DCE3` | Card outlines |
| Surface | `#F4F5F7` / `#FFFFFF` | Page background (Gray 100) / cards, header, button background (White) |

**No Primary orange anywhere on this screen** — same Teal-forward rule established on the Onboarding screen, carried through consistently; this screen has no Yes/No response action, so it has no reason to use orange at all.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Status headline | H1 / 24px | 600 SemiBold |
| "What happens next" copy | Body / 15px | 400 Regular |
| Recap-card labels/values | Body / 15px | 400 Regular |
| "Check status" button label | Button / 15px | 600 SemiBold |
| "Still under review" confirmation | Caption / 13px | 400 Regular |
| "Sign out" label | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Tap-scale feedback |
| `duration-quick` | 180ms | "Still under review" confirmation fade in/out |
| `duration-standard` | 240ms | Status-icon fade-in on mount, screen-exit cross-fade on a status change |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Icon entrance, screen-exit |

**No looping animation anywhere on this screen** — this is a genuinely indeterminate wait, and an ambient looping animation (even a subtle one) risks either implying false progress or, over a wait that could run "a day or two," becoming visually tiresome if the shopkeeper leaves the screen open. A single static icon communicates "waiting" perfectly well without needing continuous motion to sustain that message.
**Reduced motion:** covered by the app-wide `prefers-reduced-motion` query.

---

## 7. Images, icons & video — asset list

**No video, no stock photography.** The recap card's thumbnail is whatever the shopkeeper uploaded themselves (Section 3[D]).

| Asset | Format/size | Source |
|---|---|---|
| Clock/hourglass icon | Outline icon, 64×64, Signal Teal | Icon library |
| Phone-outline icon (what-happens-next card) | Line icon, 16×16, Text-secondary | Icon library |
| Recap-card photo thumbnail / document icon | 40×40, rounded 8px | Reused from the shopkeeper's own onboarding upload — no new asset |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B]/[C]/[D] | `GET /api/v1/shops/me` (Bearer, shop role, §6.1) — returns `status`, `shopName`, `category`, `address`/`formattedAddress`, `phone`, `photoUrl`/`documentUrl` |
| [E] Check status | Same endpoint, re-fetched on demand |
| Passive refresh (recommended, given Section 0's gap) | A lightweight periodic re-fetch — e.g. on screen focus/app-resume (the PWA equivalent of "the shopkeeper reopened the app," which is a natural, low-cost moment to check) rather than a continuous poll, since a continuous timer running for potentially a day or two would be wasteful for very little benefit given the phone-callback context already discussed |
| Optional enhancement | The proposed `shop:statusChanged` Socket.IO event (Section 0) — if implemented, this screen can additionally listen for it and transition immediately without waiting for the next app-resume/manual check, closing the gap fully rather than partially |

**Loading state:** brief skeleton (gray placeholder blocks for the recap card's rows) while the initial `GET /shops/me` resolves on first mount; the "Check status" button shows an inline spinner during its own re-fetch.
**Error state:** a failed fetch uses the shared toast/error-boundary pattern (§6.1) with a retry option — critically, a failed fetch should **never** be mistaken for a rejected/blocked status; the screen should only transition away on an actual, successfully-returned status change, never on an error.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with every screen in this series.
- **No bottom tab bar** — per Section 0, there is nothing else for this account to navigate to yet; this screen effectively *is* the entire app experience for a shop in this state.
- **PWA standalone mode / re-launch behavior:** since a shopkeeper may close the PWA entirely and reopen it hours or a day later, this screen should always re-fetch fresh status on mount rather than trusting any cached/stale client state — the "passive refresh on focus/resume" behavior (Section 8) is what makes that reopening moment actually useful rather than just redisplaying old information.
- **Screen-exit destinations:** `approved` → the Live Dashboard/Home screen (page-architecture item 2.4, Teal-forward, next natural build in this series); `rejected` → the Rejected/Re-apply screen (item 2.3); `blocked` is not a realistically reachable transition from `pending` (blocking is a post-approval enforcement action per §4.2/§3.5) and doesn't need to be handled here.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.3, 4.2, 5.1, 6.1, and the master endpoint table in Section 5) and NEARBY Brand/PWA Guide v1.0 (Sections 1.4, 3.3). This screen carries forward the Teal-forward, orange-free color rule established on the Onboarding screen, and names its own real-time gap plainly (no documented shop-status-change socket event) while noting — honestly, not as an excuse to skip fixing it — that the pilot's manual phone-callback approval process already provides a real, human-driven notification path that meaningfully softens the impact of that gap.*
