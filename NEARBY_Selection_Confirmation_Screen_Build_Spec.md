# NEARBY — Selection Confirmation Screen Build Specification
### Everything needed to build the `selected`-state confirmation screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1, 3.1.2, 5.3), NEARBY Brand/PWA Guide v1.0, and success/confirmation-screen UX research (Baymard's order-confirmation benchmark, Material Design's confirmation/acknowledgement guidance, "tune celebration to the importance of the action")

---

## 0. Scope — what just happened, and the one tone rule that governs this whole screen

This is where `POST /api/v1/requests/:id/select` lands the customer — `requests.status` is now `selected`, `selectedShopId` is set, and `request:selected` has just fired to the chosen shop's room and every other Yes-respondent's room (their side gets "not selected this time," with an explicit guarantee it never touches their `reliabilityScore`, §3.1/§3.1.2). From the customer's side, this is the last screen in the whole broadcast→select journey — the next thing that happens is a physical visit, not another in-app step (until the Post-visit Feedback screen, later).

**The one rule every section below follows:** confirmation-UX research is consistent that celebration should be *tuned to the size of the action* — "save big, celebratory screens for significant milestones... for routine actions, keep confirmations subtle." Picking a shop to walk into is a real decision, but it's not a purchase, a milestone, or a first-time event the way, say, completing a signup might be. So this screen should read as **calm, clear, and useful** — a checkmark and the practical next-step info the customer actually needs (address, phone, directions) — not a confetti burst or a heavy "Congratulations!" treatment. That restraint is a deliberate design choice, not an underbuild.

---

## 1. Page goals, in priority order

1. Confirm, unambiguously, that the selection went through — per the near-universal UX convention, a checkmark + brief motion is the clearest possible signal of "this succeeded," and this screen leads with exactly that.
2. Immediately hand the customer everything they need to actually go — address, distance, phone, directions — since Baymard's own order-confirmation benchmark flags this exact moment as "the best place for auxiliary next-step actions," and here those actions are concrete and useful (call, navigate), not decorative.
3. Set a light expectation for what happens after the visit (the feedback prompt) without turning this screen into a form or a promise the customer has to remember.
4. Give a clean way back into the rest of the app — this screen is an endpoint, not a dead end.

---

## 2. Layout overview

Single screen, no scroll needed on standard viewports, centered composition (mirrors the vertical, single-focus layout already used for the Live-Count screen, since both are "here's the current state of my request" screens, just at opposite ends of the journey).

```
┌─────────────────────────────┐
│ [A] Header (minimal)          │
├─────────────────────────────┤
│ [B] Success check + headline  │
├─────────────────────────────┤
│ [C] Shop summary card          │
├─────────────────────────────┤
│ [D] Action row (Call / Directions) │
├─────────────────────────────┤
│ [E] What happens next strip   │
├─────────────────────────────┤
│ [F] Done button                 │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, **no back-chevron** — this screen deliberately has no "back" action, since backing out of a completed selection doesn't correspond to anything reversible in the system (the customer can't un-select; Section 0). No title text either — the headline in [B] does that job.
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Success check + headline — the centerpiece
**Purpose:** the unambiguous "this worked" signal, per the success-state research reviewed ("a green fill paired with a checkmark icon is the most universally understood signal for a positive outcome... keep the animation brief and the messaging clear").
**Layout:** centered composition, sits in the upper third of the screen.
**Visual:** a circular badge, 72×72, Success-teal (#00B8A9) fill, White checkmark icon centered inside — **not** the broadcast-pulse arcs motif reused here (that motif means "live, ongoing, real-time" everywhere else in this product; this moment is the opposite — something has *concluded*, and reusing the pulse would blur that meaning, which matters a lot in a product whose whole visual language depends on the pulse consistently signaling "in progress").
**Copy (headline, directly below the badge):** "You're all set" (H1, Text-primary, 600 SemiBold) — chosen over a more effusive "Congratulations!" per the tone rule in Section 0; this phrase also directly echoes the "You're all set!" example microcopy flagged in the confirmation-UX research as effective, understated success language.
**Sub-headline:** "You've selected **[Shop Name]**" (Body, Text-secondary, shop name bolded inline).
**Animation:** the checkmark draws in with a brief, one-time stroke-animation (the checkmark's path animates from 0% to 100% length over ~400ms, a standard "drawn checkmark" micro-interaction, `ease-out-soft`) the instant this screen appears, followed by the circular badge doing a single soft scale pulse (1.0 → 1.05 → 1.0, ~200ms, `ease-standard`) — brief, satisfying, and over within under a second, deliberately not looping (per "keep the animation brief" — this is not the Live-Count screen's continuous pulse; it fires once and settles into a static state).

### [C] Shop summary card
**Purpose:** the practical payload — everything the customer needs to actually go, presented as a compact reference card rather than requiring a tap-through to the full Shop Profile screen again (the customer already made their decision; this card is for execution, not re-deciding).
**Layout:** card, Surface white background, 1px Border-gray outline, rounded 12px, sits below the headline.
**Content:**
- Shop photo thumbnail (56×56, rounded 8px) or initials-avatar fallback — same treatment as the Shortlist card.
- Shop name (Body, 600 SemiBold, Text-primary).
- Address (Caption, Text-secondary, from the shop's `formattedAddress`).
- Distance (Caption, Text-secondary) — "0.4 km away," the same figure carried through from every earlier screen in this journey.
**Animation:** fades+slides in (translateY 8px→0, `duration-standard`, `ease-out-soft`) slightly after the checkmark's own animation completes (a ~100ms stagger) — the checkmark is the moment, this card is what follows it, and a small sequencing gap reinforces that order without needing an explicit "step 2" label.

### [D] Action row — Call / Get directions
**Purpose:** the two next-step actions Baymard's research explicitly calls out this screen-type as the right place for.
**Layout:** two equal-width buttons side by side, 44px height, 10px radius, 8px gap between them, sit directly below the shop summary card.
**Left button — "Call shop":** outline style (White background, 1px Primary-orange outline, Primary-orange text + phone icon) — a `tel:` deep link to the shop's contact number.
**Right button — "Get directions":** filled style (Primary-orange background, White text + direction-arrow icon) — the device's native maps deep link, identical construction to the Shop Profile screen's "Get directions" link (Section [E] of that spec) — **reused, not re-specified**, same reasoning as that document's own note about reusing the Select-confirmation modal.
**Color rationale for the asymmetry:** "Get directions" gets the filled/primary treatment because it's the more universally-needed next action (a customer visiting in person needs to get there; calling ahead is helpful but optional) — a small, deliberate visual hierarchy choice between two otherwise-equal-seeming actions.
**Animation:** standard tap-scale-to-0.97 on either button; no special entrance animation beyond being part of the same fade-in group as [C] (a stagger of the whole "informational" block together, rather than staggering every individual element separately, which would feel fussy for a screen this short).

### [E] What happens next strip
**Purpose:** sets a light, honest expectation for the Post-visit Feedback screen (page-architecture item 1.11) without asking the customer to do anything right now.
**Copy:** "Once you've visited, we'll ask how it went — it helps other customers and keeps [Shop Name]'s reliability score accurate."
**Layout:** thin full-width bar, Gray 100 background, small radiating-signal or checkmark-outline icon (reuses existing icon vocabulary, no new asset), Caption text, Text-secondary.
**Why this line matters here specifically:** it quietly reinforces the same honesty already built into the Shortlist/Shop-Profile screens' selection-confirmation modal (that reliability is based on real, rated visits) — the customer is told *why* the feedback ask exists, not just that it's coming, which research on success-state microcopy notes is a small but meaningful trust-builder when a product asks something of its users later.
**Animation:** none — static, informational, same treatment as every other "what happens next"-style strip across this screen series (Home, New Broadcast).

### [F] Done button
**Layout:** fixed to the bottom of the screen, full-width minus 16px margins, 48px height, 12px radius, sits above the safe-area inset.
**Copy:** "Done" — deliberately plain, not "Back to Home" or anything more elaborate; this is a closing action, and Material Design's own acknowledgement-pattern guidance is that an acknowledgement simply needs to let the customer move on, not offer more choices at this point.
**Behavior:** returns to the Home screen, where the Active Request banner (Home page spec, Section [C]) now reflects the `selected` state — "You're headed to **[Shop Name]**" — so the information from this screen remains accessible afterward rather than disappearing once "Done" is tapped.
**Color/states:** standard secondary-style button here (White background, 1px Border-gray outline, Text-primary text) rather than filled Primary — this button is genuinely the lowest-priority action on the screen (the two action-row buttons in [D] are what the customer actually came here to use), so it's visually the quietest element, sized the same as everything else but not fighting for attention.
**Animation:** tap-scale-to-0.97; screen exits via the standard cross-fade (`duration-standard`, `ease-out-soft`) back to Home.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Success | `#00B8A9` | Checkmark badge fill |
| Primary | `#FF5A36` | "Get directions" button fill, "Call shop" button outline/text, action-row icons |
| Text primary | `#14213D` | Headline, shop name, Done button text |
| Text secondary | `#5B6472` | Sub-headline, address/distance text, what's-next strip text |
| Border | `#D8DCE3` | Shop summary card outline, Done button outline |
| Surface | `#F4F5F7` / `#FFFFFF` | What's-next strip background (Gray 100) / header, cards, Done button background (White) |

**No Warning or Danger tokens used on this screen** — there's nothing cautionary or negative to communicate here; introducing either would work against the calm, settled tone this screen is built around.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Headline ("You're all set") | H1 / 24px | 600 SemiBold |
| Sub-headline (shop name) | Body / 15px | 400 Regular (shop name inline-bolded) |
| Shop summary card: name | Body / 15px | 600 SemiBold |
| Shop summary card: address/distance | Caption / 13px | 400 Regular |
| Action-row button labels | Button / 15px | 600 SemiBold |
| What's-next strip | Caption / 13px | 400 Regular |
| Done button label | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Tap-scale feedback on all buttons |
| `duration-standard` | 240ms | Shop summary card / action row fade-slide-in, screen-exit on Done |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Badge scale-pulse |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Checkmark stroke-draw, card/action-row entrance, screen exit |
| Checkmark stroke-draw | ~400ms, one-time | [B], on screen mount only |
| Badge scale-pulse | 1.0→1.05→1.0, ~200ms, one-time | [B], immediately following the stroke-draw |

**Everything on this screen animates exactly once, on entry, and then goes still** — this is the clearest possible contrast with the Live-Count screen's continuously-looping pulse, and that contrast is intentional: one screen represents an unresolved, ongoing process; this one represents a resolved, settled outcome. The animation vocabulary should make that difference legible even with the sound off and the copy unread.
**Reduced motion:** the checkmark and badge simply appear in their final state with no stroke-draw or scale-pulse; the card/action-row entrance becomes an instant, non-animated appearance — covered by the app-wide `prefers-reduced-motion` query.

---

## 7. Images, icons & video — asset list

**No video, no shop photography beyond the small summary-card thumbnail** (already an existing asset from onboarding, not something new to produce for this screen).

| Asset | Format/size | Source |
|---|---|---|
| Checkmark icon | Vector path, drawn/animated via CSS/SVG stroke-dasharray technique, White on Success-teal circle | Custom, simple geometric shape — no external library needed, a single checkmark SVG path is sufficient |
| Shop photo thumbnail (summary card) | 56×56, rounded 8px, or initials-avatar fallback | Reused from the shop's existing onboarding photo — no new asset |
| Phone icon (Call button) | Line icon, 16×16 | Icon library (same one used app-wide) |
| Direction-arrow icon (Get Directions button) | Line icon, 16×16 | Icon library |
| What's-next strip icon | Small checkmark-outline or radiating-signal glyph, 16×16 | Reuses existing icon vocabulary from earlier screens — no new asset |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B]/[C] | Shop name, photo, address, distance — carried over directly from the shortlist/select action's already-available client state (the customer just selected this exact shop from data already loaded); no new fetch needed in the common case |
| [D] Call | `tel:` deep link using the shop's phone number, client-side only |
| [D] Get directions | Native maps deep link using the shop's stored coordinates, client-side only — identical construction to the Shop Profile screen's equivalent link |
| [F] Done | No endpoint — client-side navigation back to Home, where the Active Request banner now reads the `selected` state from the request document already updated by the select call |

**Loading state:** none needed — this screen only ever renders after a successful `POST .../select` response, which already carries everything required to populate it; there's no intermediate "loading the confirmation" state to design for.
**Error state:** not applicable to this screen itself — a failed select attempt is handled entirely within the *previous* screen's confirmation modal (Shortlist/Shop-Profile specs), which keeps its own modal open with an inline retry rather than ever routing the customer to this screen on failure. This screen only exists on the success path.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with every screen in this series; comfortably fits without scroll on a standard 360×740px reference viewport.
- **No back-gesture override needed:** since there's no meaningful "back" state for this screen (Section 3[A]), the OS/gesture back-action can simply be allowed to behave as "Done" would — returning to Home — rather than needing custom interception; this keeps the implementation simple without creating a confusing dead-end if a customer swipes back out of habit.
- **PWA standalone mode:** this screen carries no bottom tab bar (consistent with every other screen in the broadcast→select journey, which are all treated as a single focused task rather than the app's normal navigable "places") — tapping Done is what returns the customer to the tab-bar-equipped Home screen.
- **Push-notification tie-in:** if the customer wasn't actively watching the app when the selection was confirmed via some future multi-device scenario (not applicable in v1's single-selection-flow model, but worth noting for consistency with the rest of this series), this screen's data requirements are fully self-contained from the select response and don't depend on any additional real-time listening once rendered.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 3.1.2, 5.3) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). The checkmark-and-settle animation vocabulary is deliberately distinct from the broadcast-pulse motif used on the Live-Count screen — one signals "in progress," the other signals "resolved," and the two are never interchanged anywhere in this product.*
