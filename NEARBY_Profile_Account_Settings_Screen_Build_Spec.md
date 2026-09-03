# NEARBY — Profile / Account Settings Screen Build Specification
### Everything needed to build the customer's account-management screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.3, 3.5A, 3.6, 3.7, endpoint table §5), NEARBY Brand/PWA Guide v1.0, and account-settings/profile-page UX pattern research (Eleken's profile-page pattern survey, Android's Settings design guidance, uxpatterns.dev's Account Settings pattern)

---

## 0. Scope — the largest gap surfaced in this entire build series, named plainly

Every previous screen in this series that hit a documented gap (Request History's missing list endpoint, the missing push-unsubscribe endpoint, the undefined report-reason list) was a **partial** gap — one missing piece within an otherwise well-specified flow. This screen is different: **there is no endpoint anywhere in your Master Build Document's authoritative table for a customer to edit their own profile.** `GET /api/v1/auth/me` exists (reads the current session's user record), but there is no `PATCH` counterpart — no way to change a name, a phone number, or a password while logged in, and no account-deletion mechanism of any kind.

This document proposes the minimal, pattern-consistent set of additions needed to make a genuinely functional Profile screen, rather than either inventing a large new subsystem or silently shipping a screen that's mostly read-only and quietly disappointing:

1. **Proposed: `PATCH /api/v1/auth/me`** — Bearer (any role), body `{ name?, phone? }` — edits the caller's own basic profile fields, resolved from the JWT `sub` claim (never a client-supplied id, consistent with every other "me" endpoint's ownership pattern in this API). Deliberately excludes `email` (changing a login identifier safely usually needs its own re-verification flow, out of scope for a pilot) and `password` (see #2).
2. **No new endpoint needed for password changes.** A customer with `authProvider: 'local'` who wants to change their password while logged in can simply be routed through the **existing** `POST /auth/forgot-password` → `POST /auth/reset-password` flow (§3.7) — the same flow already built for the Sign-Up/Log-In screen's "Forgot password?" link. This is the honest, zero-new-infrastructure answer: your doc already has a complete, working password-change mechanism, it's just gated behind "forgot," and there's no reason a logged-in customer can't use the identical path.
3. **No account-deletion endpoint proposed.** Unlike #1 and #2, this isn't a small, obviously-safe addition — account deletion touches every collection this customer has ever created data in (requests, feedback, reports, aiConversations) and deserves its own deliberate design conversation, not a bolt-on here. This document treats it the same way the Cancelled and Push-Settings screens treated their own genuine dead ends: an honest **"Contact support to delete your account"** link, not a fabricated in-app deletion flow.

---

## 1. Page goals, in priority order

1. Establish identity clearly at the top — name, email, how they signed in — per the near-universal profile-page pattern (avatar/name/email for quick identification, confirmed across every source reviewed).
2. Group settings into a small number of clearly-labeled sections, list-style, not a wall of controls — Android's own Settings design guidance and the broader research agree: grouped lists with clear labels beat a dense single form.
3. Put sign-out at the bottom, where research says users expect it — a small but real convention worth following exactly rather than inventing a different placement.
4. Handle every genuine limitation (no password change without leaving the app's normal flow, no in-app account deletion) honestly, the same way this whole build series has handled every other real constraint — state it plainly rather than paper over it.

---

## 2. Layout overview

Single scrollable screen, reached via the bottom tab bar's "Profile" tab (Home page spec, Section [I]).

```
┌─────────────────────────────┐
│ [A] Header (title only)       │
├─────────────────────────────┤
│ [B] Identity card              │
├─────────────────────────────┤
│ [C] Account section (list)     │
├─────────────────────────────┤
│ [D] Notifications section      │
├─────────────────────────────┤
│ [E] Become a shop card         │  ← conditional (role: customer only)
├─────────────────────────────┤
│ [F] Support section             │
├─────────────────────────────┤
│ [G] Sign out                     │
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, no back-chevron (tab-bar destination), left-aligned title.
**Copy:** "Profile" — chosen over "Account" or "Settings" per the naming research reviewed (Booking.com/Airbnb-style convention: "Profile" as the umbrella entry point once logged in, avoiding the confusing "Account" label that research flags as commonly mistaken for a financial/bank account in other contexts).
**Background:** White, 1px Border-gray bottom border on scroll.
**Animation:** none.

### [B] Identity card
**Purpose:** the quick-identification block every profile-page pattern surveyed leads with.
**Layout:** card, Surface white background, no outline needed here (sits directly below the header as the page's visual anchor rather than a bordered card — a subtle distinction from every other card in this product, appropriate since this is effectively the page's header content, not a discrete unit among others).
**Content:**
- Avatar: 64×64 circle — the customer's initial (first letter of `name`) centered in a Gray-100-filled circle, Text-secondary text (no photo-upload feature proposed for v1 — nothing in the source document supports one, and adding avatar-photo storage would mean a fourth proposed addition beyond what Section 0 already covers; keeping this to initials only is the honest minimal-footprint choice).
- Name (H2, 600 SemiBold, Text-primary), directly right of or below the avatar depending on viewport width.
- Email (Body, Text-secondary) directly below the name.
- A small pill badge indicating sign-in method: "Signed in with Google" (with the Google "G" icon, 14×14) or "Local account" (no icon) — read directly from `authProvider` (§4.2's schema), and this single field is what determines whether [C]'s password-related row appears at all (Section [C] below).
**Animation:** none.

### [C] Account section
**Section label:** "Account" (H2, Text-secondary, small-caps or simply Caption-weight — a section-header treatment distinct from card content, per the settings-pattern research's emphasis on clear section grouping).
**Layout:** a plain list of rows (no card border around the whole group — Android's own Settings pattern explicitly favors list rows over boxed cards for this kind of grouped, low-visual-weight content), each row 48px minimum height, 1px Border-gray divider between rows, chevron (›) on the right for rows that navigate somewhere.

**Row 1 — "Edit name":** tapping opens a small inline edit (either an expand-in-place text field with Save/Cancel, or a lightweight bottom-sheet — either works; a bottom-sheet is slightly more consistent with this product's existing sheet-based pattern for the New Broadcast screen's manual-address expansion) → `PATCH /api/v1/auth/me` with `{ name }` on save.

**Row 2 — "Edit phone":** same treatment as Row 1, `PATCH /api/v1/auth/me` with `{ phone }`. Per §4.2's schema note ("phone required for shop/admin; optional for customer if using Google auth"), a customer who signed up via Google with no phone on file sees this row read "Add phone" instead of "Edit phone" — same underlying action, worded to match whether a value already exists.

**Row 3 — "Change password"** *(shown only when `authProvider === 'local'` — omitted entirely for a Google-authenticated account, per Section [B]'s badge already establishing that distinction)*: tapping this **navigates the customer through the existing forgot-password flow** (§3.7) rather than a bespoke "enter current password, enter new password" form — the row's supporting text makes this explicit rather than surprising: "We'll send a reset link to your email" (Caption, Text-secondary, shown beneath the row label, per the settings-pattern guidance to show what a setting *does* rather than leaving it unexplained). Tapping triggers `POST /auth/forgot-password` with the customer's own already-known email (no re-entry needed, since they're already authenticated) and shows the same generic confirmation copy already established on the Sign-Up screen: "If that email exists, we've sent a reset link."

**Row 4 — "Delete account"** *(styled distinctly — see color notes below)*: tapping opens a plain informational sheet, not a deletion flow: "To delete your account, contact our support team at **[support contact]** — we'll take care of it from there." with a single "Contact support" button (`mailto:` link, or an in-app support channel if one exists elsewhere in the product) — the second deliberate "no in-app action, honest redirect" case in this build series after the Push-Settings screen's `denied`-permission state, for the reason given in Section 0.

**Animation:** row-tap background highlight (brief, `duration-instant`, standard list-row press feedback); inline edit / bottom-sheet expansions use `duration-quick` (180ms, `ease-out-soft`), consistent with every other sheet/expand pattern in this series.

### [D] Notifications section
**Section label:** "Notifications"
**Layout:** single row, "Push notifications" label + a right-aligned status text reading the current state ("On" / "Off" / "Blocked," Success teal / Text-secondary / Text-secondary respectively) + chevron.
**Behavior:** tapping navigates directly to the already-built **Push Notification Settings screen** — this row is a pure entry point, not a duplicate of that screen's own controls, avoiding building the same toggle logic twice.
**Animation:** same row-tap feedback as Section [C].

### [E] Become a shop card *(shown only when the customer's `role` is still `'customer'` — disappears once they've promoted to `'shop'`, per §3.3's promotion-on-`POST /shops`-success rule)*
**Purpose:** a second entry point to the same growth surface already established on the Home page (Home spec, Section [H]) — placing it here too follows the "don't make the customer hunt for it" principle, since Profile is a natural place to look for "upgrade my account"-type actions.
**Layout:** card, Ink Navy background (#14213D), White text — **identical treatment to the Home page's own Become-a-shop card**, reused rather than re-designed, so the two entry points feel like the same offer, not two different features.
**Copy:** identical to the Home page spec's card copy — "Own a shop nearby? **Join NEARBY** and start getting matched with customers looking for what you sell." + "Get Started" button.
**Animation:** none, matching the Home card's own static treatment.

### [F] Support section
**Section label:** "Support"
**Layout:** simple list rows, same treatment as [C].
**Row 1 — "Contact support":** opens the same support channel referenced in [C]'s account-deletion sheet — one consistent contact method used everywhere this product needs an honest human-handoff, rather than several different "contact us" paths scattered across different screens.
**Row 2 — "About NEARBY"** *(optional, nice-to-have)*: a simple static screen or sheet with a one-paragraph description of what NEARBY is and does — not specified in detail here since it's pure static content with no data dependency, but included as a natural, low-effort addition to a Support section per the profile-page research's general pattern of including a "Help and Support" grouping.
**Animation:** same row-tap feedback as [C].

### [G] Sign out
**Layout:** full-width, sits at the very bottom of the scrollable content (not fixed — per the research's explicit finding that users expect sign-out at the natural bottom of a profile page, not pinned above the fold or floating), styled as a plain text-only row, Danger red text, centered, no icon needed, generous vertical padding (24px) separating it visually from the Support section above.
**Copy:** "Sign out"
**Confirmation:** a lightweight inline dialog — "Sign out of NEARBY?" with **Cancel** / **Sign out** actions (Cancel visually primary/default-focused, same convention already established for the Live-Count screen's cancel-confirmation and the Push-Settings "Turn off" action) — a small deliberate friction point since this ends the session, though it's a fully reversible action (the customer can just log back in), so this is a lighter-weight confirmation than, say, the Shortlist screen's shop-selection confirmation.
**Behavior:** clears the stored JWT client-side, returns to the Sign Up / Log In screen.
**Color:** the **one** deliberate use of Danger red on an otherwise entirely calm, procedural screen — sign-out is genuinely the most consequential single action available here (it ends the session), and giving it the product's one "this matters, pay attention" color, while every settings row above it stays neutral, mirrors exactly how this color is used everywhere else in this series: reserved for real stakes, never decoration.
**Animation:** tap-scale-to-0.97; confirmation dialog fades+scales in (`duration-quick`, `ease-standard`); on confirm, standard screen-exit cross-fade (`duration-standard`, `ease-out-soft`) to the Sign Up/Log In screen.

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where |
|---|---|---|
| Primary | `#FF5A36` | "Get Started" button (Become-a-shop card), any inline "Save" action in the name/phone edit sheets |
| Success | `#00B8A9` | "On" status text (Notifications row) |
| Danger | `#E5484D` | "Sign out" row and its confirmation action, "Delete account" row label (the one other row on this screen worth visually flagging as consequential, alongside sign-out) |
| Text primary | `#14213D` | Header title, name, section content text |
| Text secondary | `#5B6472` | Email, avatar initials, section labels, row supporting text, "Off"/"Blocked" status text |
| Border | `#D8DCE3` | Row dividers, header scroll-divider |
| Surface | `#F4F5F7` / `#FFFFFF` | Avatar background (Gray 100) / page background, cards (White) |
| Ink Navy | `#14213D` | Become-a-shop card background (reused from Home) |

**Two rows use Danger red, deliberately:** "Delete account" and "Sign out" — both are the screen's genuinely consequential actions (one ends a session, the other starts an irreversible process), and giving both the same visual weight, while every other row on the screen stays neutral, is the clearest way to communicate "these two are different from the rest" without needing extra copy to say so.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Name (identity card) | H2 / 18px | 600 SemiBold |
| Email / auth-method badge | Caption / 13px | 400 Regular |
| Section labels ("Account," "Notifications," "Support") | Caption / 13px | 600 SemiBold (bumped weight to read as a header despite Caption size, per the settings-pattern convention of small-but-bold section labels) |
| Row labels | Body / 15px | 400 Regular |
| Row supporting text (e.g. "We'll send a reset link to your email") | Caption / 13px | 400 Regular |
| Become-a-shop card copy | Body / 15px + Button / 15px | 400 Regular / 600 SemiBold |
| Sign-out label | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Row-tap press feedback, tap-scale on buttons |
| `duration-quick` | 180ms | Inline-edit/bottom-sheet expansions, confirmation-dialog fade-ins |
| `duration-standard` | 240ms | Screen-exit cross-fade on sign-out |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Confirmation-dialog transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Sheet/expand transitions, screen-exit |

**No looping animation anywhere on this screen** — entirely calm, procedural settings content, consistent with every non-live screen in this series.
**Reduced motion:** covered by the app-wide `prefers-reduced-motion` query, same degradation pattern as every other screen.

---

## 7. Images, icons & video — asset list

**No video, no photography.**

| Asset | Format/size | Source |
|---|---|---|
| Avatar (initials) | CSS-drawn circle, 64×64, Gray-100 fill | — |
| Google "G" icon (auth-method badge) | Official Google asset, 14×14 | Reused from the Sign-Up screen's identical use |
| Chevron (row navigation) | Line icon, 16×16, Text-secondary | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B] Identity card | `GET /api/v1/auth/me` (already documented) — `name`, `email`, `authProvider` |
| [C] Row 1/2 edits | Proposed `PATCH /api/v1/auth/me` with `{ name? }` or `{ phone? }` (Section 0) |
| [C] Row 3 (Change password) | Existing `POST /api/v1/auth/forgot-password` (§3.7), triggered with the already-known session email |
| [C] Row 4 (Delete account) | No endpoint — client-side navigation to a support contact method only |
| [D] Notifications row | Reads `Notification.permission` client-side (same source of truth as the Push Settings screen) to render the status text; tapping navigates to that existing screen |
| [E] Become a shop | Client-side navigation to the shop-onboarding flow (`POST /shops`, §3.3), identical entry point to the Home page's own card |
| [F] Support | Client-side `mailto:`/support-channel navigation only |
| [G] Sign out | Client-side JWT clear + navigation, no endpoint |

**Loading state:** the identity card shows a brief skeleton (gray placeholder blocks for avatar/name/email) while `GET /auth/me` resolves on first mount — everything else on this screen (section labels, row structure) renders immediately since it doesn't depend on fetched data.
**Error state:** a failed `PATCH /auth/me` call uses the shared toast/error-boundary pattern (§6.1) with the edit sheet remaining open and the entered value preserved for retry, rather than silently closing and losing the customer's edit.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with every screen in this series.
- **PWA standalone mode:** carries the bottom tab bar (permanent destination, per Section 2), consistent with Home, History, and the AI Assistant screens.
- **Endpoint dependency:** flagged clearly per Section 0 — this screen's editing capability (Rows 1–2 of Section [C]) depends entirely on the proposed `PATCH /auth/me` endpoint existing; without it, this screen would need to ship as fully read-only for name/phone, which is a materially worse experience worth raising with the team before this sprint starts, alongside the other gaps already surfaced in this build series (Request History's list endpoint, the push-unsubscribe endpoint).

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.3, 3.5A, 3.6, 3.7, and the master endpoint table in Section 5) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). This is the largest gap identified across this entire build series — no customer profile-editing endpoint exists at all — and the proposed fix is deliberately the smallest one that makes the screen functional: one new `PATCH` endpoint for name/phone, zero new endpoints for password changes (reusing the existing forgot-password flow), and an honest human-handoff rather than a fabricated flow for account deletion.*
