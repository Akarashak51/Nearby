# NEARBY — Shop Dashboard: Staff Management
### Full Page Build Specification — Content, Layout, Visual Design, Copy, Motion & Data

**Version 1.0**
**Companion to:** NEARBY Master Build Document v2.6.2 (§3.5A, §4.1, §4.2, §5.3) · NEARBY Brand & PWA Guide v1.0 · NEARBY Page Architecture & Competitor Analysis (§2.10)
**Surface:** `/shop/*` — Teal-forward · **Role:** shop (owner-only for management; a restricted `staff` sub-role is proposed, see §2)

---

## 0. Scope & How to Use This Document

This document specifies Staff Management, listed as page 2.10 in the Page Architecture & Competitor Analysis document — where that document flags it explicitly:

> *"Add/remove staff who can toggle open/closed or answer requests — flagged as a possible future page, not in current FR list; include only if the team wants parity [with Zomato Partner's staff management]."*

**This is important context for how to use this document.** Unlike every other page specified so far in this series (Rush-Hour Toggle, Outlet Management, Opening Hours, Reliability & Ratings, Request History), Staff Management is **not buildable from the Master Build Document v2.6.2 as it stands today** — the `users` schema (§4.1) has a single flat `role` enum (customer/shop/admin) and `shops.ownerUserId` is explicitly `1:1` (§4.2), meaning the current data model has no concept of a shop having more than one associated user account at all.

This document is therefore written as a **complete, build-ready proposal**, not a spec over already-existing plumbing. Every new field, collection, endpoint, and permission rule below is explicitly marked **NEW**, and the rest of this series' pages (Outlet Management, Opening Hours, Rush-Hour Toggle, Reliability & Ratings) are referenced throughout to define exactly which of them a staff account can and cannot touch — since introducing a second user type on the shop surface has ripple effects across every page already specified. Nothing here contradicts the Master Build Document or Brand & PWA Guide; it extends both, following the same field-table and endpoint-table conventions those documents already use for additions.

Pattern-sourced from the Zomato Restaurant Partner app's "manage staff: add/delete/invite" screen (Page Architecture doc §2.10, pattern source).

---

## 1. Overview & Purpose

### 1.1 — What it is

A screen, visible **only to the verified owner of an approved shop**, for inviting additional people to help run the day-to-day of answering broadcasts and toggling the shop open/closed — without handing them the shop's own login credentials, and without giving them access to anything that could change the shop's identity, approval status, or financial/reliability standing.

### 1.2 — Why it exists

A single-person counter shop and a shop with three rotating staff members have very different needs from the same app. The Live Dashboard (page 2.4) assumes one person is watching it and answering Yes/No inside a short response window (default 120s, Master Build Document §3.2) — for a shop with staff covering shifts, that assumption breaks unless more than one device can legitimately act on the shop's behalf.

### 1.3 — The core design constraint: staff are narrow, not junior owners

The single most important product decision this page encodes is **what staff cannot do**, not what they can. A staff account exists to answer requests and flip the open/closed shutter — full stop. It must never be able to:
- Change the shop's approval-relevant facts (name, category, address, photo) — Outlet Management (page 2.6) stays owner-only.
- Change the weekly schedule — Opening Hours (page 2.7) stays owner-only, since a schedule mistake has consequences that outlast a single shift.
- Invite or remove other staff — only the owner can grow or shrink who has access.
- View or act on Rush-Hour (page 2.5b) — treated as an owner-level judgment call about the business's current capacity, not a per-shift decision.
- See anything on the customer or admin surfaces — a staff account's `role` is still `shop`-scoped exactly like an owner's, never broader.

### 1.4 — User story

> *"As a shop owner with two part-time staff, I want them to be able to answer NEARBY requests and mark the shop closed when they lock up for the night — without giving them my password, and without worrying they could accidentally change my shop's name or hours."*

### 1.5 — Pattern source

Inspired by: Zomato Restaurant Partner's staff management screen — add/delete/invite (Page Architecture doc §2.10, pattern source). NEARBY's version is deliberately narrower in scope than Zomato's, since Zomato Partner staff can touch order-level operational detail; NEARBY staff have exactly two permissions total (§1.3), reflecting how much smaller NEARBY's action surface is to begin with (Page Architecture doc §0, §5).

---

## 2. Functional Specification (Proposed — all items marked NEW)

### 2.1 — Data model additions (NEW)

**New collection: `shopStaff`**

| Field | Type | Notes |
|---|---|---|
| `_id` | ObjectId | Primary key — **NEW** |
| `shopId` | ObjectId → shops | Required — **NEW** |
| `userId` | ObjectId → users | Set once the invite is accepted; null while `status` is `invited` — **NEW** |
| `invitedEmail` | String | The email the invite was sent to, kept even after acceptance for audit/history — **NEW** |
| `status` | String enum: `invited`, `active`, `removed` | Default `invited` — **NEW** |
| `invitedByUserId` | ObjectId → users | Always the shop's `ownerUserId` at time of invite — **NEW** |
| `invitedAt` / `acceptedAt` / `removedAt` | Date | Timestamps for each lifecycle transition — **NEW** |

A join collection, not a field on `users` or `shops`, follows the same relational pattern the Master Build Document already uses for `shopResponses` (§4.4) — a many-to-many-shaped relationship (one shop, multiple staff; in principle one person could staff more than one shop) belongs in its own collection rather than an array field on either side.

**Change to `users` (§4.1) — NEW enum value only, no structural change**

`role` gains a fourth value: `staff` (alongside the existing `customer`, `shop`, `admin`). A `staff`-role user authenticates through the same login flow as a `shop`-role user (Master Build Document §7) but is resolved to a shop via their `shopStaff` entry rather than via `shops.ownerUserId`.

### 2.2 — Permission matrix (NEW — governs every other Shop Dashboard page)

| Page | Owner | Staff |
|---|---|---|
| Live Dashboard (2.4) — view, isOpen toggle | Full access | Full access |
| Incoming Request card (2.5) — answer Yes/No | Full access | Full access |
| Rush-Hour Toggle (2.5b) | Full access | **No access** — tab hidden entirely |
| Outlet Management (2.6) | Full access | **No access** — tab hidden entirely |
| Opening Hours (2.7) | Full access | **No access** — reachable only via Outlet Management, which is already hidden |
| Reliability & Ratings (2.8) | Full access | **Read-only** — staff can see the shop's score to understand how they're doing collectively, but no page here has any write action regardless, so this is a non-issue in practice |
| Request History (2.9) | Full access | Full access — read-only for everyone already, no elevated risk in staff seeing it |
| Staff Management (this page) | Full access | **No access** — tab hidden entirely |
| Help Centre (2.12) | Full access | Full access — a staff member should be able to raise a support ticket during their own shift |
| Leave/Closure scheduler (2.13) | Full access | **No access** — scheduling planned closures is an owner-level business decision |

This table is the actual deliverable of this entire feature — every other page in this series should be understood as implicitly gaining a `role === 'staff'` guard clause the day this page ships.

### 2.3 — API endpoints (NEW)

| Endpoint | Auth | Body / Notes |
|---|---|---|
| `GET /api/v1/shops/me/staff` | Bearer (shop owner only) | Returns all `shopStaff` entries for the caller's shop, `invited` and `active`, excluding `removed` by default |
| `POST /api/v1/shops/me/staff/invite` | Bearer (shop owner only) | `{ email: string }` — creates a `shopStaff` entry with `status: 'invited'`, sends an invite email (or, if the pilot has no email-sending service yet, a shareable invite link — see §9) |
| `DELETE /api/v1/shops/me/staff/:staffId` | Bearer (shop owner only) | Sets `status: 'removed'`; if the staff account has an active session, it should be invalidated on next request (JWT check against `shopStaff.status`, not just `users.status`) |
| `POST /api/v1/staff/invites/:token/accept` | Public (token-gated) | Consumes the invite token, creates or links a `users` record with `role: 'staff'`, sets the matching `shopStaff.status` to `active` |

Resolved from the caller's own JWT `sub` exactly like every owner-scoped endpoint elsewhere in the Master Build Document (§5.3 pattern) — a staff ID is always a path parameter validated against the *caller's own* shop's `shopStaff` list server-side, never trusted from the client as belonging to the right shop.

### 2.4 — Rate limiting (NEW, following existing §5.3 convention)

`POST /api/v1/shops/me/staff/invite` — max 10 per shop per day, generous for realistic staff turnover while guarding against invite-spam abuse of the email/link-sending mechanism.

### 2.5 — What this page explicitly does not do

- It does not define shift scheduling, time clocks, or per-staff performance tracking — it is purely an access-list manager, not a workforce-management tool.
- It does not let the owner set granular, per-staff permissions — the permission matrix in §2.2 is fixed and identical for every staff member; there is exactly one non-owner role, not a spectrum of roles, keeping this simple for a small-shop context.

---

## 3. Where It Lives

- **Entry point:** Shop Dashboard nav → visible **only when the logged-in user is the shop's owner** (`users.role === 'shop'` **and** `shops.ownerUserId === userId`) — a staff-role user never sees this tab at all, per §2.2.
- Labeled **"Staff"** in the nav, alongside Home, Outlet, Reliability, History.
- The invite-acceptance flow (§2.3's fourth endpoint) is reached via a link outside the authenticated app entirely — an invited person taps an emailed/shared link, lands on a lightweight acceptance screen (§4.6), and only enters the main Shop Dashboard nav after that.

---

## 4. Section-by-Section Content & Layout

### 4.1 — Page header & primary action

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Page header | "Staff" | H1 | Gray 900 |
| Subtitle | "People who can answer requests and open or close your shop for you." | Caption | Gray 600 — states the exact permission scope from §1.3 in plain words, right at the top |
| Primary action | "Invite staff" (button, top-right or below subtitle) | Button / SemiBold | Signal Teal fill, opens the invite sheet (§4.2) |

### 4.2 — Invite sheet

Bottom sheet, same mechanics as the Rush-Hour duration sheet and other sheets across this series (Brand Guide §9.3).

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Header | "Invite a staff member" | H1 | Gray 900 |
| Helper text | "They'll get a link to set up their own login. They'll be able to answer requests and open/close your shop — nothing else." | Body | Gray 600 — restates the permission boundary at the exact moment the owner is about to grant it |
| Email field | Label: "Their email" | Caption label / Body input | Required, validated as a well-formed email client-side before enabling Send |
| Primary action | "Send invite" (disabled until a valid email is entered) | Button / SemiBold | Full-width, Signal Teal fill |
| Secondary action | "Cancel" | Button / SemiBold, text-only | Gray 600 text |

### 4.3 — Staff list (active members)

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Section header | "Active" | H2 | Gray 900, shown only if at least one active staff entry exists |
| List row | Staff member's name (once accepted) or email (while still just invited elsewhere), plus a status pill | Body / Medium (name), Caption (email fallback) | Row layout matches the Request History row pattern (page 2.9) for visual consistency — full-width, 1px Gray 300 divider, no per-row card border |
| Status pill | "Active" | Caption / SemiBold pill | Signal Teal outline (not filled) — a lighter treatment than the "Selected" badge on Request History, since this is a steady-state label, not an outcome worth celebrating |
| Row action | "Remove" (text button, trailing edge) | Button / SemiBold | Danger #E5484D text — opens the confirmation in §4.5 |

### 4.4 — Pending invites section

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Section header | "Pending" | H2 | Gray 900, shown only if at least one invite hasn't been accepted yet |
| List row | Invited email address + "Invited {relative date}" | Body (email), Caption (invited-date) | Same row pattern as §4.3 |
| Status pill | "Pending" | Caption / SemiBold pill | Warning #FFB627 outline — distinguishes "waiting on them" from the settled "Active" state, without implying anything has gone wrong |
| Row actions | "Resend" · "Cancel invite" (both text buttons, trailing edge) | Button / SemiBold | "Resend" in Signal Teal text, "Cancel invite" in Danger text |

### 4.5 — Remove/cancel confirmation

A lightweight inline confirmation (not a full modal), matching the pattern established by the Rush-Hour Toggle spec's early-cancel confirmation (§4.5 of that document):

> **Copy (removing an active staff member):** *"Remove {name}? They'll lose access right away."* — two inline buttons: **"Yes, remove"** (Danger fill) and **"Keep them"** (text link).
>
> **Copy (cancelling a pending invite):** *"Cancel this invite? The link they were sent will stop working."* — two inline buttons: **"Yes, cancel"** (Danger fill) and **"Keep it pending"** (text link).

### 4.6 — Invite-acceptance screen (outside the authenticated app)

Reached via the emailed/shared invite link, before the invitee has any account at all:

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Header | "You've been invited to help run {Shop Name} on NEARBY" | H1 | Gray 900 |
| Subtitle | "Set up your login to start answering requests and managing this shop's open/closed status." | Body | Gray 600 |
| Form fields | Name, password (email is pre-filled from the invite, not editable) | Caption label / Body input | Matches the standard signup field set (Master Build Document §7), minus the phone field, since staff contact isn't customer-facing the way a shop's own phone number is |
| Primary action | "Create my login" | Button / SemiBold | Full-width, Signal Teal fill |

### 4.7 — Empty state (no staff invited yet)

> **Copy:** *"You're the only one who can answer requests right now. Invite someone if you'd like help covering your shop."*

A single static instance of the logomark's pin (Brand Guide §9.4 empty-state pattern), consistent with the same treatment used on the Reliability & Ratings view and Request History pages.

### 4.8 — Confirmation toasts

| Action | Copy |
|---|---|
| Invite sent | "Invite sent to {email}." |
| Invite resent | "Invite resent." |
| Invite cancelled | "Invite cancelled." |
| Staff removed | "{name} removed." |

---

## 5. Visual Design System Application

### 5.1 — Color mapping

| Token | Used for in this feature | Rationale |
|---|---|---|
| Signal Teal #00B8A9 | "Invite staff" primary button, "Send invite" button, "Active" status pill (outline), "Resend" text action | Shop surface's dominant accent (Brand Guide §3.3) — inviting and maintaining staff is routine shop management, same category of action as everything else on this teal-forward surface |
| Warning #FFB627 | "Pending" status pill (outline) | Signals "awaiting action from someone else" without implying anything has gone wrong — same connotation this token carries on the Rush-Hour Toggle's active state (that spec, §5.1) |
| Danger #E5484D | "Remove" / "Cancel invite" row actions, and their confirmation buttons | Both actions revoke access — a meaningful enough change to warrant the same color used for cancel/reject actions elsewhere in this series (consistent with the Rush-Hour Toggle's "Turn off" button, §5.1 of that document) |
| Gray 900 / Gray 600 | All body copy, names, email addresses, helper and subtitle text | Standard body-text accessibility rule (§3.4), applied consistently across this whole documentation series |
| Surface / White | Page and sheet backgrounds | Standard page/card tokens (§3.2) |

| Signal Teal #00B8A9 | Warning #FFB627 | Danger #E5484D | Gray 900 #14213D | Gray 600 #5B6472 |
|---|---|---|---|---|

Orange is not used anywhere on this page, for the same reason as every other Shop Dashboard spec in this series — reserved exclusively for Yes/No response buttons elsewhere on this surface (Brand Guide §3.3).

### 5.2 — Typography mapping

| Role (Brand Guide §4.2) | Size / line-height / weight | Used here for |
|---|---|---|
| H1 | 24px / 32px, 600 SemiBold | Page header ("Staff"), invite-sheet header, invite-acceptance header |
| H2 | 18px / 26px, 600 SemiBold | "Active" and "Pending" section headers |
| Body | 15px / 22px, 400 Regular (Medium for names) | Staff names/emails, helper and subtitle text, invite-acceptance form inputs |
| Caption | 13px / 18px, 400 Regular (SemiBold for status pills) | Field labels, invited-date stamps, status pills, subtitle text |
| Button | 15px / 20px, 600 SemiBold | "Invite staff", "Send invite", row actions (Remove/Resend/Cancel invite) |

### 5.3 — Spacing, grid & component styling

- List rows: identical row geometry to the Request History page (page 2.9) — 12px vertical / 16px horizontal padding, 1px Gray 300 bottom divider — deliberately reused rather than inventing a new list-row style for what is, structurally, the same kind of content (a scrollable list of items with a status pill and trailing actions).
- Status pills: outline style (not filled), fully rounded, 4px vertical / 10px horizontal padding — a lighter visual weight than the filled "Selected" badge on Request History, appropriate since these are steady-state labels rather than one-time outcomes.
- Invite sheet and invite-acceptance screen: standard form field rhythm (16px gap between fields, 6px between label and input), matching the Outlet Management page's Basic Info form exactly.
- Section headers ("Active" / "Pending"): 24px top margin, 12px bottom margin before the first row in that section.

### 5.4 — Iconography

- No new custom iconography required. Status pills use text only, no icon, keeping them visually calm and legible.
- The logomark pin (existing brand asset) appears once, statically, in the empty state (§4.7) — identical treatment to its use on the Reliability & Ratings and Request History pages.

---

## 6. Full Copy Deck

| State / Location | Exact copy |
|---|---|
| Page header | Staff |
| Page subtitle | People who can answer requests and open or close your shop for you. |
| Primary action | Invite staff |
| Invite sheet header | Invite a staff member |
| Invite sheet helper text | They'll get a link to set up their own login. They'll be able to answer requests and open/close your shop — nothing else. |
| Invite sheet field label | Their email |
| Invite sheet primary action | Send invite |
| Invite sheet secondary action | Cancel |
| Section header — active | Active |
| Section header — pending | Pending |
| Status pill — active | Active |
| Status pill — pending | Pending |
| Row action — remove | Remove |
| Row action — resend | Resend |
| Row action — cancel invite | Cancel invite |
| Remove confirmation prompt | Remove {name}? They'll lose access right away. |
| Remove confirmation — confirm | Yes, remove |
| Remove confirmation — dismiss | Keep them |
| Cancel-invite confirmation prompt | Cancel this invite? The link they were sent will stop working. |
| Cancel-invite confirmation — confirm | Yes, cancel |
| Cancel-invite confirmation — dismiss | Keep it pending |
| Invite-acceptance header | You've been invited to help run {Shop Name} on NEARBY |
| Invite-acceptance subtitle | Set up your login to start answering requests and managing this shop's open/closed status. |
| Invite-acceptance primary action | Create my login |
| Empty state | You're the only one who can answer requests right now. Invite someone if you'd like help covering your shop. |
| Toast — invite sent | Invite sent to {email}. |
| Toast — invite resent | Invite resent. |
| Toast — invite cancelled | Invite cancelled. |
| Toast — staff removed | {name} removed. |
| Error — invalid email | Enter a valid email address. |
| Error — already invited | This person already has an invite or is already on your staff. |
| Error — save/send failed | Couldn't send the invite. Check your connection and try again. |
| Error — rate limited | Too many invites sent today — try again tomorrow. |

---

## 7. Motion & Animation Specification

Every token below is reused from the existing Motion Token table (Brand Guide §9.1) — nothing new is introduced.

| Interaction | Token(s) used | Behavior |
|---|---|---|
| Invite sheet open/close | duration-standard (240ms), ease-out-soft | Standard bottom-sheet slide-up, identical mechanics to every other sheet in this series (Rush-Hour duration sheet, Address explainer sheet) |
| Send-invite button enable | duration-instant (100ms) | Opacity crossfade from disabled to enabled on valid email entry, no bounce (calm-tone rule, Brand Guide §1.4) |
| New row appears in list (after invite sent) | duration-quick (180ms), ease-out-soft | Row fades and slides in from the top of the Pending section — a small, satisfying confirmation that the action took effect, without a flashy or celebratory animation |
| Row removed from list (staff removed / invite cancelled) | duration-quick (180ms), ease-standard | Row fades out and collapses its height smoothly, remaining rows shift up to fill the gap |
| Inline confirmation appear (§4.5) | duration-instant (100ms) | Fade-in only, no shake — consistent with every other inline-confirmation pattern in this series |
| Confirmation toasts | duration-quick (180ms), ease-out-soft | Standard bottom-edge toast (§9.3) |
| Reduced motion | Single global override (§9.5) | All of the above collapse to instant state changes — no page-specific reduced-motion work required |

---

## 8. Imagery, Graphics, Icons & Video

No photography, illustration, or video anywhere on this page. It is an access-management utility screen, and per the shop-surface voice rule (Brand Guide §1.4), decorative imagery would only slow down a task an owner wants to complete quickly. The only illustrative asset is the existing logomark pin, used once, statically, in the empty state (§4.7, §5.4) — matching the same treatment already established on the Reliability & Ratings and Request History pages, for visual consistency across every "nothing here yet" moment in the Shop Dashboard.

No skeleton-loader design work beyond the standard shimmer pattern already defined in the Brand Guide (§9.4) — `GET /api/v1/shops/me/staff` (§2.3) is a single lightweight list call, reusing the existing shimmer treatment directly.

---

## 9. States & Edge Cases

| Case | Behavior |
|---|---|
| Owner invites an email that's already active staff or already pending | Inline error on the invite sheet: "This person already has an invite or is already on your staff." — checked client-side against the already-loaded list before submit, and re-validated server-side |
| Owner has no email-sending service available yet (pilot-stage constraint) | Fallback: "Send invite" instead generates a shareable link (copy-to-clipboard) the owner sends manually via WhatsApp/SMS themselves — the underlying `shopStaff` record and token behave identically either way; only the delivery mechanism differs, and this fallback should be treated as the default assumption for an early pilot rather than an edge case, given the Master Build Document's repeated free-tier/cost-conscious framing elsewhere (§3.11 pattern) |
| Invited person never accepts | Invite remains "Pending" indefinitely — no automatic expiry in v1, since a shop owner cancelling a stale invite manually (§4.4) is simpler to build than a background expiry job, consistent with this series' general preference for owner-driven control over background automation |
| Owner removes a staff member who is mid-response to an incoming request | Their session is invalidated on next request per §2.3, but a request they already tapped Yes/No on before removal is unaffected — `shopResponses` records the response permanently regardless of the responder's continued access |
| Staff member tries to reach a hidden page directly (e.g. by URL) | Server-side 403 on any owner-scoped endpoint call, exactly as the route-guard pattern already defined for non-admin users hitting `/admin/*` (Page Architecture doc §4) — the client-side hidden nav tab is a convenience, not the actual security boundary |
| Owner account itself is blocked (Master Build Document §3.5) | All staff sessions for that shop should be treated as invalid too, since a blocked shop has no legitimate reason for anyone — owner or staff — to be actively answering requests on its behalf |
| Two staff members are both viewing the Incoming Request card for the same request | First responder wins, exactly as the existing `shopResponses` model already handles a single response per shop per request (Master Build Document §4.4) — no new conflict-resolution logic is needed, since the shop (not the individual person) is the unit the platform already tracks |

---

## 10. Accessibility

- Status pills always carry a text label ("Active", "Pending") — color is reinforcement only, never the sole signal.
- Row actions (Remove, Resend, Cancel invite) are individually focusable buttons with clear accessible names, not icon-only controls with ambiguous purpose.
- The invite sheet's email field has an explicit, programmatically-associated label, and inline validation errors use `aria-describedby`/`aria-invalid`, matching the pattern established on the Outlet Management page.
- Minimum 44×44px touch target on every row action, pill, and button, consistent with every other page in this series.
- Confirmation prompts (§4.5) trap focus appropriately and return focus to the triggering row action on dismiss.
- Reduced motion: fully covered by the existing global override (§7) — no page-specific work required.

---

## 11. Admin Visibility & Data Management

- **Proposed addition (NEW, not built in this pass):** the All Shops directory (Page Architecture page 3.4) could show a shop's current staff count alongside its other profile details, since admin already has full visibility into everything else about a shop — flagged here as a natural follow-on, not built as part of this specification.
- No interaction with the Reports queue (page 3.6) in v1 — a shop owner managing their own staff list cannot affect another shop or a customer.
- **Security note for engineering:** because this page introduces a second, more-restricted user type on the shop surface, every existing owner-scoped endpoint (`PATCH /shops/me`, the Rush-Hour endpoint, etc.) must add an explicit check that the caller is the shop's *owner*, not merely *any* user with `role: 'shop'` linked to that shop — this is the single most important backend change this feature requires beyond the endpoints listed in §2.3, and should be treated as a required update to every existing shop-scoped route, not an optional hardening step.

---

## 12. Component Inventory — Developer Handoff

| Component | Type | States | Key tokens |
|---|---|---|---|
| StaffListSection | Composite (header + rows) | active-list, pending-list, empty | H2, Body, Caption, Signal Teal / Warning pills |
| StaffRow | List item | active, pending | Body, Caption, Danger text action |
| InviteStaffSheet | Bottom sheet | closed, open, valid-email, submitting, error | H1, Body, Button, duration-standard |
| RemoveConfirmInline | Inline confirm | hidden, visible (remove), visible (cancel-invite) | Danger, Button, duration-instant |
| InviteAcceptanceScreen | Standalone public form | loading, form, submitting, error | H1, Body, Button (reused signup field pattern) |
| StaffEmptyState | Composite (logomark + copy) | visible only when zero staff exist | Body, logomark pin (static) |
| Toast (reused) | Existing shared component | invite-sent, invite-resent, invite-cancelled, staff-removed, error | duration-quick, ease-out-soft |

Six page-specific composites plus one reused shared component. Every visual primitive (sheet, pill, inline confirm, list row) is drawn from patterns already established across the Rush-Hour Toggle, Outlet Management, Reliability & Ratings, and Request History specifications in this series — no new design-system primitive is introduced.

---

## Appendix A — Quick Reference: What's New vs. Reused

| New for this feature | Reused as-is from existing docs |
|---|---|
| `shopStaff` collection (§2.1) | Every color token (§3.2, Brand Guide) |
| `users.role` gains a `staff` enum value (§2.1) | Every type-scale role (§4.2, Brand Guide) |
| Four new API endpoints (§2.3) | Bottom-sheet, list-row, inline-confirm, and toast interaction patterns (§9.3, Brand Guide — reused directly from prior specs in this series) |
| Full permission matrix governing every other Shop Dashboard page (§2.2) — the actual core deliverable of this feature | Empty-state pattern using the static logomark pin (§9.4, Brand Guide) |
| Copy deck (§6 of this document) | Reduced-motion global override (§9.5, Brand Guide) |
| Security requirement: every existing owner-scoped endpoint must add an owner-vs-staff check (§11) | |

---

## Appendix B — Build Sequencing Note

Because this feature touches the authorization logic of every other Shop Dashboard page already specified in this series, it should not be scheduled as an isolated front-end task. The recommended build order is: (1) `shopStaff` collection and the four endpoints in §2.3, (2) the owner-vs-staff authorization check retrofitted onto every existing shop-scoped endpoint (§11), (3) the client-side nav guard hiding restricted tabs for `role: 'staff'` sessions, and only then (4) this page's own UI. Building the UI first without the backend authorization work in place would leave every other page's protection resting on a hidden nav tab alone — a convenience, not a security boundary, as noted in §9.

---

*— End of specification —*
