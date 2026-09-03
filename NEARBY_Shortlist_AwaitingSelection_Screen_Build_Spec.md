# NEARBY — Shortlist / Awaiting Selection Screen Build Specification
### Everything needed to build the `awaiting_selection`-state view — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.1, 3.1.1, 3.1.2, 3.1.3, 3.8, 5.3, 6.3), NEARBY Brand/PWA Guide v1.0, and restaurant/store-list card pattern research (Zomato-style ranked list cards)

---

## 0. Scope — what changes the instant this screen appears

This screen is what the previous build (the Live-Count screen) hands off to the moment `request:ready` fires. Everything that was withheld during the `open` window is now available: **shop names, exact distances, ratings, and reliability all become visible for the first time on this screen** — the anti-gaming withholding rule from the prior spec ends exactly here, by design (§3.1).

Three backend rules from your own document directly shape this screen's layout and copy, and none of them are optional:

1. **Ranking is server-determined, not customer-sortable in v1** — shops are grouped into distance bands (width set by `DEFAULT_BAND_WIDTH_METERS`), and within each band sorted by `avgRating` descending, then `reliabilityScore` descending, with an explicit null-avgRating tiebreak so an unrated new shop never accidentally outranks an established, rated one in the same band (§3.1.1). This screen renders that order as-is — no client-side re-sort, no "sort by" control, since the ranking logic already encodes what the product considers "best."
2. **A shop can vanish between saying Yes and this screen rendering** — the shortlist-build step re-checks every Yes-respondent's current approval status the moment the window closes, and silently excludes any shop that's been blocked in the interim (§3.8). This screen's card list is exactly what the server sends; it never needs a "this shop is no longer available" error state for an individual card, because that filtering already happened server-side before `request:ready` was even sent.
3. **This isn't a 2-minute decision** — unlike the Live-Count screen's 120-second window, the customer has up to `DEFAULT_SELECTION_TIMEOUT_HOURS` (default **24 hours**, §3.1.3) to pick before the request auto-expires. This screen must not borrow the prior screen's urgent countdown-timer treatment — the deadline here is communicated calmly, as a date/time, not a ticking `mm:ss`.

---

## 1. Page goals, in priority order

1. Let the customer compare shops quickly — this is fundamentally a short ranked list, and the research on restaurant/store list cards is consistent that name, rating, and distance are the three things a card must surface without a tap.
2. Make the server's ranking legible without pretending it's arbitrary — a brief, honest "sorted by distance, then rating" line does more for trust than silently presenting an order the customer has no way to understand.
3. Make selection feel deliberate, not accidental — since a selection is effectively irreversible for that request (the customer is committing to physically visit), the tap-to-select flow includes one confirmation step, not a bare single tap.
4. Keep the 24-hour deadline visible but non-anxious — this list can be revisited calmly, unlike the live-count screen's genuine time pressure.

---

## 2. Layout overview

Single scrollable screen, standard list layout (not a modal sheet this time — this is a real "place" the customer may return to, so it gets the app's normal header treatment, not the focused-task modal styling used for New Broadcast/Live-Count).

```
┌─────────────────────────────┐
│ [A] Header (title + count)   │
├─────────────────────────────┤
│ [B] Sort-explanation strip   │
├─────────────────────────────┤
│ [C] Selection-deadline banner │
├─────────────────────────────┤
│ [D] Shop card list            │  ← the main content, scrolls
│     (repeating card component)│
├─────────────────────────────┤
│ [E] Select-confirmation modal │  ← triggered from a card
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, back-chevron left (returns to Home — the request stays `awaiting_selection`, nothing is lost by navigating away), centered title.
**Copy:** "**[N]** shops said yes" (dynamic count, matches the final Yes count from the closed window — the same number the customer watched climb on the Live-Count screen, now resolved into an actual list).
**Background:** White, 1px Border-gray bottom border appears on scroll (same subtle depth cue used on the Home page's top bar).
**Animation:** none.

### [B] Sort-explanation strip
**Purpose:** the transparency line from Goal 2 above — a single sentence that turns an otherwise-unexplained ranking into something the customer can trust.
**Copy:** "Sorted by distance, then rating" — Caption size, Text-secondary, left-aligned, sits directly below the header with generous (16px) padding.
**Layout:** plain text, no icon, no card — deliberately understated so it reads as a factual caption, not a feature being sold.
**Animation:** none.

### [C] Selection-deadline banner
**Purpose:** communicates the 24-hour (or whatever `DEFAULT_SELECTION_TIMEOUT_HOURS` is configured to) window calmly — this is the one place this screen borrows from the Live-Count screen's "there is a deadline" concept, but rendered in the calm register appropriate to a 24-hour window rather than a 2-minute one.
**Copy:** "Choose by **[formatted date/time]**" — e.g. "Choose by tomorrow, 6:45 PM" (relative-day phrasing for anything within 48 hours, falling back to an absolute date beyond that, per standard relative-time-formatting convention) — computed client-side from the request's `createdAt` + the server-communicated timeout hours, or read directly if the API exposes the resolved deadline timestamp.
**Layout:** thin full-width bar, Gray 100 background, small clock-outline icon (16×16, Text-secondary) + Caption text, sits below [B], no border/shadow.
**Color:** Text-secondary throughout — deliberately **not** Warning amber or any alert color, since 24 hours away is not yet something to flag urgently; this banner exists for orientation, not pressure (directly consistent with the brand's calm-tone rule, and a deliberate contrast with the Live-Count screen's Warning-amber final-10-seconds treatment, which *is* appropriate urgency at that timescale).
**Animation:** none — static informational content.

### [D] Shop card list — the main content
**Purpose:** the actual ranked shortlist, one card per Yes-respondent shop, in server-determined order.
**Card layout:** full-width card, Surface white background, 1px Border-gray outline, rounded 12px, 12px vertical spacing between cards, comfortable internal padding (16px).

**Card anatomy, top to bottom / left to right:**
- **Top-left:** shop photo thumbnail (56×56, rounded 8px) if the shop uploaded a storefront photo during onboarding (page-architecture item 2.1); if none, a fallback — a rounded 8px tile in Gray 100 with the shop name's first letter, Text-secondary, centered (an initials-avatar pattern, not a broken-image icon).
- **Top-right of the photo, first line:** shop name, Body size bumped to 600 SemiBold (the one piece of text on this card that most needs to stand out), Text-primary.
- **Second line:** category tag (e.g. "Electronics") in a small rounded-full pill, Gray 100 background, Caption text, Text-secondary — plus, inline after it, the open/closed status: "Open now" in Success teal text (with a small solid-dot prefix) or "Closed" in Text-secondary (never Danger red — being closed isn't an error, just a fact, though see the note below on why a closed shop can even appear here at all).
- **Third line:** distance + rating, side by side: "0.4 km away" (Caption, Text-secondary) · a 5-star rating row (filled stars in Warning-amber-adjacent gold tone — a small, deliberate exception noted below — with the numeric average alongside, e.g. "★ 4.6") **or**, if `avgRating` is null, a neutral "New to NEARBY" pill instead of a fabricated star row (never render zero/empty stars for a null rating — that reads as "bad," when the truth is simply "not yet rated," a distinction the null-avgRating tiebreak fix in your own doc already cared about getting right server-side, and the UI needs to carry that same honesty through).
- **Fourth line (reliability, rendered as a friendly badge, not a raw score):** a small pill reflecting `reliabilityScore` in human terms rather than a bare number — **"Usually reliable"** (score ≥ 80, Success-teal-tinted pill), **"Sometimes reliable"** (score 50–79, Gray-100-tinted pill, Text-secondary), or no badge at all for a shop still at the default score of 50 with zero rated visits yet (avoids implying a judgment where there's no data — consistent with the same "don't fabricate a signal" principle applied to the rating above).
- **Bottom, full-width within the card:** a single **"Select this shop"** button — filled Primary orange, 40px height, 12px radius, Button-weight White text.
**Tap targets:** the button is the primary action; tapping anywhere else on the card body (photo, name, the rest of the card) opens the fuller Shop Profile screen (page-architecture item 1.7) for a customer who wants more detail (full address, phone, hours, rating breakdown) before deciding — this dual-tap-target pattern (quick action on the card, drill-through on the rest) mirrors the standard restaurant/store-list card convention from the research reviewed, where a list card supports both an immediate action and an optional detail view without forcing every customer through the detail screen first.
**"Top pick" badge:** the first card in the list (i.e., the server's top-ranked shop) gets a small badge in its top-right corner — "Top pick" in a small Primary-orange-outlined pill — a light visual acknowledgment of the ranking from Goal 2, not a hard sell.
**Why a closed shop can appear here at all:** a shop that said Yes while open may have since closed (e.g., the window spanned closing time, or the customer takes hours to decide within the 24h deadline) — it still appears, honestly marked "Closed," rather than being silently hidden, since the customer may still choose to visit later or call ahead; only a *blocked* shop is server-side excluded (Section 0, rule 2), never a merely-closed one.

### [E] Select-confirmation modal *(triggered by tapping "Select this shop" on any card)*
**Purpose:** the one deliberate friction point before an effectively-irreversible action.
**Layout:** centered modal overlay, White card, rounded 16px, dimmed backdrop (Ink Navy at 40% opacity — the one place the customer surface borrows the Admin-navy tone for a backdrop, purely functional here, not branding).
**Copy:** "Select **[Shop Name]**?" (H2, Text-primary) + "The other shops that said yes will be notified they weren't picked — this won't affect their rating." (Caption, Text-secondary — a small but important piece of honesty, since it's directly promised in your own doc that un-selected Yes responses never penalize `reliabilityScore`, §3.1/§3.1.2, and telling the customer this plainly removes any hesitation about "am I hurting someone's business by not picking them").
**Actions:** "Cancel" (text-only, Text-secondary) and "Confirm" (filled Primary orange) — Confirm is the visually primary action here, unlike the Live-Count screen's cancel-confirmation dialog where the *safe* choice was primary; here the customer has already expressed intent by tapping Select, so confirming that intent is the expected next step, not something to visually discourage.
**On confirm:** `POST /api/v1/requests/:id/select` with the chosen `shopId` → `requests.status` becomes `selected`, `request:selected` fires to the chosen shop and every other Yes-respondent (their copy: "not selected this time," no reliability impact, per §3.1) → the customer's screen cross-fades to the Selection Confirmation screen (page-architecture item 1.8).
**Animation:** modal fades+scales in (`duration-quick`, 180ms, `ease-standard`) over the dimmed backdrop; on confirm, cross-fades to the next screen (`duration-standard`, 240ms, `ease-out-soft`) — consistent with every other confirm-and-transition pattern in this series.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced, with one noted exception)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | "Select this shop" buttons, "Top pick" badge outline, Confirm action in the selection modal |
| Success | `#00B8A9` | "Open now" status text, "Usually reliable" badge |
| Text primary | `#14213D` | Shop names, modal heading |
| Text secondary | `#5B6472` | Category tags, distance text, "Closed" status, deadline-banner text, "Sometimes reliable" badge, sort-explanation strip |
| Border | `#D8DCE3` | Card outlines |
| Surface | `#F4F5F7` / `#FFFFFF` | Fallback-avatar tile, category-tag pill, deadline banner (Gray 100) / cards, header, modal (White) |
| Ink Navy | `#14213D` at 40% opacity | Selection-modal backdrop dim only (functional overlay use, not a themed surface) |

**One deliberate exception:** star-rating icons use a **gold/amber star tone** rather than any of the four brand palette colors — this is a near-universal rating convention strong enough that deviating from it (e.g. rendering stars in brand orange) would actually reduce legibility of "this is a rating," a rare case where following an outside convention beats internal palette purity. Keep this exception scoped to star icons only; do not extend it elsewhere.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Sort-explanation strip | Caption / 13px | 400 Regular |
| Deadline-banner text | Caption / 13px | 400 Regular |
| Shop name | Body / 15px | 600 SemiBold |
| Category tag / open-closed status | Caption / 13px | 400 Regular |
| Distance / rating text | Caption / 13px | 400 Regular |
| Reliability badge text | Caption / 13px | 400 Regular |
| "Select this shop" button | Button / 15px | 600 SemiBold |
| Modal heading | H2 / 18px | 600 SemiBold |
| Modal body copy | Caption / 13px | 400 Regular |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Card/button tap-scale (0.97) |
| `duration-quick` | 180ms | Selection-confirmation modal fade+scale-in |
| `duration-standard` | 240ms | Card-list entrance stagger (see below), screen-exit cross-fade on confirmed selection |
| `stagger-step` | 40ms | Delay between cards on initial list paint |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Modal transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Card entrance, screen-exit |

**Card-list entrance:** cards fade in + translateY(10px→0) over `duration-standard` with `stagger-step` (40ms) between them on first paint — the exact same reveal pattern already used for the Home page's Repeat Request carousel, reused here since both are "a set of cards appearing at once" (consistency across the product, not a new pattern).
**No broadcast pulse anywhere on this screen** — the pulse is reserved exclusively for the `open`-window live state (prior spec); this screen represents a *resolved* window with a static, browsable result, and using the pulse here would misleadingly suggest something is still actively updating in real time when it isn't (aside from the customer's own action of selecting).
**Reduced motion:** the card-entrance stagger and modal fade are both covered by the app-wide `prefers-reduced-motion` query — degrade to a simple opacity-only fade with no translate/stagger delay.

---

## 7. Images, icons & video — asset list

**No video.** Shop photos are the one place in this entire screen series where real photography legitimately appears — but only shop-supplied photos (from onboarding), never stock imagery.

| Asset | Format/size | Source |
|---|---|---|
| Shop photo thumbnail | 56×56, rounded 8px, shop-supplied at onboarding (Cloudinary-hosted per the Master Build Doc's storage choice, §3) | User-generated, not a design asset |
| Fallback initials avatar | CSS-drawn (Gray 100 tile + first-letter text), no image asset | — |
| Star-rating icon | Filled/outline star, 14×14, gold/amber tone (Section 4 exception) | Icon library |
| Clock-outline icon (deadline banner) | Line icon, 16×16, Text-secondary | Icon library (same library used app-wide) |
| Back-chevron icon | Line icon, 24×24 | Icon library |
| "Top pick" badge | Text-only pill, no icon needed | — |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [A]/[D] Shop list | Delivered via the `request:ready` Socket.IO payload (or `GET /api/v1/requests/:id` if the client arrives here via a REST poll / push-notification deep link rather than a live socket event) — includes each shop's `name`, `photoUrl`, `category`, distance (computed server-side from the `2dsphere` query), `avgRating` (nullable), `reliabilityScore`, and current open/closed status |
| [C] Deadline banner | Computed from `createdAt` + `DEFAULT_SELECTION_TIMEOUT_HOURS` (or a server-resolved deadline timestamp, if the API exposes one directly — preferred, to avoid any client/server clock-skew edge case) |
| [E] Selection | `POST /api/v1/requests/:id/select` with `{ shopId }` → triggers `request:selected` (server → chosen shop's room + every other Yes-respondent shop's room) and the customer's transition to the Selection Confirmation screen |

**Loading state:** if this screen is reached via a fresh page load (e.g. a push-notification deep link after the app was closed) rather than a live in-app transition from the Live-Count screen, show a brief full-card skeleton list (3 gray placeholder cards, shimmer) while `GET /api/v1/requests/:id` resolves — the one place on this screen a skeleton is appropriate, since the whole page's content is this fetched list.
**Error state:** a failed initial fetch uses the shared toast/error-boundary pattern with a retry option (Master Build Doc §6.1); a failed `POST .../select` call (network hiccup) keeps the confirmation modal open with an inline error line beneath the Confirm button ("Something went wrong — try again") rather than silently closing the modal and losing the customer's already-made choice.
**Timeout edge case:** if the 24-hour deadline passes while the customer is looking at this exact screen without selecting, the selection-timeout sweep (§3.1.3) will eventually fire `request:expired` server-side — the client should listen for this event even on this screen (not just the Live-Count screen) and transition to the Expired screen's "shops responded but the window closed" copy variant (§6.3's `yesCount ≥ 1` case) if it arrives while the customer is still browsing the shortlist.

---

## 9. Responsive & platform notes

- **Viewport:** standard vertical list, max-width 480px centered, same convention as every other screen in this series — no grid/multi-column card layout at wider viewports, since a ranked list's order is the point, and a grid would obscure that ordering.
- **Long lists:** in the pilot's 20–50-shop, single-locality context, shortlists are expected to stay short (a handful of Yes-respondents per broadcast, not dozens) — no pagination or "load more" needed for v1; if a future multi-locality launch ever produces meaningfully longer lists, standard list virtualization would be the natural addition, noted here as a deliberate non-concern for the current pilot scope rather than an oversight.
- **PWA standalone mode:** this screen keeps the app's normal header/back-navigation chrome (Section 2) rather than the modal-sheet treatment used for New Broadcast/Live-Count — it's meant to feel like a page the customer can leave and return to over the 24-hour window, not a task they're mid-way through and shouldn't back out of.
- **Push-notification re-entry:** per the Master Build Doc's fix giving the customer-side push subscription its first real consumer (dispatch on `awaiting_selection`), a customer who taps that notification should land directly on this screen with the shortlist already resolved from the notification payload or an immediate fetch — not back on the (by-then-irrelevant) Live-Count screen.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 3.1.1, 3.1.2, 3.1.3, 3.8, 5.3, 6.3) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). The server-determined ranking (distance band → rating → reliability, with the null-avgRating tiebreak) is rendered as-is with no client-side re-sort control, and the reliability score is deliberately translated into a plain-language badge rather than exposed as a raw number — both choices preserve the integrity of the ranking logic your v2.6.1/v2.6.2 fixes specifically hardened.*
