# NEARBY — Expired / No-Response Screen Build Specification
### Everything needed to build both copy variants of the `expired`-state screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1.3, 6.3), NEARBY Brand/PWA Guide v1.0, and empty-state UX research (Mobbin, UXPin, Eleken, SubUX — "never a dead end," positive/non-blaming language, one clear next action)

---

## 0. Scope — the two variants this one screen has to render, and why they can't share copy

This screen is what the Live-Count screen hands off to on `request:expired` — but your own v2.6.2 fix (§6.3) is explicit that this is **not one screen with one message.** The event payload carries `yesCount`, and the two cases it distinguishes are genuinely different situations that call for different advice:

| Case | `yesCount` | What actually happened | What the fix corrected |
|---|---|---|---|
| **Zero-response expiry** | `0` | No shop answered within the response window | Original copy: "no shops responded, re-broadcast at a larger radius" — still correct for this case |
| **Selection-timeout expiry** | `≥ 1` | Shops *did* answer, the customer saw a shortlist, but never selected within `DEFAULT_SELECTION_TIMEOUT_HOURS` (§3.1.3) | The bug your v2.6.2 fix closed: this case was showing the *same* "no shops responded, widen your radius" copy, which is actively wrong — radius was never the problem, the customer just didn't act in time |

Every empty-state UX principle in the research reviewed converges on the same rule: **never say "no results" without saying why, and never leave the customer without exactly one clear next action.** A generic single "Expired" screen would violate that twice over — it would misdiagnose the second case, and "expired" alone tells the customer nothing about what to do next. So this document specs **two distinct card configurations**, not two different screens, sharing one layout shell.

---

## 1. Page goals, in priority order

1. Tell the truth about what happened, specifically enough that the customer trusts the advice that follows — "we couldn't find anything" reads very differently depending on whether shops answered or not, and research on empty-state copy is explicit about never blaming the user ("No results found" → "We couldn't find anything — try adjusting filters").
2. Give exactly one clear, correct next action per case — a widened-radius re-broadcast for the zero-response case, a plain re-search for the timeout case — never both options at once, since offering the wrong one (radius, when radius wasn't the issue) is the exact mistake being corrected here.
3. Keep the tone matter-of-fact, not apologetic and not alarming — this is a routine, expected outcome of how the broadcast model works (not every search finds a shop, not every customer decides in time), not a failure state.
4. Never present this as a dead end — both variants end in a clear path back into the product (Section 3, both variants share a re-search action).

---

## 2. Layout overview

Single screen, shared shell across both variants, centered composition (same general "single-focus status screen" pattern as the Live-Count and Selection-Confirmation screens before it).

```
┌─────────────────────────────┐
│ [A] Header (minimal)          │
├─────────────────────────────┤
│ [B] Icon + headline (variant-specific) │
├─────────────────────────────┤
│ [C] Explanation copy (variant-specific) │
├─────────────────────────────┤
│ [D] Original request recap    │
├─────────────────────────────┤
│ [E] Primary action button      │  ← variant-specific label/behavior
├─────────────────────────────┤
│ [F] Secondary action link      │
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, back-chevron left (returns to Home), no title text (headline in [B] carries that role).
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Icon + headline — variant-specific
**Layout:** centered composition, upper third of the screen — an icon above a headline, mirroring the Selection-Confirmation screen's badge-and-headline structure, but in the calm/neutral register appropriate to this outcome rather than that screen's success register.

**Variant A — Zero-response (`yesCount: 0`):**
- **Icon:** a simple outline radar/radius glyph (a circle with a small dot at center and a few faint concentric rings — literally a still, non-pulsing version of the broadcast-pulse motif, communicating "we broadcast, nothing came back" through the *absence* of the pulse animation the customer saw earlier on the Live-Count screen for this exact request), 64×64, Text-secondary tone (not Danger — nothing went wrong, there's simply nothing to show).
- **Headline:** "No shops responded" (H1, Text-primary, 600 SemiBold).

**Variant B — Selection-timeout (`yesCount ≥ 1`):**
- **Icon:** a simple outline clock/hourglass glyph, 64×64, Text-secondary tone — communicates "time" rather than "nothing found," since something *was* found here.
- **Headline:** "Your search window closed" (H1, Text-primary, 600 SemiBold) — deliberately avoids the word "expired" in the customer-facing headline (that's the internal status name, not customer language) and deliberately avoids implying failure, since shops did respond.

**Animation (both variants):** icon fades in (`duration-standard`, 240ms, `ease-out-soft`) on screen mount — a single, calm entrance, no looping, no celebratory motion (this isn't a success state, but it also isn't an error — the animation register sits between the Selection-Confirmation screen's one-time checkmark and total stillness, appropriately muted for a neutral outcome).

### [C] Explanation copy — variant-specific
**Purpose:** the "why" that empty-state research insists must accompany any "nothing to show" message — this is the section that most directly implements your v2.6.2 fix.

**Variant A copy:** "No open shops in your search radius answered within the response window. This can happen if there just aren't many shops nearby right now, or if your radius was set narrow." (Caption/Body size, Text-secondary) — this is the one variant where radius is a legitimate, honest thing to mention, since it may genuinely have been a contributing factor.

**Variant B copy:** "**[N] shops** responded to your search, but the selection window closed before you picked one. Your radius wasn't the issue — feel free to search again whenever you're ready." (Caption/Body size, Text-secondary, `N` filled from the `yesCount` payload value) — explicitly reassures the customer that radius is a non-issue here, directly implementing the correction your fix made, and frames the missed window as a normal, no-fault thing ("whenever you're ready," not "you missed it").

**Tone note applying to both:** neither variant uses the word "unfortunately," "sorry," or any apologetic framing — per the empty-state research's "helpful, not apologetic" guidance — since neither outcome represents the product failing; both are ordinary, expected results of how a broadcast with a real time window works.

### [D] Original request recap
**Purpose:** identical in function and layout to the Live-Count screen's recap card (Section [E] of that spec) — reminds the customer exactly what they searched for, useful context whether they're about to re-broadcast the same thing or search for something different.
**Layout:** card, Surface white background, 1px Border-gray outline, rounded 12px — same three-row icon+label structure (🔍 product query, 📍 radius, 📌 location) as the Live-Count screen's recap card, reused directly rather than re-specified.
**Animation:** none — static reference card, same as its earlier counterpart.

### [E] Primary action button — variant-specific behavior
**Layout:** full-width minus 16px margins, 48px height, 12px radius, filled Primary orange — same visual treatment in both variants; **only the copy and destination differ.**

**Variant A — "Search a wider area":** tapping this pre-fills a *new* New Broadcast screen with the same `productQuery` and `location` already carried over, but with the radius selector defaulted to the **next quick-pick tier up** from what was originally used (e.g. if the original was 1km, this pre-selects 2km) rather than an arbitrary jump — a small, considerate detail that directly acts on the explanation given in [C] rather than just linking back to a blank form. The customer still confirms/can adjust before re-broadcasting (never auto-submits silently, consistent with every other pre-fill behavior already established in this series, e.g. the Home page's Repeat Request carousel).

**Variant B — "Search again":** tapping this opens a *new* New Broadcast screen pre-filled with the same `productQuery` and `location`, but radius **unchanged** from the original (since, per [C], radius was never implicated) — this is a deliberately plainer action than Variant A's, matching §6.3's explicit instruction that this case gets "a plain re-search action and no radius suggestion."

**Animation (both):** standard tap-scale-to-0.97; screen transitions to New Broadcast via the standard cross-fade (`duration-standard`, `ease-out-soft`), consistent with every screen-to-screen handoff in this series.

### [F] Secondary action link
**Purpose:** the second, lower-priority path back into the product — per empty-state research's guidance to keep recovery actions to a small number and clearly prioritized (never more than one primary + a light secondary), this is a plain text link, not a second button competing with [E].
**Copy:** "Back to Home" — Caption size, Text-secondary (not Primary orange, since this is deliberately the non-default choice), centered below [E].
**Behavior:** returns to Home without re-broadcasting anything — for a customer who's decided not to search again right now.
**Animation:** none beyond standard tap feedback.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | Primary action button ("Search a wider area" / "Search again"), recap-card icons (if colored, consistent with the Live-Count screen's icon treatment) |
| Text primary | `#14213D` | Headline, recap-card labels |
| Text secondary | `#5B6472` | Icon tone (both variants), explanation copy, secondary link text |
| Border | `#D8DCE3` | Recap card outline |
| Surface | `#F4F5F7` / `#FFFFFF` | (Gray 100 not prominently used on this screen — kept deliberately plain/white throughout, since there's no banner or strip content here needing a tint) / header, recap card, page background (White) |

**No Danger red anywhere on this screen, in either variant** — this is the single most important color decision here: neither "no shops responded" nor "you missed the window" is an error, and coloring either headline/icon in Danger red would misrepresent a normal, expected outcome as something having gone wrong, directly contradicting the tone principle established in Section 1.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Headline (both variants) | H1 / 24px | 600 SemiBold |
| Explanation copy | Body / 15px | 400 Regular |
| Recap-card labels | Body / 15px | 400 Regular |
| Primary action button label | Button / 15px | 600 SemiBold |
| Secondary link ("Back to Home") | Caption / 13px | 400 Regular |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Tap-scale feedback on both buttons/links |
| `duration-standard` | 240ms | Icon fade-in on mount, screen-exit to New Broadcast (on primary action) or Home (on secondary link) |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Icon entrance, screen-exit transitions |

**No looping animation of any kind on this screen** — consistent with the Selection-Confirmation screen's "settled state" register, but even more muted, since there's no celebratory moment to mark here either; this screen's single fade-in is the entirety of its motion budget.
**Reduced motion:** the icon simply appears in its final state with no fade — covered by the app-wide `prefers-reduced-motion` query, same as every other screen in this series.

---

## 7. Images, icons & video — asset list

**No video, no photography.**

| Asset | Format/size | Source |
|---|---|---|
| Radar/radius glyph (Variant A icon) | Outline icon, 64×64, Text-secondary — a still (non-animated, non-pulsing) rendering related to but visually distinct from the broadcast-pulse motif | Icon library, or a simplified static derivative of `nearby-mark.svg`'s arc concept — team's choice, either is consistent with the brand |
| Clock/hourglass glyph (Variant B icon) | Outline icon, 64×64, Text-secondary | Icon library |
| Recap-card icons (🔍 📍 📌) | Emoji, consistent with every earlier recap card in this series | Reused, no new asset |
| Back-chevron icon | Line icon, 24×24 | Icon library |

---

## 8. Data & state requirements

| Section | Data / logic |
|---|---|
| [B]/[C] variant selection | Driven entirely by the `request:expired` event's `yesCount` field (or the equivalent field on a `GET /api/v1/requests/:id` response if reached via REST fallback/poll rather than a live socket event) — `yesCount === 0` renders Variant A, `yesCount >= 1` renders Variant B; no other branching logic needed, this one field is sufficient per your own schema (§3.1.1/§4.3–4.4, `shopResponses` already tracks this) |
| [D] Recap card | Same `productQuery`, `radiusMeters`, `formattedAddress` fields already used by the Live-Count screen's identical card — carried over from the request document, no new fetch |
| [E] Primary action | Client-side pre-fill of a new New Broadcast screen instance with `productQuery` + `location` (+ radius logic per variant, Section 3[E] above) — no server call from this screen itself; the actual `POST /api/v1/requests` call happens on the New Broadcast screen's own submit, same as every other re-search entry point in this product (Home's Repeat Request carousel, etc.) |
| [F] Secondary link | Client-side navigation only |

**Loading state:** none needed — this screen only renders once the `expired` status and its `yesCount` are already known (delivered directly in the triggering event's payload), so there's no intermediate fetch this screen has to wait on.
**Error state:** not applicable — same reasoning as the Selection-Confirmation screen; this is purely a display/informational screen with two possible content configurations, not a screen that itself makes a network call that could fail.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with every screen in this series; comfortably fits without scroll on a standard reference viewport in both variants.
- **Accessibility:** per the empty-state accessibility guidance reviewed, this screen's arrival should be announced to screen-reader users via an `aria-live="polite"` region on the headline/explanation block — since the customer may have this screen appear while their attention is elsewhere (backgrounded app, arriving via a push notification), not just while actively watching the Live-Count screen countdown.
- **Push-notification tie-in:** per the Master Build Doc's push-subscription fix (dispatch on `awaiting_selection`/`expired`), a customer who taps a push notification for this outcome should land directly on the correct variant of this screen (the notification payload or an immediate `GET /api/v1/requests/:id` fetch should already carry `yesCount`) — never on a generic, un-branched version of this screen while the variant is still being determined.
- **PWA standalone mode:** no bottom tab bar (consistent with every screen in the broadcast→outcome journey) — the header's back-chevron and the [F] secondary link both return to Home, where the tab bar resumes.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1.3, 6.3) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). The two-variant structure specced here is not a design embellishment — it is the direct UI implementation of the exact bug your own v2.6.2 audit caught and fixed: a single, undifferentiated "expired" screen would silently reintroduce the misleading "widen your radius" advice for a customer who simply ran out of time to select from a shortlist that already existed.*
