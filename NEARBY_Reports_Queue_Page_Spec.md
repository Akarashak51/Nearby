# NEARBY — Admin Console: Reports Queue
### Condensed Page Spec — v1.0
**Refs:** Master Build Doc v2.6.2 (§3.5, §4.7, §5.3) · Brand & PWA Guide v1.0 · Page Architecture (§3.6)
**Surface:** `/admin/*` — Navy-forward, Orange for destructive actions (§3.3) · **Role:** admin

---

## 1. Overview
Triage queue for customer-filed reports (`report:new` event → admin room, §3.5) against a shop or user. Standard trust & safety queue (Page Architecture §3.6, pattern source: present in every marketplace admin panel). Feeds directly into the Block/Suspend flow (prior spec) when a report warrants enforcement.

## 2. Functional Spec — reuses `reports` schema as-is (§4.7), no new fields
- `reportedType`: `shop` | `user`, `reportedId`, `reason` (short category), `details` (free text), `status`: `open` | `actioned` | `dismissed` (default `open`), `adminNote`.
- `GET /api/v1/admin/reports` (§3.6) — paginated, filterable by status, existing convention (§5.2).
- `PATCH /api/v1/admin/reports/:id` (§5.3) — `{ status: 'actioned' | 'dismissed', adminNote }`.
- **"Actioned" doesn't auto-trigger enforcement** — resolving a report as `actioned` and actually blocking a shop / suspending a user are two separate steps (this queue → the relevant enforcement page) — no implicit coupling, so admin must explicitly navigate to Block/Suspend if enforcement is the outcome; this page only tracks triage status.
- Real-time: new reports arrive live via the `report:new` socket event into the admin room (§3.5) — this page should reflect that without a manual refresh.

## 3. Where It Lives
Admin nav → "Reports" (badge shows open-report count, same count surfaced on Dashboard/Overview, page 3.2). Originates from customer's Report a Shop (customer page 1.16).

## 4. Section-by-Section

| Element | Copy | Type role | Notes |
|---|---|---|---|
| Page header | "Reports" | H1 | Ink Navy |
| Filter chips | All · Open · Actioned · Dismissed | Button/SemiBold | Active = Ink Navy fill; default view = Open |
| List row | Reported type + name (shop/user), reason category, submitted date, status pill | Body/Caption | Row geometry matches every list page in this series |
| **Detail — subject** | Link to the reported shop's profile (All Shops Directory drill-in, prior spec) or user record | Body/link | One tap takes admin straight to context — critical, since a report is meaningless without seeing who/what it's about |
| **Detail — reason** | Short category (e.g. "Fake listing," "No response," "Inappropriate content") | Body | From `reason` field |
| **Detail — details** | Reporter's free-text explanation | Body | From `details`, may be empty |
| **Detail — reporter** | Reporting customer (admin-only visibility, never shown to the reported party) | Caption | From `reporterId` |
| Admin note field | Label: "Admin note (internal)" | Caption/Body input | Free text, saved to `adminNote` — never shown to customer or shop |
| Action — Dismiss | "Dismiss" | Button, outline | Sets `status: 'dismissed'` — for reports with no merit |
| Action — Mark actioned | "Mark as actioned" | Button/SemiBold | Sets `status: 'actioned'` — used *after* admin has separately taken (or decided not to take) enforcement action |
| Shortcut — Block shop | "Block this shop →" | Button/SemiBold, Orange | Only shown if `reportedType: 'shop'`; routes to the Block/Suspend flow (prior spec), pre-filling context |
| Empty (Open filter) | "No open reports." | Body | Calm, factual — a genuinely good state, but not celebrated with color/animation |

## 5. Visual Design
- **Colors:** Ink Navy (headers, filters, "Mark as actioned"), Gray 900/600 (body/meta). Status pills: Open = Warning #FFB627 outline (needs attention, not yet a failure), Actioned = Signal Teal outline (resolved-positively), Dismissed = Gray 100/Gray 600 neutral (resolved, no action needed — same "not a failure" neutral treatment used for non-outcomes throughout this series). **Orange = "Block this shop" shortcut only** (§3.3).
- **Background:** White rows on Gray 100 — standard admin list pattern, consistent with Approval Queue and All Shops Directory.
- **Cards:** Standard row geometry (12px vertical/16px horizontal padding, 1px Gray 300 divider) reused directly from every list page in this series.

## 6. Graphics / Images / Video
None beyond what the linked shop/user profile itself shows (photo, etc., owned by those pages) — this queue itself is text/data only, no illustration or decorative imagery, consistent with every admin page in this series.

## 7. Animation
- Filter chip switch: duration-instant (100ms).
- New report arrives live (`report:new`): row fades/slides into the top of the Open list (duration-standard, 240ms, ease-out-soft) — same treatment as a new card arriving on the shop-side Incoming Request feed, since this is genuinely time-sensitive admin-facing content.
- Resolve (dismiss/actioned): row fades out of the current filter view if it no longer matches (duration-quick, 180ms).
- No animation on the admin note field beyond standard focus/typing.

## 8. States & Edge Cases

| Case | Behavior |
|---|---|
| Report against a shop that's already been blocked | Detail view still opens normally; "Block this shop" shortcut is hidden/replaced with a note: "This shop is already blocked." |
| Report against a user account already suspended/banned | Same pattern — shortcut replaced with status note |
| Multiple reports against the same shop | Each is its own row/ticket — no auto-grouping in v1 (kept simple, matching this series' general preference against over-building); admin can see report history by checking the shop's own profile if cross-referencing is needed |
| Reporter files with empty `details` | Still valid — `reason` category alone is sufficient to triage; `details` is optional per schema (§4.7) |
| Report dismissed then reporter files again for the same issue | New, independent report row — no merge logic |

## 9. Accessibility
- Status pills always carry text labels, color reinforcement only.
- Filter chips use `role="tab"`/`aria-selected`.
- New live-arriving reports announced via `aria-live="polite"` on the list container.
- 44×44px min touch targets throughout.

## 10. Admin/Data Management
This page is the T&S data-management surface itself. `adminNote` is internal-only (never customer/shop-visible) — must be enforced server-side, not just hidden client-side, since it may contain sensitive triage reasoning.

## 11. Component Inventory
| Component | States |
|---|---|
| ReportsQueueList | populated, empty, loading |
| ReportStatusFilterBar | all/open/actioned/dismissed active |
| ReportDetailPanel | viewing, note-editing, submitting |
| StatusPill | open, actioned, dismissed |
| Toast (reused) | actioned, dismissed, error |

---
*— End of condensed spec —*
