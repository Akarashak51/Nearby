# NEARBY — Admin Console: All Shops Directory
### Condensed Page Spec — v1.0
**Refs:** Master Build Doc v2.6.2 (§3.5, §4.2, §4.4, §5.3) · Brand & PWA Guide v1.0 · Page Architecture (§3.4)
**Surface:** `/admin/*` — Navy-forward, Orange reserved for destructive actions (§3.3) · **Role:** admin

---

## 1. Overview
Every shop regardless of status — pending, approved, rejected, blocked — filterable, with drill-in to a full profile and stats (Page Architecture §3.4). This is the **complete register**, distinct from the Approval Queue (§3.3, pending-only, action-oriented); this page is browse-and-inspect, action is secondary (Block only — approve/reject stays on the Queue).

## 2. Functional Spec
- `GET /api/v1/admin/shops?status=&category=&page=` — paginated, existing convention (§5.2), all statuses (unlike Queue's pending-only filter).
- Drill-in reuses the same shop document (§4.2) already shown on the Approval Queue's detail view — no new fields.
- **Block action** (only for `status: 'approved'` shops) — `PATCH /admin/shops/:id/block` (§3.5, NEW v2.6.1) — the one destructive action reachable from here; auto-cancels the shop's pending requests (§3.8). This is the page's only Orange-colored action (§3.3: Orange reserved for destructive admin actions).
- Stats shown per shop on drill-in are **read-only aggregates** already computed for the Reliability & Ratings view (shop-side spec, §2.1) — `reliabilityScore`, `avgRating`, `foundCount`/`notFoundCount` — this page reuses them, doesn't recompute.

## 3. Where It Lives
Admin nav → "Shops" (distinct tab from "Approvals"). Drill-in opens as a detail panel/route, not a modal (enough content — profile + stats + history — to warrant its own screen).

## 4. Section-by-Section

| Element | Copy | Type role | Notes |
|---|---|---|---|
| Page header | "All shops" | H1 | Ink Navy |
| Filter — status | Chips: All · Pending · Approved · Rejected · Blocked | Button/SemiBold | Active chip Ink Navy fill |
| Filter — category | Dropdown/select | Caption | Platform's category list (same set used on shop-side Outlet Management) |
| Search | "Search by shop name" | Body input | Client or server search over `shopName` |
| List row | Shop name, category, status pill, city/area | Body/Caption | Status pill color per §5.1 below |
| **Drill-in — profile** | Name, category, address, phone, photo/doc | Body | Same layout as Approval Queue's detail (reused component) |
| **Drill-in — stats** | Reliability score, avg rating, found/not-found counts | Body/Caption | Read-only, pulled from shop doc |
| **Drill-in — status history** | e.g. "Approved on {date}", "Blocked on {date}" if applicable | Caption | If `rejectionReason` exists, shown here too |
| Block action (approved shops only) | "Block this shop" | Button/SemiBold | Orange #FF5A36 fill — the one legitimate Orange use on `/admin/*` (§3.3) |
| Block confirm | "Block {shopName}? This cancels their pending requests and removes them from search immediately." | Inline confirm | Two buttons: "Yes, block" (Orange) / "Cancel" |
| Empty (filtered) | "No shops match this filter." | Body | — |

## 5. Visual Design
- **Colors:** Ink Navy (headers, active filter, primary nav elements), Gray 900/600 (body/meta), Gray 300 (dividers). **Status pills:** Approved = Signal Teal outline (borrowed from shop-surface Success token, used here purely as a status indicator, not a call-to-action color); Pending = Warning #FFB627 outline; Rejected = Gray 100/Gray 600 (neutral, not a failure state — mirrors the Request History spec's rationale for neutral non-outcomes); Blocked = Danger #E5484D outline. **Orange = Block button only** (§3.3).
- **Background:** White cards/rows on Gray 100 page background — standard admin list pattern, consistent with Approval Queue.
- **Cards:** List rows use the same row geometry as every list page across this whole documentation series (12px vertical/16px horizontal padding, 1px Gray 300 divider) — deliberate reuse for a consistent admin-console feel.
- **Typography:** Standard scale (§4.2), no special treatment.

## 6. Graphics / Images / Video
- Shop photo/doc shown on drill-in only (not in the list row, to keep the directory scannable) — same full-resolution/zoom treatment as the Approval Queue.
- No illustration, video, or decorative imagery — pure data-browsing utility.
- Empty-state has no illustration either (unlike shop-side empty states) — admin surfaces in this series stay plain/functional throughout, consistent with the Admin Login spec's restraint principle.

## 7. Animation
- Filter chip switch: duration-instant (100ms), matches shop-side chip pattern.
- Row → drill-in transition: duration-quick (180ms) slide/fade, standard navigation feel, no special admin treatment.
- Block confirm inline reveal: duration-instant (100ms) fade, no shake (calm-tone rule applies even to destructive actions — clarity, not alarm).
- Row removed after Block (if filtered view excludes blocked shops): duration-quick (180ms) fade/collapse.

## 8. States & Edge Cases

| Case | Behavior |
|---|---|
| Shop has zero rated visits (new/pending) | Stats section shows the same "not yet scored" framing as the shop-side Reliability & Ratings empty state — reused copy/logic, not redefined here |
| Admin blocks a shop with requests mid-flight | Auto-cancel confirmed by the confirm-dialog copy itself (§4) — no separate follow-up screen needed, `PATCH /admin/shops/:id/block` handles it server-side (§3.5) |
| Search returns zero results | Same empty-state copy as filter-empty (§4) |
| Rejected shop later re-applies and is re-approved | Directory reflects current `status` only — history/timeline (§4 drill-in) shows the full status trail so admin isn't confused by an apparent "skip" from rejected to approved |

## 9. Accessibility
- Status pills carry text labels always, color is reinforcement only (consistent with every list page in this series).
- Filter chips use `role="tab"`/`aria-selected`.
- Block confirm requires explicit second tap — no accidental single-tap destructive action.
- 44×44px min touch targets throughout.

## 10. Admin/Data Management
This page is itself the primary admin data-management surface for shops — no further meta-layer. Block is the only write action; all else is read/filter/search.

## 11. Component Inventory
| Component | States |
|---|---|
| ShopDirectoryList | populated, empty, loading |
| StatusFilterBar | all-active, per-status-active |
| ShopDrillInPanel | viewing, block-confirm-open |
| StatusPill | approved, pending, rejected, blocked |
| Toast (reused) | blocked, error |

---
*— End of condensed spec —*
