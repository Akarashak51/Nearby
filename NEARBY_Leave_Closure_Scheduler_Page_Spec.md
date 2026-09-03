# NEARBY — Shop Dashboard: Leave / Temporary Closure Scheduler
### Full Page Build Specification — Content, Layout, Visual Design, Copy, Motion & Data

**Version 1.0**
**Companion to:** NEARBY Master Build Document v2.6.2 (§3.1, §3.3, §3.4, §4.2, §5.3) · NEARBY Brand & PWA Guide v1.0 · NEARBY Page Architecture & Competitor Analysis (§2.13)
**Surface:** `/shop/*` — Teal-forward · **Role:** shop

---

## 0. Scope & How to Use This Document

This document specifies the Leave / Temporary Closure Scheduler, listed as page 2.13 in the Page Architecture & Competitor Analysis document — "mark shop closed for a date range (festival, personal) without losing `approved` status," pattern-sourced from the Zomato Restaurant Partner app's "Schedule leaves in advance."

This is the third and last of the Shop Dashboard's three time-based availability controls, alongside the Rush-Hour Toggle (page 2.5b, this series) and Opening Hours (page 2.7, this series) — and this document leans on both of those specifications throughout, since a shop owner will naturally compare the three and expect them to feel like one coherent family of controls, not three unrelated features that happen to sit near each other in the nav.

Like the Rush-Hour Toggle, this page requires a small, explicitly-marked data model addition, since nothing in the Master Build Document's current `shops` schema (§4.2) represents a *future-dated, planned* closure — `isOpen` is a live, immediate switch (§3.4), not a scheduler. Every new field and the one new derived eligibility rule below are marked **NEW**.

Nothing in this document contradicts the Master Build Document v2.6.2 or the Brand & PWA Guide v1.0.

---

## 1. Overview & Purpose

### 1.1 — What it is

A screen for a shop owner to plan ahead: "I'll be closed from the 14th to the 16th for a festival" or "I'm taking next Tuesday off" — set once, in advance, rather than remembered and manually toggled on the day itself. It removes the shop from broadcast matching for the scheduled date range automatically, and automatically restores it afterward, with no day-of action required from the owner.

### 1.2 — How this differs from Rush-Hour and from the manual isOpen switch

This is the third distinct "the shop is temporarily unavailable" mechanism on the Shop Dashboard, and the three exist for three genuinely different situations — this distinction is the single most important thing this page's own copy and empty states must make clear, or a shop owner could reasonably wonder "wait, didn't I already have a way to do this?":

| Mechanism | Timescale | Set when | Use case |
|---|---|---|---|
| `isOpen` manual switch (Live Dashboard, page 2.4) | Right now, indefinite until switched back | In the moment | "I'm locking up early today" |
| Rush-Hour Toggle (page 2.5b) | Minutes to a few hours (30 min–4 hr cap) | In the moment, while busy | "I'm slammed right now, pause new requests" |
| **Leave / Closure Scheduler (this page)** | Full days, planned ahead of time | Days or weeks in advance | "I'll be shut for Diwali, the 12th through the 14th" |

### 1.3 — Why this needs its own mechanism rather than just "remember to toggle isOpen on the day"

The entire value of this feature is removing a task from the owner's memory — Zomato Partner's own framing ("schedule leaves in **advance**," Page Architecture doc §2.13, pattern source) is the point: a shop owner planning a trip two weeks out shouldn't have to remember, on the morning of, to open the app and flip a switch, and shouldn't risk forgetting to flip it back on when they return. This page's core promise is "set it once, forget about it, it handles both ends automatically."

### 1.4 — User story

> *"As a shop owner, I know three weeks from now I'll be closed for a family wedding — I want to tell NEARBY that now, so I don't have to remember to close (and reopen) the shop on the actual days, and so customers searching during that window aren't matched to a shop that won't answer."*

### 1.5 — Pattern source

Inspired by: Zomato Restaurant Partner's "Schedule leaves in advance" (Page Architecture doc §2.13, pattern source). NEARBY's version is simpler than a full leave-management system (no leave "types," no approval workflow, since there's no franchise-operator hierarchy above a single shop owner in NEARBY's model) — it is a single-purpose date-range scheduler, nothing more.

---

## 2. Functional Specification

### 2.1 — Data model addition (NEW)

**New collection: `shopClosures`** — a list rather than a single field on `shops`, because a shop can reasonably have more than one planned closure on the books at once (e.g. a long-planned trip next month, plus a shorter closure already known for next week).

| Field | Type | Notes |
|---|---|---|
| `_id` | ObjectId | Primary key — **NEW** |
| `shopId` | ObjectId → shops | Required — **NEW** |
| `startDate` | Date (date-only, no time component) | Required — interpreted in the shop's own `timezone` field (§4.2), consistent with how Opening Hours' `openTime`/`closeTime` are handled — **NEW** |
| `endDate` | Date (date-only, inclusive) | Required, must be on or after `startDate` — **NEW** |
| `reason` | String | Optional, free text (e.g. "Festival," "Family event") — shown only to the owner themselves, never to customers (§2.6) — **NEW** |
| `createdAt` | Date | Timestamp — **NEW** |

A closure is a whole-day concept, not a time-of-day one — unlike Opening Hours' per-day windows, there's no meaningful "closed from 2 PM to 5 PM three weeks from now" use case; the whole point (§1.3) is planning full days off, so `startDate`/`endDate` are deliberately date-only, not datetime, to keep the input simple.

### 2.2 — Eligibility logic this page's data feeds (NEW rule, extending §3.4)

Restated and extended from Master Build Document §3.4: a shop is eligible for a broadcast match only if `isOpen` is true AND (if a schedule exists) the current time falls inside that day's `openingHours` window. This page adds one more AND condition to that same check: **AND today's date does not fall within any `shopClosures` entry for this shop, evaluated in the shop's own timezone.**

This sits at exactly the same point in the matching query as the Rush-Hour Toggle's own eligibility addition (that spec, §2.1) — both are conditions layered onto the existing `$nearSphere`-plus-eligibility check (§3.1), and both are evaluated the same way: compared at query time, not enforced by a scheduled job (§2.3 below).

### 2.3 — No cron job needed (same reasoning as Rush-Hour, §2.3 of that spec)

A closure starts and ends itself by date comparison at query time — there is nothing to schedule, dispatch, or clean up server-side. On the day a closure's `endDate` passes, the very next matching query for that shop simply finds no active `shopClosures` entry covering "today" and treats the shop as eligible again, with zero added infrastructure — the same reuse-over-new-services reasoning the Master Build Document applies elsewhere (§3.1.3, §3.11) and the Rush-Hour Toggle specification applies to its own auto-expiry.

### 2.4 — Interaction with `isOpen` and Rush-Hour

- A shop can still be manually toggled `isOpen: false` or put into Rush-Hour during a scheduled closure window — this is redundant but harmless, since the closure condition alone already excludes the shop from matching; no special-case logic is needed to prevent "double closing."
- The reverse also holds: a scheduled closure does not touch `isOpen` at all (unlike Rush-Hour, which the Rush-Hour spec notes gets silently cleared if the owner manually goes Closed, §2.5 of that document) — a closure ending should restore the shop to exactly whatever `isOpen`/Opening-Hours state it already had, with no assumption that the owner wants to be reopened the moment the closure ends if they'd separately marked themselves closed for an unrelated reason beforehand.

### 2.5 — API endpoints (NEW)

| Endpoint | Auth | Body / Notes |
|---|---|---|
| `GET /api/v1/shops/me/closures` | Bearer (shop) | Returns this shop's closures, both upcoming and past, newest-starting-first — same pagination convention as every other list endpoint (§5.2) |
| `POST /api/v1/shops/me/closures` | Bearer (shop) | `{ startDate, endDate, reason? }` — creates a new closure entry; server validates `endDate >= startDate` and that the range doesn't overlap an existing closure for the same shop (§9) |
| `DELETE /api/v1/shops/me/closures/:closureId` | Bearer (shop) | Cancels a planned closure — only permitted while `startDate` is still in the future (§9); a closure already in progress or completed cannot be deleted, only viewed as history |

Resolved from the caller's JWT `sub` exactly like every other owner-scoped endpoint in this documentation series — never a client-supplied shop id.

### 2.6 — Customer-facing visibility (none, by design)

A customer broadcasting during a shop's scheduled closure simply never matches that shop — there is no customer-facing message like "this shop is on leave," because customers don't see a specific shop at all until it responds Yes (§3.1.1), and a closed shop never receives the broadcast in the first place. The `reason` field (§2.1) is owner-facing only, for their own reference when looking back at "why was I closed that week" — never surfaced anywhere on the customer app.

### 2.7 — Rate limiting

`POST /api/v1/shops/me/closures` — max 20 per shop per month, following the same convention used elsewhere in this series — generous for realistic planning needs, a guard against accidental spam/misuse.

---

## 3. Where It Lives

- **Entry point:** Shop Dashboard nav, grouped with Opening Hours as a sibling under Outlet-related settings (reachable from Outlet Management, page 2.6, as a third segmented-control tab alongside "Basic Info" and "Opening Hours" — completing that page's tab set rather than introducing a whole separate nav entry) — this keeps all three of the shop's schedule-shaping settings (basic info, weekly hours, planned closures) in one place, consistent with how a shop owner is likely to think of them as one "manage my availability" area.
- A single screen: an upcoming-closures list, an "Add a closure" action, and a past-closures history section below.

---

## 4. Section-by-Section Content & Layout

### 4.1 — Page header & primary action

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Page header | "Planned closures" | H1 | Gray 900 |
| Subtitle | "Let customers' searches skip your shop automatically on days you know you'll be closed." | Body | Gray 600 — states the value proposition (§1.3) in one line |
| Primary action | "Add a closure" | Button / SemiBold | Signal Teal fill, opens the date-range sheet (§4.2) |

### 4.2 — Add-a-closure bottom sheet

Same sheet mechanics as every other bottom sheet in this series (Brand Guide §9.3).

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Header | "When will you be closed?" | H1 | Gray 900 |
| Helper text | "Your shop won't show up in any customer searches during these dates. You don't need to do anything else — it starts and ends on its own." | Body | Gray 600 — this is the single most important reassurance on the whole page: nothing further is required of the owner (§1.3) |
| Start date field | Label: "First day closed" | Caption label / date picker | Calendar-style picker, cannot select a date in the past |
| End date field | Label: "Last day closed" | Caption label / date picker | Cannot select a date before the start date (§9); defaults to the same day as start date if left untouched, covering the common single-day-off case with minimal taps |
| Reason field (optional) | Label: "What's the occasion? (optional)" | Caption label / Body input | Free text, short, e.g. "Diwali," "Family trip" — explicitly optional, never required |
| Primary action | "Save closure" (disabled until both dates are set) | Button / SemiBold | Full-width, Signal Teal fill |
| Secondary action | "Cancel" | Button / SemiBold, text-only | Gray 600 text |

### 4.3 — Upcoming closures list

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Section header | "Upcoming" | H2 | Gray 900, shown only if at least one future or in-progress closure exists |
| Closure row | Date range (e.g. "Oct 12 – Oct 14") + reason if provided + a status pill | Body / Medium (dates), Caption (reason), Caption/SemiBold pill | Row geometry matches the Request History / Staff Management row pattern used throughout this series |
| Status pill — Scheduled | "Scheduled" | — | Gray 100 fill, Gray 600 text — neutral, matching this series' consistent quiet treatment for "nothing has happened yet, nothing to flag" states |
| Status pill — In progress | "In progress" | — | Signal Teal outline — the shop is currently closed under this entry right now |
| Row action (Scheduled only) | "Cancel" (text button, trailing edge) | Button / SemiBold | Danger #E5484D text — opens the confirmation in §4.5; hidden entirely on an "In progress" row, since an already-started closure can't be cancelled (§2.5) |

### 4.4 — Past closures (history)

| Element | Content / Copy | Type role | Notes |
|---|---|---|---|
| Section header | "Past" | H2 | Gray 900, shown only if at least one completed closure exists |
| Closure row | Date range + reason if provided | Body, Caption | Same row geometry as §4.3, minus the status pill and row action — a completed closure has no further state or action, it's pure history |

### 4.5 — Cancel-closure confirmation

Inline confirmation, matching the pattern established across this series (Rush-Hour Toggle §4.5, Staff Management §4.5):

> **Copy:** *"Cancel this closure? You'll go back to your normal hours for these dates."* — two inline buttons: **"Yes, cancel"** (Danger fill) and **"Keep it"** (text link).

### 4.6 — Empty state (no closures ever scheduled)

> **Copy:** *"You don't have any planned closures. Add one if you know you'll be closed on specific days ahead of time."*

A single static instance of the logomark's pin (Brand Guide §9.4 empty-state pattern), consistent with the same treatment used on the Reliability & Ratings, Request History, and Staff Management pages.

### 4.7 — Confirmation toasts

| Action | Copy |
|---|---|
| Closure added | "Closure added." |
| Closure cancelled | "Closure cancelled." |

---

## 5. Visual Design System Application

### 5.1 — Color mapping

| Token | Used for in this feature | Rationale |
|---|---|---|
| Signal Teal #00B8A9 | "Add a closure" / "Save closure" buttons, "In progress" status pill (outline) | Shop surface's dominant accent (Brand Guide §3.3) — planning ahead is a routine, positive shop-management action, the same category as every other action on this teal-forward surface |
| Gray 100 / Gray 600 | "Scheduled" status pill, reason text, helper/subtitle text | Neutral tone — a scheduled-but-not-yet-started closure is a normal planning state, consistent with this series' repeated use of quiet gray for "nothing to flag yet" states (Request History outcome badges, Help Centre's "Sent" ticket pill) |
| Danger #E5484D | "Cancel" row action and its confirmation button | Cancelling a planned closure resumes customer visibility — a meaningful enough change to warrant the same color used for cancel/reject actions throughout this series |
| Gray 900 | Date ranges, form labels, page header | Standard body-text accessibility rule (§3.4) |
| Surface / White | Page and sheet backgrounds | Standard page/card tokens (§3.2) |

| Signal Teal #00B8A9 | Gray 100 #F4F5F7 | Danger #E5484D | Gray 900 #14213D | Gray 600 #5B6472 |
|---|---|---|---|---|

Orange is not used anywhere on this page, consistent with every Shop Dashboard spec in this series — reserved exclusively for Yes/No response buttons elsewhere on this surface (Brand Guide §3.3).

### 5.2 — Typography mapping

| Role (Brand Guide §4.2) | Size / line-height / weight | Used here for |
|---|---|---|
| H1 | 24px / 32px, 600 SemiBold | Page header ("Planned closures"), date-sheet header |
| H2 | 18px / 26px, 600 SemiBold | "Upcoming" and "Past" section headers |
| Body | 15px / 22px, 400 Regular (Medium for date ranges) | Closure date ranges, helper text in the sheet |
| Caption | 13px / 18px, 400 Regular (SemiBold for status pills) | Field labels, reason text, status pills, subtitle |
| Button | 15px / 20px, 600 SemiBold | "Add a closure", "Save closure", "Cancel" row action |

### 5.3 — Spacing, grid & component styling

- Closure rows (upcoming and past): identical geometry to the Request History / Staff Management row pattern — 12px vertical / 16px horizontal padding, 1px Gray 300 bottom divider, no per-row card border.
- Add-a-closure sheet: standard form field rhythm (16px gap between fields, 6px between label and input), matching the Rush-Hour duration sheet and Outlet Management's form fields.
- Date pickers: native-feeling calendar widget, matching whatever the platform's standard date-input pattern is elsewhere in the app — no custom calendar component designed specifically for this page.
- Status pills: outline style, fully rounded, 4px vertical / 10px horizontal padding — same lighter-weight pill treatment as Staff Management's "Active"/"Pending" pills, appropriate for steady-state labels rather than one-time outcomes.

### 5.4 — Iconography

- No new custom iconography required. A small calendar-glyph icon (16×16, Gray 600) may lead each date field in the sheet, using a standard system-style calendar icon rather than a custom asset.
- The logomark pin (existing brand asset) appears once, statically, in the empty state (§4.6), identical treatment to its use across the Reliability & Ratings, Request History, and Staff Management pages.

---

## 6. Full Copy Deck

| State / Location | Exact copy |
|---|---|
| Page header | Planned closures |
| Subtitle | Let customers' searches skip your shop automatically on days you know you'll be closed. |
| Primary action | Add a closure |
| Sheet header | When will you be closed? |
| Sheet helper text | Your shop won't show up in any customer searches during these dates. You don't need to do anything else — it starts and ends on its own. |
| Start date label | First day closed |
| End date label | Last day closed |
| Reason label | What's the occasion? (optional) |
| Sheet primary action | Save closure |
| Sheet secondary action | Cancel |
| Section header — upcoming | Upcoming |
| Section header — past | Past |
| Status pill — scheduled | Scheduled |
| Status pill — in progress | In progress |
| Row action | Cancel |
| Cancel confirmation prompt | Cancel this closure? You'll go back to your normal hours for these dates. |
| Cancel confirmation — confirm | Yes, cancel |
| Cancel confirmation — dismiss | Keep it |
| Empty state | You don't have any planned closures. Add one if you know you'll be closed on specific days ahead of time. |
| Toast — added | Closure added. |
| Toast — cancelled | Closure cancelled. |
| Error — end before start | Last day can't be before the first day. |
| Error — overlapping | These dates overlap with a closure you've already scheduled. |
| Error — save failed | Couldn't save this closure. Check your connection and try again. |
| Error — rate limited | Too many closures added recently — try again later. |

---

## 7. Motion & Animation Specification

Every token below is reused from the existing Motion Token table (Brand Guide §9.1) — nothing new is introduced.

| Interaction | Token(s) used | Behavior |
|---|---|---|
| Add-a-closure sheet open/close | duration-standard (240ms), ease-out-soft | Standard bottom-sheet slide-up, identical mechanics to every other sheet in this series |
| Save button enable | duration-instant (100ms) | Opacity crossfade from disabled to enabled once both dates are set, no bounce |
| New closure appears in "Upcoming" | duration-quick (180ms), ease-out-soft | Row fades and slides in from the top of the list — same pattern as a new row appearing in Staff Management's list |
| Row moves from "Upcoming" to "In progress" styling (on its start date) | No animation — this happens between sessions, not live in front of the owner, since it's date-driven, not something they trigger; renders directly in its current state on next page load | Consistent with the "no loading flicker to design for" reasoning used elsewhere in this series for data that's simply correct-on-load |
| Closure cancelled | duration-quick (180ms), ease-standard | Row fades out and collapses its height smoothly, remaining rows shift up |
| Confirmation toast | duration-quick (180ms), ease-out-soft | Standard bottom-edge toast (§9.3) |
| Reduced motion | Single global override (§9.5) | All of the above collapse to instant state changes — no page-specific reduced-motion work required |

---

## 8. Imagery, Graphics, Icons & Video

No photography, illustration, or video anywhere on this page. It is a planning/scheduling utility, and per the shop-surface voice rule (Brand Guide §1.4), decorative imagery would only slow down a quick planning task. The only illustrative asset is the existing logomark pin, used once, statically, in the empty state (§4.6, §5.4) — matching the same treatment already established on the Reliability & Ratings, Request History, and Staff Management pages.

No skeleton-loader design work beyond the standard shimmer pattern already defined in the Brand Guide (§9.4) — `GET /shops/me/closures` (§2.5) is a single lightweight list call.

---

## 9. States & Edge Cases

| Case | Behavior |
|---|---|
| Owner tries to set an end date before the start date | Client-side validation blocks this before submit; inline error: "Last day can't be before the first day." (§6) |
| Owner tries to add a closure overlapping an existing one | Server rejects with 400; inline error: "These dates overlap with a closure you've already scheduled." — checked client-side against the already-loaded upcoming list where possible, and re-validated server-side |
| Owner adds a closure that starts today | Treated the same as any future closure — "today" is a valid start date, and the shop is immediately excluded from matching per §2.2's eligibility check the moment it's saved, with no special same-day handling needed |
| Owner tries to cancel a closure that has already started | The "Cancel" row action is hidden entirely for "In progress" rows (§4.3) — an in-progress closure can only be waited out, not retroactively cancelled, since customers may have already been correctly excluded from matching during part of it |
| Owner wants to shorten an in-progress closure (end it early) | Not supported as an edit in v1 — the owner's fallback is the manual `isOpen` switch on the Live Dashboard (page 2.4), which independently controls live availability regardless of a closure entry; the closure entry itself simply becomes a no-op past its `isOpen`-driven reopening, since §2.2's eligibility check is an AND condition and `isOpen: true` alone doesn't override an active closure — **noted as a real v1 gap**: an owner ending a trip early cannot actually reopen early through this page alone. Recommended fix for v1.1: allow editing an in-progress closure's `endDate` to today; flagged here rather than silently built, since it changes the "closures are immutable once started" rule stated elsewhere in this spec |
| Shop status becomes blocked while closures are scheduled | Irrelevant to display — a blocked shop is already excluded from all matching regardless of any closure entry (Master Build Document §3.5/§3.8), consistent with how the Rush-Hour Toggle specification treats the same scenario (that spec, §9) |
| Two closures somehow overlap due to a race condition (rare) | Server-side validation at write time (§2.5) is the actual guard; the eligibility check itself (§2.2) is unaffected by an overlap even if one somehow occurred, since it only needs to find *any* covering closure entry, not exactly one |

---

## 10. Accessibility

- Date pickers are fully keyboard-operable and announce the selected date to screen readers, not relying on a purely visual calendar grid.
- Status pills always carry a text label ("Scheduled", "In progress") — color is reinforcement only, never the sole signal.
- The "Cancel" row action is a clearly labeled, individually focusable button, not an icon-only control with ambiguous purpose.
- Inline validation errors use `aria-describedby`/`aria-invalid` on the associated date field, matching the pattern established on Outlet Management and Opening Hours.
- Minimum 44×44px touch target on every button, pill, and date-field tap target.
- Reduced motion: fully covered by the existing global override (§7) — no page-specific work required.

---

## 11. Admin Visibility & Data Management

- No new admin-facing surface is required — a shop's planned closures are a routine self-service scheduling action with no approval-relevant or trust-and-safety implication.
- **Optional future addition (not built in this pass):** the All Shops directory (Page Architecture page 3.4) could surface a shop's current/upcoming closures alongside its other profile details, since admin already sees everything else about a shop — noted as a natural extension, consistent with how this documentation series flags possible follow-ons without building them prematurely.
- No interaction with the Reports queue (page 3.6) — planning a closure cannot affect another shop or a customer in a way that would generate a report.

---

## 12. Component Inventory — Developer Handoff

| Component | Type | States | Key tokens |
|---|---|---|---|
| ClosureListSection | Composite (header + rows) | upcoming-list, past-list, empty | H2, Body, Caption, Signal Teal / Gray 100 pills |
| ClosureRow | List item | scheduled, in-progress, past (no pill/action) | Body, Caption, Danger text action (scheduled only) |
| AddClosureSheet | Bottom sheet | closed, open, valid, submitting, error | H1, Body, Button, duration-standard |
| CancelClosureConfirmInline | Inline confirm | hidden, visible | Danger, Button, duration-instant |
| ClosureEmptyState | Composite (logomark + copy) | visible only when zero closures exist | Body, logomark pin (static) |
| Toast (reused) | Existing shared component | closure-added, closure-cancelled, error | duration-quick, ease-out-soft |

Five page-specific composites plus one reused shared component — every visual primitive (sheet, pill, inline confirm, list row) is drawn from patterns already established across the Rush-Hour Toggle, Opening Hours, Staff Management, and Help Centre specifications in this series. No new design-system primitive is introduced.

---

## Appendix A — Quick Reference: What's New vs. Reused

| New for this feature | Reused as-is from existing docs |
|---|---|
| `shopClosures` collection (§2.1) | Every color token (§3.2, Brand Guide) |
| One eligibility condition added to the §3.4/§3.1 matching query (§2.2) | Every type-scale role (§4.2, Brand Guide) |
| Three new API endpoints (§2.5) | Bottom-sheet, list-row, inline-confirm, and toast interaction patterns (§9.3, Brand Guide — reused directly from prior specs in this series) |
| Copy deck (§6 of this document) | Empty-state pattern using the static logomark pin (§9.4, Brand Guide) |
| Known v1 gap flagged, not silently built: no early-end editing for an in-progress closure (§9) | Reduced-motion global override (§9.5, Brand Guide) |

---

*— End of specification —*
