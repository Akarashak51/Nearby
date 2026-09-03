# NEARBY — Sign Up / Log In Page Build Specification
### Everything needed to build the auth screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.5A, 3.6, 3.7 password reset, 7), NEARBY Brand/PWA Guide v1.0, and current login/signup UX research (Google's own Sign-in-with-Google placement guidance, 2026 login-screen convention research)

---

## 0. Scope & what makes this page different from a typical app's auth screen

Three things from your own Master Build Document constrain this page and override generic "best practice" wherever they conflict:

1. **Only two auth methods exist: email/password (local) and Google Sign-In.** No phone/OTP, no Apple, no Facebook (Section 7). Don't add extra social buttons "for completeness" — every source reviewed on 2026 login conventions confirms 2–3 options is the ceiling before choice-paralysis sets in, and NEARBY only has two, which is already the simple end of that range.
2. **`role` is never a signup field.** Every account is created as `role: 'customer'` server-side, no exceptions (Section 3.5A — this closed a real privilege-escalation hole in v2.5). The one thing the client *can* send is an `accountType: 'customer' | 'shop'` flag, which the server maps to role internally. This page must never expose a raw "role" selector — only the customer/shop *intent* toggle described in Section 3 below.
3. **No OTP/SMS verification exists for the pilot** (Section 3.3 — phone reachability is confirmed by manual admin callback, not an automated OTP service). So this page must not imply "we'll text you a code" anywhere in its copy.

---

## 1. Page goals, in priority order

1. Get a new customer to a working account in the fewest fields possible — 81% of mobile signup abandonment happens after the form starts (per 2026 login-UX research reviewed), so every field here has to earn its place.
2. Surface **Google Sign-In prominently and equally with email/password** — for a consumer app (not B2B/SSO), research is consistent that social login should be a primary, above-the-fold CTA, not a secondary afterthought.
3. Let a prospective shop owner self-identify their intent (**"I want to open a shop"**) *without* exposing anything that looks like a role/permission picker — the account is still created as `customer`, the shop application happens afterward via `POST /shops` (page-architecture doc, item 2.1).
4. Never block on multi-step wizardry — this is a single-screen form with a Login/Sign-up toggle, not a multi-step flow (Section 3.7's forgot-password is the *only* other screen this page hands off to).

---

## 2. Layout overview

Single centered card, mobile-first, max width 400px, vertical stack. No split-screen hero-image layout (2026 login-screen trend research explicitly flags heavy background imagery as a common over-design mistake that slows load — directly conflicts with the Brand Guide's zero-latency-loading principle, Section 4.1/9.6).

```
┌─────────────────────────────┐
│ [A] Logomark + tagline       │
├─────────────────────────────┤
│ [B] Mode toggle (Log in/Sign up) │
├─────────────────────────────┤
│ [C] Google Sign-In button    │  ← above the fold, primary
├─────────────────────────────┤
│ [D] "or continue with email" divider │
├─────────────────────────────┤
│ [E] Email/password form      │
├─────────────────────────────┤
│ [F] Account-type toggle       │  ← Sign-up mode only
├─────────────────────────────┤
│ [G] Submit button             │
├─────────────────────────────┤
│ [H] Forgot password link      │  ← Log-in mode only
├─────────────────────────────┤
│ [I] Mode-switch footer text   │
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Logomark + tagline
**Layout:** centered, 64×64 logomark (`nearby-mark.svg`, Brand Guide §2.2) above the wordmark "NEARBY" (Display size, 32px/700 Bold — the *one* place on the whole app that Display size is justified, per Typography Scale's own note that Display is "rare — mobile-first layout").
**Copy:** wordmark + tagline directly below in Caption size, Text-secondary color:
> "Ask around. Instantly."

(Brand Guide §1.2 — primary tagline, not the shortened "Nearby. Right now." variant, since this screen has room and this is the tagline's proper home.)
**Background:** plain White — no photography, no gradient, no illustration behind the logomark (matches Brand Guide 9.6's explicit rejection of showcase-style imagery for the installed app, and directly avoids the "heavy background images slow initial load" mistake flagged in current login-screen research).

### [B] Mode toggle — Log in / Sign up
**Layout:** two-segment pill toggle directly below the tagline, 36px height, rounded-full, Gray 100 background track, active segment White background with a subtle 1px Border-gray outline, Text-primary label. Default state depends on entry point (deep link from "Get Started" → Sign up; default app open → Log in).
**Copy:** "Log in" · "Sign up"
**Animation:** active-segment background slides between positions over `duration-instant` (100ms), `ease-standard` — a single sliding-pill transition, not a hard swap, so it doesn't feel like two separate pages (research above stresses this page having "a unified experience" regardless of which path a user picks).

### [C] Google Sign-In button
**Placement rationale:** placed **above** the email/password form, per Google's own current developer guidance ("place it prominently alongside other sign-in methods") and 2026 consumer-app convention research (social login as primary CTA for consumer apps, email below) — this is a direct, deliberate choice, not an arbitrary ordering.
**Layout:** full-width button, 48px height, White background, 1px Border-gray outline (per Google's brand button guidelines — do not recolor this button with Brand Orange; following the provider's own button spec is itself a trust signal per the research reviewed).
**Copy:** "Continue with Google" (Google's standard button copy — not "Sign in with Google" only, since this single button serves both the Log-in and Sign-up toggle states without needing to change its own label).
**Icon:** official Google "G" logomark, left-aligned, 20×20, per Google's asset guidelines.
**Behavior:** obtains a Google ID token via Google Identity Services client-side; no client secret ever touches the browser (Master Build Doc §7). On success: server verifies signature/audience, auto-links to an existing `authProvider: 'local'` account by matching email, or creates a new `authProvider: 'google'`, `role: 'customer'` account — either way, the same NEARBY JWT is issued and the user lands on Home.
**Animation:** standard tap-scale-to-0.97 (`duration-instant`) only; a brief inline spinner replaces the "G" icon while the token round-trip is in flight (no full-page loading overlay — keep the rest of the form visible and inert).

### [D] Divider
**Copy:** "or continue with email"
**Layout:** horizontal rule + centered caption text, Text-secondary color, 24px vertical margin above/below. This exact divider pattern is called out in current research as "a universal convention" — deliberately not reinvented here.

### [E] Email / password form
**Fields (Log-in mode):** Email, Password (2 fields only).
**Fields (Sign-up mode):** Email, Password, Confirm Password (3 fields — kept to the minimum; name/phone are deliberately *not* collected at signup, since nothing in the FRs requires them before the customer's first broadcast, and every field cut here reduces the 81%-abandonment risk cited above).
**Layout:** stacked full-width inputs, 48px height, 12px radius, Surface white background, 1px Border-gray outline, Body-size (15px) label floats above on focus/fill (standard floating-label pattern — saves vertical space over separate static labels, keeps the form visually short).
**Password field:** includes a show/hide toggle (eye icon, Text-secondary) — flagged directly in the login-screen research reviewed ("view password" as a standard, expected affordance).
**Validation copy (inline, shown on blur/submit, never proactively before the user has typed anything):**
- Invalid email format → "Enter a valid email address"
- Password under 8 characters → "Password must be at least 8 characters"
- Confirm-password mismatch (sign-up only) → "Passwords don't match"
- Server-side duplicate-email on submit → "An account with this email already exists — log in instead?" (inline link that flips the mode toggle, doesn't dead-end the user)
**Color:** default border Border-gray; focus border Primary orange 2px; error state border Danger red (#E5484D) with Danger-colored helper text beneath the field.
**Animation:** border-color transition on focus/error, `duration-instant`, `ease-standard`. Error helper text fades+slides in (translateY 4px→0) over `duration-quick`.

### [F] Account-type toggle *(Sign-up mode only)*
**Purpose:** lets a prospective shop owner self-identify without exposing a role picker (Section 0.2 above) — maps client-side to the `accountType` field, never a `role` field.
**Layout:** two large tappable cards, side by side (stack vertically below ~340px width), each with an icon + one line of copy, radio-style selection (single-select, one always active — default "I'm looking for something").
**Copy:**
- Card 1: 🔍 "I'm looking for something" — subcopy: "Broadcast a search to nearby shops"
- Card 2: 🏪 "I run a shop nearby" — subcopy: "Get matched with customers looking for what you sell"
**Behavior:** selecting Card 2 sets `accountType: 'shop'` on the signup payload — server-side this still only ever produces `role: 'shop'` via the documented, restricted mapping (Section 3.5A), never a client-controlled role value. After successful signup, a Card-2 selection routes the user straight into the **Shop onboarding / Profile creation** screen (page-architecture doc, item 2.1) instead of Home, so the stated intent is honored immediately rather than left as a dead-end toggle.
**Color:** selected card gets a 2px Primary-orange border + Gray-100 fill; unselected stays White with 1px Border-gray.
**Animation:** border/fill transition on selection, `duration-instant`.

### [G] Submit button
**Copy:** "Log in" or "Create account" depending on mode.
**Layout:** full-width, 48px height, Primary orange (#FF5A36) background, White Button-weight text (600 SemiBold, 15px), 12px radius.
**States:** default → Primary; pressed → Primary-pressed (#E24521); disabled (empty required fields) → Gray 300 background, Text-secondary text, non-interactive; loading (request in flight) → button retains its color, label replaced by a small centered spinner, width unchanged (prevents layout shift).
**Animation:** tap-scale-to-0.97 on press (`duration-instant`); on successful submit, the whole card cross-fades into the app shell (`duration-standard`, `ease-out-soft`) rather than a hard route change — same pattern used for `request:selected` on the Home/Request screens (Brand Guide §9.2), reused here for consistency across every screen-to-screen handoff in the app.

### [H] Forgot password link *(Log-in mode only)*
**Copy:** "Forgot password?" — right-aligned, directly below the password field, Caption size, Text-secondary color (becomes Primary orange on hover/press only, not by default — keeps it visually secondary to the main submit action).
**Behavior:** opens a lightweight inline state (not a separate route) asking for email only → `POST /auth/forgot-password` → confirmation copy: **"If that email exists, we've sent a reset link."** (deliberately non-committal phrasing — never confirms or denies account existence, a standard security practice that also happens to match the brand's neutral, never-manufactures-urgency tone). Rate-limited server-side to 3 requests per email per hour (Master Build Doc §3.7/rate-limit table) — if triggered, same generic confirmation copy is shown regardless (never reveal the rate limit itself to the client).
**Not applicable to Google accounts:** if the email entered matches an `authProvider: 'google'` account, the confirmation copy is replaced with: **"This account signs in with Google — use the Continue with Google button above."** (Master Build Doc §3.7 — a Google-auth account has no password to reset; this must be handled gracefully, not silently ignored.)

### [I] Mode-switch footer
**Copy:** "Don't have an account? **Sign up**" (shown in Log-in mode) / "Already have an account? **Log in**" (shown in Sign-up mode) — centered, Body text with the action word in Primary orange, tap flips the [B] toggle.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where on this page |
|---|---|---|
| Primary | `#FF5A36` | Submit button, focused input border, active account-type card border, mode-switch action word |
| Primary-pressed | `#E24521` | Submit button pressed state |
| Danger | `#E5484D` | Validation error borders/text |
| Text primary | `#14213D` | Wordmark, field labels, form copy |
| Text secondary | `#5B6472` | Tagline, forgot-password link, placeholder text, unselected divider text |
| Border | `#D8DCE3` | Input outlines, Google button outline, unselected account-type card border |
| Surface | `#F4F5F7` / `#FFFFFF` | Page background (Gray 100) / card, input, Google-button backgrounds (White) |

**No Success/Warning tokens used on this page** — those are reserved for live request states elsewhere in the app; introducing them here would blur their meaning.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| "NEARBY" wordmark | Display / 32px | 700 Bold |
| Tagline | Caption / 13px | 400 Regular |
| Mode toggle labels | Body / 15px | 400 Regular (active segment 600 SemiBold) |
| Google button label | Button / 15px | 600 SemiBold |
| Divider text | Caption / 13px | 400 Regular |
| Form field labels/input text | Body / 15px | 400 Regular |
| Account-type card title | H2 / 18px | 600 SemiBold |
| Account-type card subcopy | Caption / 13px | 400 Regular |
| Submit button label | Button / 15px | 600 SemiBold |
| Forgot-password / mode-switch text | Caption / 13px | 400 Regular |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Mode-toggle pill slide, input focus/error border color, all tap-scale feedback |
| `duration-quick` | 180ms | Inline validation error fade+slide-in |
| `duration-standard` | 240ms | Card cross-fade into app shell on successful auth |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Focus/toggle transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Post-auth screen handoff |

**Explicitly no motion on:** logomark/tagline, Google button icon (beyond the loading-state spinner), account-type card selection beyond the border/fill transition. **Reduced motion:** covered automatically by the app-wide `prefers-reduced-motion` query (Brand Guide §9.5) — no page-specific override needed.

---

## 7. Images, icons & video — asset list

**No video, no hero photography, no background imagery** — directly required by Brand Guide §9.6 (no showcase motion for the installed app) and by current login-screen research flagging heavy background images as a load-time anti-pattern.

| Asset | Format/size | Source |
|---|---|---|
| Logomark | SVG, 64×64 render of `nearby-mark.svg` | Already delivered, Brand Guide §7.1 |
| Google "G" icon | Official Google-provided asset, 20×20 | Google Identity Services branding kit — do not recreate/restyle this icon |
| Eye / eye-slash icon (password show/hide) | Line icon, 18×18, Text-secondary | Icon library (same one used app-wide) |
| Account-type card icons | 🔍 and 🏪 emoji, or matching line icons if the team wants a fully custom feel | Emoji recommended for v1 — zero-asset-cost, consistent with the Home page spec's category-chip decision |
| Error/success inline icons (optional, small check/alert glyph next to helper text) | Line icon, 14×14 | Optional polish, not required for v1 |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [C] Google Sign-In | Client: Google Identity Services SDK → ID token. Server: token verify → `POST /api/v1/auth/google` (or equivalent) → JWT |
| [E]/[G] Email/password submit | `POST /api/v1/auth/signup` (mode: sign-up, body: `{ email, password, accountType }`) or `POST /api/v1/auth/login` (mode: log-in, body: `{ email, password }`) → JWT on success |
| [F] Account-type toggle | Client-side state only until submit; travels as `accountType` in the signup payload, never a `role` field (Section 0.2) |
| [H] Forgot password | `POST /api/v1/auth/forgot-password` → generic confirmation copy regardless of outcome (Section 3[H] above) |
| Post-auth routing | JWT payload's `role` (Master Build Doc §3.6) determines landing: `customer` + no `accountType: 'shop'` intent → Home; `accountType: 'shop'` just submitted → Shop onboarding screen; existing `shop`/`admin` role on login → their respective dashboards |

**Loading state:** submit button shows inline spinner (Section [G]) — no full-page overlay, no skeletons needed on this page since there's no list/card data to placeholder.
**Error state:** server errors (network failure, 500, etc.) surface via the shared toast/error-boundary pattern (Master Build Doc §6.1) rather than a bespoke error UI on this page — keeps this screen's error handling consistent with every other screen in the app.

---

## 9. Responsive & platform notes

- **Viewport:** centered card, max-width 400px, vertical-fill on mobile (360–428px reference), same centered treatment on tablet/desktop — no split-screen/hero-image layout at any breakpoint (Section 2 rationale applies at every width).
- **Keyboard handling:** on mobile, the card should scroll to keep the focused input above the on-screen keyboard — standard behavior, but worth stating explicitly since a 3-field sign-up form plus the account-type cards can otherwise push the submit button below the fold on small devices.
- **Autofill:** email/password fields should use standard `autocomplete` attributes (`email`, `current-password` / `new-password`) so browser/OS password managers and Google Smart Lock work normally — this materially reduces the field-typing friction that drives the 81% abandonment figure cited in Section 1.
- **PWA standalone mode:** this screen has no top app-bar chrome (unlike Home) since a signed-out user hasn't entered the app shell yet — the logomark in Section [A] is the only branding present, which is why its size and prominence matter more here than anywhere else in the product.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.5A, 3.6, 3.7, 7) and NEARBY Brand/PWA Guide v1.0 (Sections 1–5, 9). No color, motion token, field, or auth behavior introduced here contradicts either source document — in particular, no role-selection field of any kind appears anywhere on this page, per the explicit v2.5→v2.6 security fix in Section 3.5A.*
