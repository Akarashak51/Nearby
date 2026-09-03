# NEARBY — Post-Visit Feedback Screen Build Specification
### Everything needed to build the `POST /requests/:id/feedback` screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1.2, 4.5, FR-4.1/4.2), NEARBY Brand/PWA Guide v1.0, and rating/feedback-UI pattern research (accessible star-rating patterns, Netflix's binary-over-scale simplification rationale, rating-popup timing/tone research)

---

## 0. Scope — one honest gap this document has to fill, and how it's filled

Your Master Build Document defines exactly what this screen submits — `outcome` (`found`/`not_found`, required), `rating` (1–5, required), `note` (optional free text) via `POST /requests/:id/feedback` (§4.5) — and exactly what that submission feeds: `foundCount`/`notFoundCount` and the reliability-score recalculation (§3.1.2). **What it does not define is when or how this screen is triggered.** There's no visit-detection mechanism (no GPS check-in, no QR scan), and no scheduled push job for this specific prompt is documented — the only push dispatch your doc names explicitly is on `awaiting_selection`/`expired` (§3.1's fixes list), not on a "time to rate your visit" trigger.

**This is a real gap, not something to silently invent an elaborate solution for.** The pragmatic, zero-new-infrastructure answer, consistent with everything else in this document's minimal-pilot-footprint philosophy: **a few hours after selection, the Home page's Active Request banner (Home page spec, Section [C]) quietly changes from "You're headed to [Shop Name]" to "How did your visit to [Shop Name] go?"** — same banner, same location, no push notification, no new job, no new schema field (the existing `requests.selectedAt`-adjacent timestamp is enough to compute "a few hours have passed" client-side or via a simple server-side check). Tapping that updated banner opens this screen. This is stated here as a **documented assumption**, not a literal spec requirement — flagged clearly so the team can confirm or adjust the exact timing threshold (a sensible default: 2 hours after selection, configurable the same way `DEFAULT_SELECTION_TIMEOUT_HOURS` already is, §3.11).

---

## 1. Page goals, in priority order

1. Capture the one fact that actually matters for reliability scoring — **did the shop actually have what the customer was looking for** (`outcome`) — as fast and unambiguously as possible; this is the input the reliability formula weights most heavily (`confirmedFoundRate` at 0.7 of the score, §3.1.2), so it deserves the most prominent, easiest-to-answer treatment on the screen.
2. Capture a star rating without over-complicating the scale — research on rating-scale design consistently favors simplicity (Netflix's own move from 5-star to thumbs-up/down specifically to reduce friction and increase participation) — but since your schema requires a 1–5 numeric `rating` specifically (§4.5, not a binary), this screen implements the classic 5-star pattern rather than reinterpreting the schema, while keeping every other part of the screen as low-friction as that research recommends.
3. Never force the customer through this screen — it's a banner-triggered, optional prompt, not a blocking step; a customer who ignores it loses nothing except the ability for that particular visit to count toward the shop's score (§3.1.2 already treats an un-rated selection as simply out of scope for the formula, no penalty to anyone).
4. Keep the optional note field genuinely optional and lightweight — this is a quick prompt, not a full review form.

---

## 2. Layout overview

Single screen, no bottom tab bar (a focused prompt, same modal-style treatment as New Broadcast), compact — this should feel like a 15-second task, not a form.

```
┌─────────────────────────────┐
│ [A] Header (close + title)    │
├─────────────────────────────┤
│ [B] Shop reminder row          │
├─────────────────────────────┤
│ [C] Outcome toggle (the primary ask) │
├─────────────────────────────┤
│ [D] Star rating                │
├─────────────────────────────┤
│ [E] Optional note field        │
├─────────────────────────────┤
│ [F] Submit button               │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, an **X (close) icon** on the left rather than a back-chevron — signals this screen can be dismissed at any time without penalty, distinct from the app's normal back-navigation semantics (this is a prompt the customer opted into by tapping a banner, not a "place" they navigated to). Centered title.
**Copy:** "Rate your visit"
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Shop reminder row
**Purpose:** simple context — which shop is this about (a customer may not open this the same day they visited, so a reminder earns its place here even on a short screen).
**Layout:** single row, shop photo thumbnail (40×40, rounded 8px, or initials-avatar fallback — same treatment as every other shop-thumbnail instance in this series) + shop name (Body, 600 SemiBold, Text-primary), sits directly below the header.
**Copy:** just the shop's name and photo — no extra descriptive text needed here, this row's only job is identification.
**Animation:** none.

### [C] Outcome toggle — the primary ask
**Purpose:** this is Goal 1 — the single most consequential input on the whole screen, and the one this layout gives the most visual weight to (more than the star rating below it, a deliberate hierarchy choice, since `confirmedFoundRate` carries more than double the weight of `speedScore` in the reliability formula, §3.1.2 — though the UI doesn't need to expose that weighting numerically, giving this question the top, larger position communicates the same priority implicitly).
**Copy (question label above the toggle):** "Did you find what you were looking for?"
**Layout:** two large, equal-width tappable segments side by side, 56px height (taller than a typical button — this is the screen's headline interaction), rounded 12px, each with an icon + one word.
- **Left segment — "Found"**: checkmark icon + "Found" label.
- **Right segment — "Not found"**: x/slash icon + "Not found" label.
**Selected-state color:** the selected segment fills with **Success teal** (#00B8A9) for "Found" or a **neutral Gray-700-equivalent dark fill** for "Not found" (deliberately **not** Danger red — "not found" is a normal, valid, honest answer a customer should feel entirely comfortable giving, and coloring it as an alarm would subtly discourage honest negative feedback, directly undermining the reliability score's whole purpose). Unselected segment stays White background with 1px Border-gray outline.
**Behavior:** single-select, no default pre-selected — the customer must make an active choice (this is the one field on the screen that should never silently default, since it's the primary reliability signal); the Submit button (Section [F]) stays disabled until this is chosen.
**Animation:** fill-color transition on selection, `duration-instant` (100ms), `ease-standard`; selecting either segment triggers section [D] (star rating) to become interactive if it wasn't already — see Section [D] below for the reasoning on sequencing.

### [D] Star rating
**Purpose:** Goal 2 — the required 1–5 `rating` field (§4.5).
**Layout:** five star icons in a horizontal row, centered, each a generous 32×32 tap target (larger than the small 14×14 display-only stars used elsewhere in this series on the Shortlist/Shop-Profile cards — this is the *interactive, selectable* star pattern, which research on rating components is clear needs a meaningfully larger, easily-tappable target than a read-only display version).
**Copy (label above):** "How would you rate the visit overall?"
**Interaction:** tap-to-select, with a live label beneath the stars that updates to a plain-language descriptor as the customer taps or hovers/drags across them (1 "Poor" · 2 "Below average" · 3 "Okay" · 4 "Good" · 5 "Excellent") — directly implementing the "each point on the scale should have a clear meaning" principle from the rating-UI research reviewed, so a customer isn't left guessing what a "3" is supposed to represent relative to a "4."
**Sequencing relative to [C]:** stars are visually present but rendered at reduced opacity (~40%) and non-interactive until an outcome is chosen in [C] above — a small but meaningful nudge that reinforces Goal 1's priority (answer the more important question first) without hard-blocking the screen into a rigid multi-step wizard; both fields are still on one screen, this is a soft visual sequencing cue, not a gate.
**Color:** selected stars filled in the same gold/amber tone used for display-only stars elsewhere in this product (the Section-4-noted brand-guide exception carried over here, since consistency in what "a star rating" looks like across the whole app matters more than strict palette purity); unselected stars a plain outline in Border-gray.
**Accessibility:** each star exposes a proper accessible label ("1 Star," "2 Stars," etc., per standard rating-component accessibility guidance) rather than relying on color/fill alone to communicate the selected value — the live plain-language label beneath the stars (Section above) does double duty here, since it's already visible to sighted and screen-reader users alike.
**Animation:** each star scales briefly (1.0→1.15→1.0, ~150ms, `ease-standard`) as it's selected, a small satisfying tap-response consistent with the "micro-interactions... make the rating process more engaging" principle from the research reviewed — brief and one-time per tap, not a sustained or looping effect.

### [E] Optional note field
**Purpose:** the optional `note` field (§4.5) — a lightweight escape valve for anything the star/outcome fields didn't capture, never presented as required.
**Copy (placeholder):** "Anything else you'd like to share? (optional)"
**Layout:** multi-line text area, 3 visible rows, 12px radius, Surface white background, 1px Border-gray outline, sits below the star rating with clear visual separation (extra top margin) so it doesn't read as connected to/required alongside the stars.
**Character limit:** soft cap ~200 characters — enough for a genuine comment, not an invitation to write a full review; a small counter appears only once the customer starts typing, not as a persistent "0/200" from the start (which would visually suggest more obligation than intended for an optional field).
**Color:** default border Border-gray; focus border Primary orange 2px, same as every other text input in this product.
**Animation:** border-color transition on focus, `duration-instant`, `ease-standard`.

### [F] Submit button
**Layout:** fixed to the bottom of the screen, full-width minus 16px margins, 48px height, 12px radius, sits above the safe-area inset.
**Copy:** "Submit"
**States:** disabled (Gray 300 background, Text-secondary text) until **both** [C] (outcome) and [D] (star rating) have a selection — the two required fields per schema; enabled (filled Primary orange, White Button-weight text) once both are set; loading (in-flight `POST /requests/:id/feedback` call) shows a centered spinner in place of the label, same pattern as every other submit button in this series.
**Behavior:** on success, transitions to a brief, calm confirmation state (not a full separate screen — see below) before returning to Home.
**Animation:** tap-scale-to-0.97; on success, a small inline **"Thanks for letting us know"** confirmation cross-fades in over the button's position (`duration-quick`, 180ms) for about 1 second, then the whole screen cross-fades back to Home (`duration-standard`, `ease-out-soft`) — deliberately brief and quiet, matching the "tune celebration to the size of the action" principle already established on the Selection-Confirmation screen; rating a visit is a small, everyday action, so this gets a correspondingly small acknowledgment, not a dedicated success screen of its own.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Success | `#00B8A9` | "Found" segment's selected fill |
| Primary | `#FF5A36` | Submit button, note-field focus border |
| Text primary | `#14213D` | Shop name, question labels, note-field text |
| Text secondary | `#5B6472` | Star-rating live descriptor label, note-field placeholder |
| Border | `#D8DCE3` | Unselected outcome-segment outline, unselected star outline, note-field outline |
| Surface | `#F4F5F7` / `#FFFFFF` | "Not found" segment's selected fill uses a dark-neutral tone rather than a palette token — see note below / header, note field, page background (White) |

**"Not found" segment's fill color note:** rather than introducing a new hex value, use the existing **Text-primary** navy (#14213D) as this segment's selected-state fill (White text on it) — a strong, clear "this is selected" signal without borrowing Danger red (ruled out in Section 3[C]'s reasoning) or inventing a new gray tone not already in the palette.
**Star-rating gold/amber exception:** carried over identically from the Shortlist/Shop-Profile screens' display-only stars (Section 4 of those specs) — same reasoning, same scope limitation to star icons only.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Shop name (reminder row) | Body / 15px | 600 SemiBold |
| Outcome question label | H2 / 18px | 600 SemiBold |
| Outcome segment labels ("Found" / "Not found") | Body / 15px | 600 SemiBold |
| Star-rating question label | H2 / 18px | 600 SemiBold |
| Star-rating live descriptor ("Good," "Excellent," etc.) | Caption / 13px | 400 Regular |
| Note field placeholder/text | Body / 15px | 400 Regular |
| Character counter | Caption / 13px | 400 Regular |
| Submit button label | Button / 15px | 600 SemiBold |
| Success confirmation ("Thanks for letting us know") | Button / 15px | 600 SemiBold (matches the button label it replaces) |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Outcome-segment fill transition, tap-scale feedback, note-field focus border |
| `duration-quick` | 180ms | Success-confirmation cross-fade in place of the Submit button |
| `duration-standard` | 240ms | Screen-exit cross-fade back to Home after the brief confirmation |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Segment/star selection transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Screen-exit |
| Star-select scale pulse | 1.0→1.15→1.0, ~150ms, one-time per tap | [D], on each star selection |

**No looping animation anywhere on this screen** — consistent with every screen in this series that represents a settled, non-real-time interaction; the only motion here is direct response to the customer's own taps, nothing ambient or automatic.
**Reduced motion:** the star-select pulse and outcome-fill transition are covered by the app-wide `prefers-reduced-motion` query, degrading to instant color/state changes with no scale/pulse.

---

## 7. Images, icons & video — asset list

**No video, no photography beyond the small shop-reminder thumbnail** (an existing asset, not new).

| Asset | Format/size | Source |
|---|---|---|
| Shop photo thumbnail (reminder row) | 40×40, rounded 8px, or initials-avatar fallback | Reused from the shop's existing onboarding photo |
| Checkmark icon ("Found" segment) | Line icon, 20×20 | Icon library |
| X/slash icon ("Not found" segment) | Line icon, 20×20 | Icon library |
| Star icon (interactive, 5×) | 32×32 tap target, filled/outline states, gold/amber tone | Icon library — a larger rendering of the same star icon used in display-only form elsewhere |
| Close (X) icon (header) | Line icon, 24×24 | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| Screen entry | Triggered from the Home page's repurposed Active Request banner (Section 0's documented assumption) — arrives with the relevant `requestId` and `shopId` already known client-side |
| [B] Shop reminder | Shop name/photo already available from the original selection (no new fetch needed) |
| [C]/[D]/[E] | Client-side form state only until submit |
| [F] Submit | `POST /api/v1/requests/:id/feedback` with `{ outcome, rating, note? }` — server increments `shops.foundCount`/`notFoundCount` per `outcome` and triggers an immediate reliability-score recalculation (§3.1.2, `services/reliability.js`, run over the shop's full rated-visit history, not just this one visit) |

**Loading state:** none needed beyond the Submit button's own in-flight spinner — nothing on this screen requires a fetch before it can render.
**Error state:** a failed submission uses the shared toast/error-boundary pattern (§6.1) with the form state preserved (outcome/rating/note stay filled in, so the customer isn't asked to redo the whole thing) and the Submit button re-enabled for another attempt.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with every screen in this series; comfortably fits without scroll on a standard reference viewport, even with the note field expanded.
- **Keyboard handling:** the optional note field should not push the Submit button off-screen when focused — standard scroll-into-view behavior, consistent with every other text-input screen in this product.
- **Dismissal:** since this is an optional, banner-triggered prompt (Section 0), the close (X) action in the header and the standard OS/gesture back-action should both simply return to Home with nothing submitted and no confirmation-to-discard dialog — unlike the Live-Count screen's Cancel action (which has real consequences worth confirming), abandoning this form has no consequence beyond "this one visit won't count toward the score," which doesn't warrant an interruption to confirm.
- **Re-prompting:** if the customer dismisses this screen without submitting, the Home banner (Section 0) should keep showing the "How did your visit go?" prompt for a reasonable window (e.g. until the request's natural lifecycle would otherwise be considered stale) rather than nagging indefinitely or disappearing after one dismissal — this exact threshold is, like the initial trigger timing, a reasonable default the team should confirm rather than a value fixed by the source documents.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1.2, 4.5, FR-4.1/4.2) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). One design decision here is explicitly a filled gap rather than a literal spec requirement: the trigger mechanism (a repurposed Home-banner prompt a few hours after selection) is a documented, minimal-footprint assumption, since neither source document defines a visit-detection or feedback-reminder mechanism — flagged clearly so the team can confirm the exact timing threshold before building it.*
