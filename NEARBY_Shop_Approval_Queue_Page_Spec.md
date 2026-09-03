# NEARBY — Admin Console: Shop Approval Queue
### Condensed Page Spec — v1.0
**Refs:** Master Build Doc v2.6.2 (§3.3, §4.2, §5.3) · Brand & PWA Guide v1.0 · Page Architecture (§3.3)
**Surface:** `/admin/*` — Navy-forward, Orange reserved for destructive actions only (§3.3) · **Role:** admin

---

## 1. Overview
List of `pending` shops → drill into profile + photo/doc + address plausibility → Approve/Reject with reason (§3.3, FR-3.1). Every card must give admin enough to judge all **four fixed criteria** without leaving the page:
**(a)** phone reachable — pilot-scale manual callback, no OTP (§3.3) · **(b)** address geocodes to a plausible, real, non-duplicate location · **(c)** photo/doc not a duplicate of another shop's submission · **(d)** declared category consistent with photo/address context.

## 2. Functional Spec
- `GET /api/v1/admin/shops?status=pending` (§5.3, NEW v2.6) — paginated, oldest-first (fairness — first-submitted, first-reviewed) or newest-first (toggle, admin preference; default oldest-first).
- `PATCH /api/v1/admin/shops/:id` — `{ status: 'approved' }` or `{ status: 'rejected', rejectionReason }` — `rejectionReason` **required** on reject (§3.3: "so the applicant can be told why and reapply").
- Approve/Reject is **not undoable from this page** — an approved shop can only later be Blocked (separate page, §3.5, distinct action/endpoint); a rejected shop can only be fixed by the shop owner re-applying (§3.3 FIXED v2.6.1) — this page has no "undo" button by design.
- No new schema — reuses `shops.status`, `photoUrl`/`documentUrl`, `address`, `phone`, `category` (§4.2) as-is.

## 3. Where It Lives
Admin nav → "Approvals" (badge shows pending count, sourced from the same list's total). Entry row from Dashboard/Overview (page 3.2) also deep-links here.

## 4. Section-by-Section

| Element | Copy | Type role | Notes |
|---|---|---|---|
| Page header | "Shop approvals" | H1 | Ink Navy |
| Subheader | "{N} pending" | Caption | Gray 600 |
| Sort toggle | "Oldest first" / "Newest first" | Caption/link | — |
| List row (collapsed) | Shop name, category, submitted date, thumbnail | Body/Caption | Tap opens full detail (drawer or same-page expand) |
| **Detail — photo/doc** | Full-size image, zoomable | — | The single most safety-critical element for criterion (c) — must render large, not thumbnail-only |
| **Detail — address** | Full address text + small static map pin | Body | For criterion (b); a static map image (no interactive/live map needed) is sufficient |
| **Detail — phone** | Number, tap-to-call | Body | Admin manually calls per (a); no in-app call-logging needed for pilot scale |
| **Detail — category** | Declared category | Body | Judged against photo/address by the admin's own eye (d) — no automated check |
| Approve action | "Approve" | Button/SemiBold | Signal-positive green or Ink Navy fill (Orange NOT used — reserved for destructive only, §3.3) |
| Reject action | "Reject" | Button/SemiBold, outline | Opens reason field below before enabling submit |
| Reject reason field | Label: "Reason for rejection" | Caption/Body input | Required, free text, sent verbatim as `rejectionReason` |
| Empty queue | "No shops waiting for approval." | Body | Calm, not celebratory — just a factual state |

## 5. Visual Design
- **Colors:** Ink Navy #14213D (headers, primary actions), Gray 900/600 (body/meta text), Gray 300 (dividers/borders). **Danger #E5484D** on the Reject button's confirm step only. **Orange is never used on this page** — Brand Guide §3.3 reserves it strictly for destructive admin actions elsewhere (e.g. Block), and Approve/Reject here are pre-approval decisions, not enforcement — using Danger (not Orange) for Reject keeps that distinction intact.
- **Background:** White cards on Gray 100 page background — standard admin list pattern.
- **Cards:** 16px radius, 1px Gray 300 border; detail view can be a full-width expand or a side drawer — either way, photo gets the most visual real estate on the screen.
- **Typography:** Standard scale (§4.2) — no special admin treatment needed beyond Navy substituting for Teal/Orange as the surface accent.

## 6. Graphics / Images / Video
- **Photo/document image is the core content of this page** — full resolution on tap/zoom, not compressed thumbnails, since criterion (c) (duplicate detection) depends on admin actually seeing detail.
- Static map thumbnail (address pin) — no live/interactive map required, this isn't a navigation task.
- No illustration, no video, no decorative imagery — this is a working queue, not a branded moment.

## 7. Animation
- Row expand to detail: duration-quick (180ms) — standard.
- Approve/Reject success: row fades out of the queue list (180ms) — no celebratory animation (approving a shop is routine admin work, not a reward moment).
- Reject reason field reveal: duration-quick (180ms) fade/expand under the Reject button.
- No animation on the photo itself beyond a standard pinch/tap-to-zoom.

## 8. States & Edge Cases

| Case | Behavior |
|---|---|
| Reject tapped, no reason entered | Submit disabled until reason is non-empty (§2) |
| Two admins review the same shop simultaneously (rare, single-admin pilot) | Last write wins — no optimistic-concurrency handling needed at pilot scale (consistent with single-admin assumption, §3.5A) |
| Photo fails to load | Placeholder + "Couldn't load image — try refreshing" inline, Approve/Reject stay disabled until resolved (can't judge criterion c blind) |
| Shop re-applies after rejection | Re-enters this same queue as a fresh `pending` row (§3.3 re-apply flow) — no distinct "resubmitted" visual treatment required, though a small "Re-applied" tag is a nice-to-have, not required |

## 9. Accessibility
- Photo has alt text ("Storefront photo submitted by {shopName}"); zoom controls keyboard-operable.
- Reject button requires reason before submit is enabled — button disabled-state announced via `aria-disabled`.
- 44×44px min touch targets throughout.

## 10. Admin/Data Management
This page **is** the admin/data-management surface for shop approval — no further admin-of-admin layer exists. Every approve/reject action is final in the sense described in §2 (no undo here).

## 11. Component Inventory
| Component | States |
|---|---|
| ApprovalQueueList | populated, empty, loading |
| ApprovalDetailView | viewing, photo-zoomed, reject-reason-open, submitting |
| Toast (reused) | approved, rejected, error |

---
*— End of condensed spec —*
