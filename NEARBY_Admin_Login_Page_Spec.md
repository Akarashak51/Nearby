# NEARBY — Admin Console: Admin Login
### Condensed Page Spec — v1.0
**Refs:** Master Build Doc v2.6.2 (§3.5A, §5.1) · Brand & PWA Guide v1.0 · Page Architecture (§3.1)
**Surface:** `/admin/*` — Navy-forward · **Role:** admin only

---

## 1. Overview
Separate, unlinked login for `role: 'admin'` only — never reachable from customer/shop nav (§3.5A). **No public signup**: the sole admin account is created via a one-time `scripts/createAdmin.js` seed script run manually against prod (§3.5A) — this page has no "Sign up" link, no "Forgot password via self-serve reset flow" tie-in beyond what's explicitly built, and no social login. It's the one login screen in the app that must look and feel deliberately unadorned/serious — no broadcast-arc motifs, no marketing tone.

## 2. Functional Spec
- `POST /api/v1/auth/login` (existing, same endpoint as customer/shop) — `{ email, password }` → JWT with `role: 'admin'`.
- Client checks returned `role`; if not `admin`, reject client-side even if credentials were valid (defense in depth on top of server-side `requireAdmin.js` route gating, §3.5A) — copy: generic invalid-credentials message, **never** "this account isn't an admin" (avoids leaking role info to a prober).
- Rate limit: standard auth rate limit already defined for login elsewhere (§5.3 convention) — no relaxation for this surface.
- No "remember me" — admin sessions should be short-lived by convention given elevated privileges (JWT `exp`, §3.5A already defines expiry).

## 3. Where It Lives
Standalone route `/admin/login`, no nav chrome around it (no header, no footer links to other surfaces) — reinforces that this is a separate application area (§3.5A: "structurally separate").

## 4. Section-by-Section

| Element | Copy | Type role | Notes |
|---|---|---|---|
| Logo | Static logomark, small, top | — | No animation (Brand Guide §9.2 — motion reserved for customer-facing moments) |
| Header | "Admin sign in" | H1 | Ink Navy #14213D |
| Email field | Label: "Email" | Caption/Body | — |
| Password field | Label: "Password" | Caption/Body | Standard show/hide toggle |
| Primary action | "Sign in" | Button/SemiBold | Ink Navy fill (not Orange — Orange reserved for destructive actions only on this surface, Brand Guide §3.3) |
| Error (generic) | "Incorrect email or password." | Caption, Danger #E5484D | Same message for wrong password AND non-admin role (see §2) |
| Rate-limit error | "Too many attempts — try again in a few minutes." | Caption, Danger | — |

No "forgot password" link is specified here as a deliberate omission unless the team confirms an admin-specific reset path exists — flagged rather than assumed, since self-serve reset for a single high-privilege account is a security decision, not a UI one.

## 5. Visual Design
- **Colors:** Ink Navy #14213D (primary/header/button), Gray 900 body text, Gray 300 borders, Danger #E5484D errors only. **No Orange, No Teal** — this screen predates role resolution, so no surface-accent color applies yet; Navy is used as the admin surface's own identity color (Brand Guide §3.3).
- **Background:** Plain white or a very light Gray 100 field, centered card, no imagery, no gradient.
- **Card:** 16px radius, 1px Gray 300 border, generous 32px padding (more than shop/customer forms — this screen should feel deliberate, not quick).
- **Typography:** H1 24px/600, Body 15px/400, Caption 13px/400 — same type scale as rest of app (§4.2), no special admin type treatment needed.

## 6. Graphics / Images / Video
None. No illustration, no photo, no video. A static small logomark only — this is the one screen in the entire product where restraint itself is the correct design choice.

## 7. Animation
- Field focus: instant border-color change, no transition needed beyond default browser/CSS (duration-instant, 100ms, if any).
- Error appear: fade-in only (100ms), no shake — consistent with calm-tone rule (§1.4) applied even here.
- Button press: standard tap feedback (existing pattern), nothing custom.
- **No loading illustrations, no logomark animation** — reduced-motion default in spirit even without the media query, since this page has no celebratory or brand moment to earn motion.

## 8. States & Edge Cases

| Case | Behavior |
|---|---|
| Valid admin creds | JWT issued, redirect to Admin Dashboard/Overview (page 3.2) |
| Valid creds, non-admin role | Generic invalid-credentials error (§2, §4) |
| Wrong password | Same generic error |
| Rate-limited | Rate-limit copy shown (§4) |
| JWT expires mid-session elsewhere in admin console | Redirect back here, no silent refresh (matches short-lived-session posture, §2) |

## 9. Accessibility
- Explicit `<label>` on both fields, `aria-invalid` + `aria-describedby` on error, 44×44px min touch targets, focus-visible outline in Ink Navy.

## 10. Admin/Data Management
N/A — this page precedes all admin data access; nothing here writes beyond the login attempt itself (rate-limit counters, existing mechanism).

## 11. Component Inventory
| Component | States |
|---|---|
| AdminLoginForm | idle, submitting, error, rate-limited |
| Toast (reused) | none needed — errors are inline only |

---
*— End of condensed spec —*
