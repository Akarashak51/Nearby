# NEARBY — Cancelled Screen Build Specification
### Everything needed to build both copy variants of the `cancelled`-state screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.8, 3.9A, 6.3), NEARBY Brand/PWA Guide v1.0, and empty-state/account-status UX research (same body of research applied to the Expired screen, extended here to a genuinely dead-end case)

---

## 0. Scope — the two variants, and the one that breaks the "never a dead end" rule on purpose

Per §6.3, this screen's copy "distinguishes customer-initiated vs. auto-cancel-on-suspension" — and per §3.8, these are two very different events:

| Case | Trigger | Who can still act afterward |
|---|---|---|
| **Customer-initiated** | The customer tapped Cancel on the Live-Count screen and confirmed (§3.8, "a customer may cancel their own request any time while status is `open`") | Fully normal — nothing else about their account has changed |
| **Auto-cancel-on-suspension** | An admin suspended or banned the customer's account while their request was `open`; the system cancels it automatically, no separate admin action needed (§3.8) | **Nothing** — §3.8 is explicit elsewhere that "a suspended or banned customer cannot log in or broadcast" |

Every empty-state build in this series so far has followed the rule "never a dead end, always one recovery action." **This screen's second variant is the one deliberate exception to that rule in the whole product**, and that's not an oversight — it would be dishonest to show a "search again" button to a customer whose account literally cannot search again. Section 3 below specs both variants explicitly, including where Variant B stops offering next steps.

**A narrow but real timing note:** since a suspended/banned customer can't log back in afterward, this Variant-B screen is likely the *last* screen that customer sees from an active session — probably arriving via a live Socket.IO event while they were still watching their Live-Count screen, moments before their session's next state-changing request gets rejected by `middleware/requireActive.js` (§5.3's JWT-status-recheck logic). The copy has to work for exactly that moment.

---

## 1. Page goals, in priority order

1. **Variant A:** confirm the cancellation went through cleanly, and get the customer back to searching with zero friction — this was their own choice, and it should feel like a closed loop, not a setback.
2. **Variant B:** state plainly and factually what happened, without being alarmist or accusatory, and without pretending there's a next step when there isn't one.
3. Both variants: never use blaming or apologetic language (per the empty-state tone principle already established in the Expired-screen spec) — Variant A didn't do anything wrong, and Variant B's copy is a factual notice, not a punishment delivered through UI copy.

---

## 2. Layout overview

Shared shell, same general single-focus status-screen pattern as every "outcome" screen in this series — but **the two variants diverge more structurally here than the Expired screen's two variants did**, since Variant B has no action row.

```
┌─────────────────────────────┐
│ [A] Header (minimal)          │
├─────────────────────────────┤
│ [B] Icon + headline (variant-specific) │
├─────────────────────────────┤
│ [C] Explanation copy (variant-specific) │
├─────────────────────────────┤
│ [D] Original request recap    │  ← Variant A only
├─────────────────────────────┤
│ [E] Primary action / account notice │  ← content differs entirely by variant
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row. **Variant A:** back-chevron left (returns to Home — this was the customer's own choice, nothing prevents normal navigation). **Variant B:** no back-chevron — there's no meaningful "back" destination for a session that's about to lose access anyway; an empty header bar is more honest than a back action that may immediately fail.
**Background:** White, no border/shadow, both variants.
**Animation:** none.

### [B] Icon + headline — variant-specific

**Variant A — Customer-initiated:**
- **Icon:** a simple outline "x-in-circle" or slash glyph, 64×64, Text-secondary tone — plain and neutral, not styled as an error.
- **Headline:** "Search cancelled" (H1, Text-primary, 600 SemiBold).

**Variant B — Auto-cancel-on-suspension:**
- **Icon:** a simple outline account/shield-alert glyph, 64×64, **Warning amber** tone (#FFB627) — the one deliberate departure from the neutral Text-secondary icon tone used everywhere else in this "outcome screen" family. This is a genuine account-level notice, not a routine product outcome like an expired search, and Warning amber (per the Brand Guide's semantic usage — "use with caution" states) is the right register: serious enough to signal "pay attention," without reaching for Danger red, which the brand reserves for destructive actions and validation failures rather than account-status notices.
- **Headline:** "Your search was cancelled" (H1, Text-primary, 600 SemiBold) — deliberately does not lead with "Account suspended" as the headline; the cancellation is the direct, concrete fact this screen exists to report, and the account-status explanation follows immediately in [C] rather than leading with what could otherwise read as an accusation before any context is given.

**Animation (both variants):** icon fades in (`duration-standard`, 240ms, `ease-out-soft`) on mount — same restrained, one-time entrance as the Expired screen's icons, no looping.

### [C] Explanation copy — variant-specific

**Variant A copy:** "You cancelled this search. The shops that had already been notified know it's no longer needed." (Body, Text-secondary) — this second sentence matters: it closes the loop on something the customer may wonder about (did the shops just get left hanging?), directly answering it without being asked, consistent with the confirmation-modal copy already established on the Shortlist/Shop-Profile screens' select flow ("this won't affect their rating" — the same instinct to proactively explain a side effect the customer can't otherwise see).

**Variant B copy:** "This search was cancelled because your account has been suspended. If you believe this is a mistake, contact support." (Body, Text-primary — bumped from Text-secondary to Text-primary for this one variant, since this is the single most important sentence the customer will read in this flow, and it deserves full-strength text color rather than the muted secondary tone used for routine explanatory copy elsewhere). **Does not** explain *why* the account was suspended (no report details, no specifics) — that information belongs to the admin side's moderation record, not this screen; stating the fact of suspension plainly is honest, relitigating the reason is not this screen's job and risks either exposing sensitive report data or sounding punitive.

### [D] Original request recap *(Variant A only)*
**Purpose:** identical in function to the recap card used on the Live-Count and Expired screens — reminds the customer what they were searching for, useful if they're about to search again.
**Layout:** same card treatment, reused directly (🔍 product query, 📍 radius, 📌 location) — Surface white background, 1px Border-gray outline, rounded 12px.
**Not shown in Variant B:** the recap is irrelevant to a customer who's about to lose access to the account entirely — showing "here's what you were searching for" alongside an account-suspension notice would read as tone-deaf, and adds nothing useful given there's no action this screen can offer them anyway.
**Animation:** none — static, same as every other instance of this card.

### [E] Primary action / account notice — the section that diverges completely

**Variant A:** a single full-width filled Primary-orange button, "Search again" (48px height, 12px radius, Button-weight White text) — pre-fills a new New Broadcast screen with the same `productQuery`/`location`/`radius` as the cancelled request (identical pre-fill behavior to the Expired screen's Variant-B "Search again" action — same product query, same location, same radius, no adjustment implied since nothing about this cancellation suggests the original parameters were wrong). Below it, a plain "Back to Home" text link (Caption, Text-secondary), same secondary-action pattern as the Expired screen.

**Variant B:** **no button, no "search again," no "back to Home" link.** In their place, a single plain informational row:
- A "Contact support" text link (Caption, Primary orange — the one interactive element on this variant) — opens a pre-addressed support contact method (e.g. a `mailto:` link to the platform's support address, or an in-app contact form if one exists elsewhere in the product; whichever the team already has, reused here rather than building a new channel just for this screen).
- Directly below it, a plain "Sign out" text link (Caption, Text-secondary) — since continuing to browse a session that's about to be rejected by `requireActive.js` on the next state-changing request serves no purpose; offering a clean, deliberate sign-out is more respectful than letting the customer stumble into an unexplained 401 on their next tap.
**Why this is the right call despite "never a dead end":** the empty-state research this whole screen series has followed is written for product states — no results, no tasks, no data — where a next action always exists somewhere in the product. An account-suspension notice is categorically different: pretending a next step exists here would be a worse experience than honestly saying there isn't one, and quietly offering an off-ramp (contact support, sign out) is the honest version of "give the user something to do," not a violation of that principle.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | Variant A's "Search again" button, Variant B's "Contact support" link |
| Warning | `#FFB627` | Variant B's icon only — the one use of Warning tone anywhere in this screen family |
| Text primary | `#14213D` | Headline (both variants), Variant B's explanation copy (bumped from secondary per Section 3[C]) |
| Text secondary | `#5B6472` | Variant A's icon tone and explanation copy, recap-card labels, both variants' secondary links ("Back to Home" / "Sign out") |
| Border | `#D8DCE3` | Recap card outline (Variant A only) |
| Surface | `#FFFFFF` | Header, recap card, page background — this screen stays plain white throughout, no Gray-100 tinted strips, consistent with the Expired screen's deliberately unadorned treatment |

**No Danger red anywhere on this screen, either variant** — same reasoning as the Expired screen: even Variant B's account-suspension case isn't framed as an "error" the interface is reporting on itself; Danger stays reserved for destructive actions and validation failures elsewhere in the product, and Warning amber is the correct, one-step-more-serious register for an account-level notice without overstating it as Danger-level alarm.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Headline (both variants) | H1 / 24px | 600 SemiBold |
| Variant A explanation copy | Body / 15px | 400 Regular |
| Variant B explanation copy | Body / 15px | 400 Regular (Text-primary color per Section 3[C], not a weight change) |
| Recap-card labels (Variant A) | Body / 15px | 400 Regular |
| "Search again" button label (Variant A) | Button / 15px | 600 SemiBold |
| "Back to Home" / "Sign out" links | Caption / 13px | 400 Regular |
| "Contact support" link (Variant B) | Caption / 13px | 400 Regular (color-differentiated via Primary orange, not weight) |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Tap-scale feedback on all interactive elements, both variants |
| `duration-standard` | 240ms | Icon fade-in on mount (both variants), screen-exit on Variant A's "Search again"/"Back to Home" |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Icon entrance, screen-exit transitions |

**No looping animation, no celebratory motion, in either variant** — consistent with the entire "outcome screen" family's restrained motion budget; Variant B in particular should feel calm and unhurried despite carrying more consequential news, not urgent or alarming, which sustained or attention-grabbing motion would work against.
**Reduced motion:** icon appears in final state directly, no fade — covered by the app-wide `prefers-reduced-motion` query, same as every other screen in this series.

---

## 7. Images, icons & video — asset list

**No video, no photography, either variant.**

| Asset | Format/size | Source |
|---|---|---|
| X-in-circle / slash glyph (Variant A icon) | Outline icon, 64×64, Text-secondary | Icon library |
| Account/shield-alert glyph (Variant B icon) | Outline icon, 64×64, Warning amber | Icon library |
| Recap-card icons (🔍 📍 📌, Variant A only) | Emoji, consistent with every earlier recap card in this series | Reused, no new asset |
| Back-chevron icon (Variant A header only) | Line icon, 24×24 | Icon library |

---

## 8. Data & state requirements

| Section | Data / logic |
|---|---|
| [B]/[C]/[E] variant selection | Driven by which trigger produced the `request:cancelled` event: a client-initiated cancel (the customer's own `POST /requests/:id/cancel` call, so the client already knows this is Variant A the instant it fires) vs. a server-initiated auto-cancel arriving as an unsolicited `request:cancelled` event the client didn't request (Variant B) — the payload should include enough to distinguish the two cleanly (e.g. a `reason` field such as `customer_initiated` vs `account_suspended`) so the client never has to guess |
| [D] Recap card (Variant A) | Same `productQuery`, `radiusMeters`, `formattedAddress` fields already used by the Live-Count/Expired screens' identical card |
| [E] Variant A "Search again" | Client-side pre-fill of a new New Broadcast screen instance, no server call from this screen itself |
| [E] Variant B "Contact support" | Client-side `mailto:` link or navigation to an existing in-app support contact point — no new backend endpoint required for this screen specifically |
| [E] Variant B "Sign out" | Standard client-side sign-out (clears the stored JWT, returns to the Sign Up/Log In screen) — this is the one screen in the product where sign-out is offered as a primary-ish action rather than being tucked into Profile settings, since it's the honest, respectful thing to surface given the account's state |

**Loading state:** none needed for either variant — this screen only renders once the `cancelled` status and its distinguishing reason are already known from the triggering event.
**Error state:** not applicable — purely a display/informational screen in both variants, same as the Expired screen.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with every screen in this series; both variants fit comfortably without scroll on a standard reference viewport.
- **Accessibility:** per the same `aria-live="polite"` guidance applied to the Expired screen, Variant B in particular should announce its headline/explanation clearly to screen-reader users, since a customer may not be visually watching the screen at the exact moment an admin-triggered suspension arrives.
- **Session handling after Variant B:** since the account is suspended/banned, any subsequent state-changing request this session attempts (beyond the explicit Sign-out action) will be rejected by `middleware/requireActive.js` regardless of what this screen does — the client should treat that rejection gracefully if it occurs (e.g. the customer taps something before reading the Sign-out link) by redirecting to the Sign Up/Log In screen with a neutral "please sign in" state, not a confusing generic error toast.
- **PWA standalone mode:** Variant A carries no bottom tab bar (consistent with the rest of the broadcast→outcome journey); Variant B carries neither a tab bar nor a header back-action, being the most deliberately minimal, dead-end-adjacent screen in the entire product — appropriately so, given what it represents.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.8, 3.9A, 6.3) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). Variant B is the one screen in this entire build series that deliberately does not end in a product-recovery action — that's a direct, considered consequence of §3.8's own rule that a suspended/banned customer cannot log in or broadcast, not an inconsistency with the empty-state principles followed everywhere else.*
