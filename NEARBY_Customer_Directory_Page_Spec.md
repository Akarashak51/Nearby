# NEARBY — Admin Console: Customer Directory
### Condensed Page Spec — v1.0
**Refs:** Master Build Doc v2.6.2 (§3.5, §3.8, §4.1, §5.3) · Brand & PWA Guide v1.0 · Page Architecture (§3.8)
**Surface:** `/admin/*` — Navy-forward, Orange for destructive actions (§3.3) · **Role:** admin

---

## 1. Overview
Lightweight search/view of customer accounts for support purposes — "no edit beyond support actions" (Page Architecture §3.8). Pattern-sourced from generic "manage customers" admin tables. This is the customer-side mirror of the All Shops Directory (prior spec) — browse/inspect first, enforcement (suspend/ban/reinstate) is the only write path.

## 2. Functional Spec — reuses `users` schema as-is (§4.1), no new fields
- Fields shown: `email`/`phone`, `authProvider` (local/google), `isVerified`, `status` (active/suspended/banned), `suspensionReason`, `createdAt`.
- `GET /api/v1/admin/users?status=&search=` — paginated, existing convention (§5.2) — **endpoint not explicitly named in Master Build Doc's table but implied by "customer directory" + existing admin-list pagination pattern**; flagged as a small, natural addition consistent with `GET /admin/shops`'s own shape, not a new architectural concept.
- `PATCH /api/v1/admin/users/:id/suspend` (§5.3, existing) — `{ suspensionReason }` — **auto-cancels any open request** (§3.8) the instant this fires, no separate admin step.
- `PATCH /api/v1/admin/users/:id/reinstate` (§3.5, existing) — sets `status: 'active'` back — "if a suspension is later found unwarranted" — **this is the one enforcement action in the whole admin console with a documented reversal path**, unlike shop-blocking (prior spec, which has none) — worth designing the UI to make reversible actions feel appropriately lower-stakes than the shop-block flow.
- **Suspend vs. Ban — same endpoint, same effect, terminology distinction only:** the schema has one `suspend` endpoint but a `status` enum of `active | suspended | banned` (§4.1) — the Master Build Doc doesn't define a separate `/ban` endpoint. Recommend treating "Ban" in the UI as `PATCH .../suspend` with `status` set to `banned` via the same body shape (extend `{ suspensionReason, status: 'suspended' | 'banned' }`), rather than assuming an undocumented second endpoint — **flagged here as a required small clarification for backend**, not silently invented.

## 3. Where It Lives
Admin nav → "Customers." Also reachable from the Reports queue (prior spec) when `reportedType: 'user'` — same shortcut pattern as that page's "Block this shop" link.

## 4. Section-by-Section

| Element | Copy | Type role | Notes |
|---|---|---|---|
| Page header | "Customers" | H1 | Ink Navy |
| Search | "Search by email or phone" | Body input | Server-side search |
| Filter chips | All · Active · Suspended · Banned | Button/SemiBold | Active chip Ink Navy fill |
| List row | Email/phone, join date, status pill | Body/Caption | Standard row geometry, reused across this whole series |
| **Detail — account** | Email/phone, sign-up method (local/Google), verified status, joined date | Body | Read-only |
| **Detail — status history** | "Suspended on {date}: {suspensionReason}" if applicable | Caption | — |
| Action — Suspend | "Suspend account" | Button/SemiBold, Orange fill | Destructive-adjacent, §3.3 |
| Action — Ban | "Ban account" | Button/SemiBold, Orange fill (darker/emphasized variant or same Orange with distinct label) | More severe than Suspend — same color family since both are destructive per §3.3's binary rule (Orange = destructive), but copy must clearly distinguish severity (§6) |
| Action — Reinstate (suspended/banned only) | "Reinstate account" | Button/SemiBold, Signal-positive (Ink Navy or a neutral confirm color — **not** Orange, since this reverses a destructive action rather than performing one) | The one reversible admin action in this console |
| Reason field (suspend/ban) | Label: "Reason" (required) | Caption/Body input | → `suspensionReason` |
| Consequence note | "Any request this customer has open right now will be cancelled automatically." | Caption | Gray 600 — states §3.8's effect plainly, matching the Block/Suspend shop spec's own consequence-summary pattern |
| Empty (filtered) | "No customers match this filter." | Body | — |

## 5. Visual Design
- **Colors:** Ink Navy (headers, filters, Reinstate action), Gray 900/600 (body/meta). Status pills: Active = Signal Teal outline, Suspended = Warning #FFB627 outline, Banned = Danger #E5484D outline (more severe than Suspended, distinct color per severity — unlike shops, which only have one enforcement state). **Orange = Suspend/Ban buttons** (§3.3 destructive rule).
- **Background:** White rows on Gray 100 — same pattern as every admin list page in this series.
- **Cards:** Standard row geometry (12px/16px padding, 1px Gray 300 divider), consistent with All Shops Directory and Reports Queue.

## 6. Graphics / Images / Video
None — no avatar/photo exists in the `users` schema, no illustration needed. Pure data/text utility page, consistent with every admin surface in this series.

## 7. Animation
- Filter chip switch: duration-instant (100ms).
- Suspend/Ban/Reinstate confirm reveal: duration-instant (100ms) fade, no shake (calm-tone rule applies even here).
- Row status-pill update after action: duration-quick (180ms) crossfade to new pill color/label.

## 8. States & Edge Cases

| Case | Behavior |
|---|---|
| Customer has an open request at moment of suspend/ban | Auto-cancelled server-side per §3.8 — consequence note (§4) sets this expectation before admin confirms |
| Admin reinstates a banned customer | Full reset to `active` — no partial/probationary state exists in the schema, so the UI shouldn't imply one |
| Reason left empty on suspend/ban | Action button stays disabled until filled, matching the Block-shop flow's own required-reason pattern |
| Customer signed up via Google (`authProvider: 'google'`) | No password-reset-adjacent actions shown for this account (n/a) — detail view simply reflects `authProvider` as read-only info |
| Search returns zero results | Same empty-state copy as filter-empty (§4) |

## 9. Accessibility
- Status pills always carry text labels, color reinforcement only — especially important here since Suspended/Banned are visually similar-severity colors that must not rely on color alone to distinguish.
- Reason field: `aria-required`, disabled-action state announced via `aria-disabled`.
- 44×44px min touch targets throughout.

## 10. Admin/Data Management
This page is the customer-moderation surface itself — no further layer above admin. Reinstate is the one built-in undo path in the entire admin console (§2) — worth documenting for support-process training, not just UI.

## 11. Component Inventory
| Component | States |
|---|---|
| CustomerDirectoryList | populated, empty, loading |
| CustomerStatusFilterBar | all/active/suspended/banned |
| CustomerDetailPanel | viewing, suspend-form-open, ban-form-open, reinstate-confirm |
| StatusPill | active, suspended, banned |
| Toast (reused) | suspended, banned, reinstated, error |

---
*— End of condensed spec —*
