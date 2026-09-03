# NEARBY — Home Page (Customer App) Build Specification
### Everything needed to build `/app` — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2, NEARBY Brand/PWA Guide v1.0, and a Blinkit/Zepto/Zomato home-screen analysis

---

## 0. Scope — read this first

There are, precisely, **two different "home page" concepts** in a PWA like NEARBY, and the Brand Guide (Section 9.6) explicitly distinguishes them. This document builds **both**, but the primary deliverable is #1 — it's the page the whole product actually runs on.

1. **In-app Home screen** (`/app` or `/app/home`) — what a customer sees every time they open the installed PWA. Calm, fast, minimal motion, no "marketing" feel. **This is Section 1–9 below.**
2. **Pre-install marketing landing page** (only if NEARBY ever builds one at the root domain, separate from the installed app) — a one-time page a visitor sees *before* installing. Heavier, showcase-style motion is allowed here per Brand Guide 9.6. **This is Section 10, kept short since it's out of current build scope.**

If your team only needs one of these, it's #1 — build that first.

---

## 1. Page goals (what this screen has to do, in priority order)

1. Get a returning customer into a new broadcast in **one tap** — this is the core action, everything else is secondary (mirrors Blinkit's sticky top search bar — the single most-praised element in every UX teardown reviewed).
2. Show a **first-time user** what NEARBY actually does in under 5 seconds, without a forced walkthrough.
3. Surface an **in-progress request** if one exists, so the customer never has to hunt for it (this is NEARBY's version of Zepto's live-order-tracking banner).
4. Offer **one-tap repeats** of recent searches (NEARBY's version of Blinkit/Zepto's "Buy Again").
5. Never invent urgency — countdown/live-count copy stays neutral per Brand Guide Section 1.4 ("3 shops have answered," never "Hurry, only 3 shops left!").

---

## 2. Layout overview

Mobile-first, single-column, max content width 480px centered on tablet/desktop (this is a utility PWA, not a marketing site — no wide multi-column hero). Standard vertical stack, top to bottom:

```
┌─────────────────────────────┐
│ [A] Top bar (logo + location)│  ← sticky
├─────────────────────────────┤
│ [B] Search / Broadcast bar   │  ← sticky, sits just below A
├─────────────────────────────┤
│ [C] Active Request banner    │  ← conditional
├─────────────────────────────┤
│ [D] Quick Category chips     │
├─────────────────────────────┤
│ [E] Repeat Request carousel  │  ← conditional (returning users)
├─────────────────────────────┤
│ [F] How It Works (3 steps)   │  ← conditional (first-time users)
├─────────────────────────────┤
│ [G] Recent Activity list     │  ← conditional (returning users)
├─────────────────────────────┤
│ [H] Become-a-shop CTA card   │
├─────────────────────────────┤
│ [I] Bottom tab bar            │  ← fixed
└─────────────────────────────┘
```

Sections A and B are the only sticky elements (matches Blinkit's single praised sticky-search pattern — don't make more than the search bar sticky, or the page feels cluttered).

---

## 3. Section-by-section spec

### [A] Top bar
**Purpose:** orientation — where am I, what's my location.
**Layout:** single row, 56px height. Left: NEARBY logomark (24×24, Section 2 Brand Guide) + wordmark "NEARBY," Inter H2/SemiBold, Text-primary (#14213D). Right: location pill.
**Location pill:** rounded-full chip, Gray 100 background (#F4F5F7), pin icon + truncated address ("Civil Lines, Prayagraj" — max 24 chars then ellipsis), tap opens the manual-address override (Section 6.2, Master Build Doc). Caption-size text (13px), Text-secondary color (#5B6472).
**Background:** White (#FFFFFF), 1px bottom border in Border gray (#D8DCE3) on scroll (appears only once content scrolls under it — subtle depth cue, no shadow).
**Animation:** none — this bar should feel completely static/reliable (calm-tone rule).

### [B] Search / Broadcast bar
**Purpose:** the single most important element on the page — start a new broadcast.
**Copy (placeholder text, rotates or static — pick static for v1 simplicity):**
> "What are you looking for nearby?"

**Layout:** full-width rounded input (12px radius), 48px height, magnifying-glass icon left, Surface white background, 1px Border gray outline. Sits directly below the top bar, becomes sticky once the user scrolls past it.
**Interaction:** tapping opens the full **New Broadcast screen** (item 1.3 in the page-architecture doc) — this input is a launcher, not an inline search-as-you-type field, since NEARBY has no product catalog to autocomplete against (unlike Blinkit/Zepto, which autosuggest from a live SKU index).
**Color:** placeholder text in Text-secondary (#5B6472); icon in Text-secondary; on focus, border becomes Primary (#FF5A36), 2px.
**Animation:** border-color transition on focus, duration-instant (100ms), ease-standard.

### [C] Active Request banner *(shown only if the customer has a request in `open`, `awaiting_selection`, or `selected` status)*
**Purpose:** NEARBY's equivalent of a live-order-tracking strip — never let a customer lose track of an in-flight broadcast.
**Layout:** full-width card, rounded 12px, sits directly under the search bar, Success-tinted background (#00B8A9 at 10% opacity) with a solid Success-color left accent bar (4px).
**Content by state:**
- `open`: "Your search for **[product]** is broadcasting — **[N] shops have answered**" + live countdown chip (mm:ss to `expiresAt`) + tap-through to the Request-in-progress screen.
- `awaiting_selection`: "**[N] shops** said yes — pick one to visit" + tap-through to Shortlist screen. Background shifts to Warning-tinted (#FFB627 at 10%).
- `selected`: "You're headed to **[Shop Name]**" + shop's phone-call icon shortcut. Background stays Success-tinted.
**Animation:** the broadcast-pulse pattern (Brand Guide 9.2) — while `open`, a small version of the logomark's three arcs animates as looping staggered expanding rings, ~1.6s loop, CSS `@keyframes`, opacity fade, confined to a 20×20px icon inside the card (not full-screen — this is a home-screen banner, not the dedicated tracking screen). The live count-up (N shops answered) eases from old→new value over duration-quick (180ms), never overshoots.
**If no active request:** this section collapses entirely — zero height, no placeholder card (don't show an empty-state card here; save empty-state treatment for sections that are core to first-use, like G).

### [D] Quick Category chips
**Purpose:** lower the activation energy for a first-time or indecisive user — a scaffold, not a catalog (NEARBY has no product catalog, so unlike Blinkit's category *grid* leading to a filtered product list, these chips **pre-fill the search bar text** and jump straight to the New Broadcast screen).
**Layout:** horizontal scroll row, 8 chips, rounded-full pills, 36px height, Gray 100 background, Text-primary label, small emoji or line-icon prefix.
**Suggested chip set (pilot-locality generic — swap per category mix once real shop categories are known):** Groceries · Medicines · Electronics · Hardware · Stationery · Mobile Accessories · Bakery · Household
**Animation:** none beyond standard tap scale-to-0.97 (Brand Guide 9.3).

### [E] Repeat Request carousel *(shown only if the customer has ≥1 past request)*
**Purpose:** NEARBY's "Buy Again" equivalent — Zepto and Blinkit both cite this as their single highest-leverage repeat-engagement section.
**Header:** "Search again" — H2 SemiBold, Text-primary, left-aligned, 16px bottom margin.
**Layout:** horizontal-scroll card row, each card 140×80px, rounded 10px, Surface white background with Border-gray 1px outline. Card content: product text (truncated 2 lines, Body size), small caption below showing "Last time: [Shop Name]" if that request ended in `selected`, or nothing if it expired/was cancelled.
**Interaction:** tap re-fills the New Broadcast screen's product field with that past query (radius resets to default — location is never reused per Section 6.2, deliberate no-persistence rule) and lets the customer confirm before broadcasting again, rather than auto-submitting silently.
**Animation:** cards fade in + translateY(10px→0) over duration-standard (240ms) with stagger-step (40ms) between cards on initial paint — same pattern as the shortlist reveal (Brand Guide 9.2), reused here since both are "a list of cards appearing."

### [F] How It Works *(shown only to first-time users — i.e., zero past requests; disappears permanently once the customer completes their first broadcast, replaced by section E)*
**Purpose:** Since NEARBY's model (broadcast → shops answer → pick one) is not an interaction pattern people already know from Blinkit/Zepto/Zomato, a lightweight explainer earns its place here — this is the one place on the home screen where an explainer is justified (LogRocket's hero-section research: skip explainer hero content for browse-first products, but NEARBY's home screen genuinely isn't browse-first, it's broadcast-first, and first-timers won't intuit that from the search bar alone).
**Layout:** 3-step horizontal strip (or stacked on very narrow viewports <340px), each step a simple icon + one line of copy, no illustration/photography — matches the brand's "no 3D/showcase motion" rule (Brand Guide 9.6).
**Copy:**
1. **Ask** — "Tell us what you need"
2. **Broadcast** — "Nearby shops get notified instantly"
3. **Pick** — "Choose who to visit from those who said yes"
**Color:** icons in Primary orange, circled in Gray 100. Text in Text-primary.
**Animation:** static — this is instructional content, not a live state; no motion needed or wanted.

### [G] Recent Activity list *(shown only for returning users; shows last 3 requests, "See all" link to full history — item 1.12)*
**Layout:** simple list rows, each with: product text, relative timestamp (Caption, Text-secondary), status chip (colored per Semantic Usage table — Success green for `selected`, Warning amber for `expired`/`cancelled` is arguably wrong-toned since those aren't warnings; use neutral Gray chip for `expired`/`cancelled`, Success for `selected`).
**Empty state (first-time user, before section F disappears and this would otherwise show):** don't render this section at all until the customer has ≥1 completed request — no "no activity yet" placeholder needed since section F already communicates onboarding.

### [H] Become-a-shop CTA card
**Purpose:** growth surface — every customer is a potential shop referral in a hand-onboarded pilot (Section 1, Master Build Doc: 20–50 hand-onboarded shops).
**Layout:** full-width card, rounded 12px, Ink Navy background (#14213D) — this is the *one* place the customer surface deliberately borrows the Admin surface's navy, since this is functionally an "operator-facing" action even though it lives on the customer screen. White text.
**Copy:** "Own a shop nearby? **Join NEARBY** and start getting matched with customers looking for what you sell." + "Get Started" button (white background, Navy text — inverted from the card, for contrast).
**Animation:** none — static promotional card, no motion budget spent here.

### [I] Bottom tab bar
**Tabs:** Home (active) · History · AI Assistant · Profile — 4 tabs, icon + label, 56px height, fixed to viewport bottom, White background, top 1px Border-gray divider.
**Active state:** Primary orange icon + label; inactive: Text-secondary gray.
**Animation:** none beyond standard tap feedback.

---

## 4. Color usage summary (all values from Brand Guide Section 3.1/3.2 — do not introduce new hex values)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | Search bar focus border, active tab icon, "How It Works" icons, category-chip active state |
| Primary-pressed | `#E24521` | Any Primary button's pressed state (e.g. Become-a-shop button if ever inverted) |
| Success | `#00B8A9` | Active Request banner (`open`/`selected` states), broadcast-pulse rings |
| Warning | `#FFB627` | Active Request banner (`awaiting_selection` state) |
| Danger | `#E5484D` | Not used on Home in a neutral flow — reserved for error toasts only |
| Text primary | `#14213D` | Body copy, headers, product text |
| Text secondary | `#5B6472` | Captions, timestamps, placeholder text, location pill |
| Border | `#D8DCE3` | Card outlines, input borders, top-bar scroll divider |
| Surface | `#F4F5F7` / `#FFFFFF` | Page background (Gray 100) / card and input backgrounds (White) |
| Ink Navy | `#14213D` | Become-a-shop CTA card background (borrowed intentionally, see Section H above) |

**Contrast note (Brand Guide 3.4):** Orange on white passes AA for large text/icons only, not body copy — so no orange body-copy text anywhere on this page; orange is icon/button/border only, matching the spec above throughout.

---

## 5. Typography summary (Brand Guide Section 4.2 — Inter, system-fallback stack)

| Element | Role/size | Weight |
|---|---|---|
| "NEARBY" wordmark (top bar) | H2 / 18px | 600 SemiBold |
| Section headers ("Search again", "How It Works") | H2 / 18px | 600 SemiBold |
| Search bar placeholder | Body / 15px | 400 Regular |
| Active Request banner main line | Body / 15px | 400 Regular (bold-weight product name inline via `<strong>`, not a separate style) |
| Category chip labels | Caption / 13px | 400 Regular |
| Recent Activity timestamps | Caption / 13px | 400 Regular |
| Become-a-shop CTA button | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide Section 9.1–9.5 — reuse these tokens exactly, do not invent new durations)

| Motion token | Value | Applied to on this page |
|---|---|---|
| `duration-instant` | 100ms | Search-bar focus border, chip/card tap-scale (0.97) |
| `duration-quick` | 180ms | Live count-up ease in Active Request banner, skeleton→content swap |
| `duration-standard` | 240ms | Repeat Request carousel card reveal, screen-transition fade on tap-through |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Focus-state transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Carousel card entrance |
| `stagger-step` | 40ms | Delay between Repeat Request cards on initial paint |
| Broadcast pulse | ~1.6s loop, opacity fade | Small logomark icon inside Active Request banner, `open` state only |

**Reduced motion:** a single `prefers-reduced-motion: reduce` query (already defined app-wide per Brand Guide 9.5) disables the broadcast pulse and carousel stagger on this page automatically — no page-specific override needed.

**Explicitly no motion on:** top bar, category chips (beyond tap feedback), How-It-Works section, Become-a-shop card, bottom tab bar. Motion budget on this page is spent entirely on things that represent *live state* (Active Request banner) or *list reveals* (carousel) — matches the brand's "motion signals real-time state, not decoration" rule (Brand Guide Section 9 intro).

---

## 7. Images, icons & video — asset list

**No video anywhere on this page** (and no video anywhere in the PWA per the Brand Guide's zero-cost/zero-latency loading decisions — Section 4.1's font-fallback and Section 6's no-splash-image default both point the same direction: nothing that adds load weight for a free-tier Render/Netlify stack).

| Asset | Format/size | Source |
|---|---|---|
| Logomark (top bar) | SVG, 24×24 render of `nearby-mark.svg` | Already delivered, Brand Guide §7.1 |
| Logomark arcs (broadcast pulse, Active Request banner) | Inline SVG/CSS animation, ~20×20px | Derived from `nearby-mark.svg`'s arc paths — no new asset, reuse existing source file |
| Location pin icon | Line icon, 16×16, Text-secondary color | Icon library (e.g. Lucide/Feather — pick one and use consistently app-wide) |
| Search/magnifying-glass icon | Line icon, 20×20 | Same icon library |
| Category chip icons | Emoji (fastest, zero-asset-cost) or 16×16 line icons if the team prefers a more custom feel | Emoji recommended for v1 — zero image requests, matches the "no extra cost" pattern running through both source docs |
| How-It-Works step icons | 3× line icons (a speech-bubble for "Ask", a radiating-signal glyph for "Broadcast" — can literally reuse the logomark's arc motif, a checkmark/hand for "Pick") | Icon library + one reused brand asset |
| Empty/placeholder states | No illustration — this page deliberately has none per Section 3.G above | — |

**No stock photography, no shop photos on this screen** — shop photos belong on the Shortlist and Shop Profile screens (page-architecture doc, items 1.6–1.7), not Home. Keeping Home photo-free is itself a deliberate performance/consistency choice, not an oversight — it matches Blinkit's own home-screen critique in the UX research reviewed, where product-card photography is exactly what makes that screen heavy, and NEARBY's Home has no products to photograph in the first place.

---

## 8. Data & state requirements (what has to be wired up for this page to work)

| Section | Data needed | Endpoint / event |
|---|---|---|
| [A] Location pill | Current resolved location (from last geolocation/geocode) | Client-side state only, not persisted across sessions per Section 6.2 |
| [C] Active Request banner | Any request in `open`/`awaiting_selection`/`selected` status for this customer | `GET /api/v1/requests/:id` on load; live updates via `response:update`, `request:ready`, `request:selected` Socket.IO events (Master Build Doc §5, §6.3) |
| [E] Repeat Request carousel | Last N requests, most recent first | Request history endpoint, filtered/sorted client-side or via query param |
| [G] Recent Activity | Same history data, last 3 | Same endpoint as E, different slice |
| [F] visibility toggle | "Has this customer ever completed a request?" flag | Derived from history endpoint returning empty vs. non-empty |
| [H] Become-a-shop | Static — no data dependency | Links to `POST /shops` flow (page-architecture item 2.1) |

**Loading state:** skeleton loaders (Brand Guide 9.4) for sections C, E, and G only while their data is in flight — gray placeholder blocks, Gray 100/300, left-to-right shimmer, swapped for real content over `duration-quick`. Sections A, B, D, F, H, I render immediately (no data dependency, no skeleton needed).

**Error state:** if the history/active-request fetch fails, sections C/E/G simply don't render (fail silent, no error banner on Home) — a home-screen data hiccup shouldn't block the one thing that matters, which is the search bar. The shared toast/error-boundary pattern (Master Build Doc §6.1) can still fire in the background if useful for debugging, but it must never block interaction with [B].

---

## 9. Responsive & platform notes

- **Viewport:** designed mobile-first (360–428px reference width); scales to a centered 480px-max column on tablet/desktop rather than reflowing into multi-column — this is a utility app used primarily on-the-go, not a desktop dashboard.
- **PWA standalone mode:** per manifest `display: "standalone"` (Brand Guide §5.1), no browser chrome — the top bar [A] effectively becomes the app's only "chrome," so its 56px height and static (no-motion) behavior matters more here than it would in a browser tab.
- **Cold-start:** Render free-tier cold starts mean the Active Request banner's data may lag a beat behind first paint — the skeleton state (Section 8) exists specifically to cover this, consistent with Master Build Doc §8.4's cold-start handling elsewhere in the app.
- **Install prompt:** the PWA install prompt, if shown, should not compete with section [B] — surface it as a dismissible banner *above* [A], not as a modal blocking the search bar.

---

## 10. (Optional / future) Pre-install marketing landing page — short addendum

Only relevant if NEARBY builds a separate root-domain page before the installed app, per Brand Guide §9.6's explicit carve-out. Not required for the current build.

If built, it may differ from Sections 1–9 above in these specific ways:
- A genuine **hero section** is justified here (unlike the in-app Home) — LogRocket's hero-section research supports one for a not-yet-adopted, one-time-visit page: headline restating the tagline **"Ask around. Instantly."**, one-line subhead explaining the broadcast model, single CTA ("Install NEARBY" / "Try it now").
- Heavier motion is explicitly permitted (Brand Guide 9.6): a feature carousel or an illustrated hero animation showing the broadcast-pulse concept at a larger, marketing scale — this is the *only* place in the whole product where that's sanctioned.
- Should still end with the same 3-step How-It-Works content as Section [F] above, and the same Become-a-shop CTA as Section [H] — reuse copy, don't reinvent it.
- Must not use the app's own PWA manifest/start_url routing — this page sits outside the installed app per definition.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.1, 6.1–6.3) and NEARBY Brand/PWA Guide v1.0 (Sections 1.4, 2–5, 9). No color, motion token, or copy tone introduced here contradicts either source document.*
