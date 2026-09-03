# NEARBY — Admin Console: Block / Suspend Shop
### Condensed Page Spec — v1.0
**Refs:** Master Build Doc v2.6.2 (§3.5, §3.8, §4.2, §5.3) · Brand & PWA Guide v1.0 · Page Architecture (§3.5)
**Surface:** `/admin/*` — Navy-forward, Orange for destructive actions (§3.3) · **Role:** admin

---

## 1. Terminology note (read first)
Shops only have **one** enforcement state: `blocked` (§4.2 status enum: pending/approved/rejected/blocked). There is **no separate "suspend" state for shops** — "suspended"/"banned" exist only on `users` for customer accounts (§4.1, §3.5). This page's Page Architecture title carries "Suspend" loosely from the generic marketplace-moderation pattern it's sourced from; the actual mechanism specified below is **Block only**, distinct from Reject (§3.3, pre-approval) exactly as the Master Build Doc distinguishes them (§4.2, FIXED v2.6.1).

## 2. Overview
The dedicated flow for taking enforcement action against an **already-approved** shop — for misconduct, policy violation, or any reason requiring removal that isn't a pre-approval rejection. Distinct from the All Shops Directory's embedded Block action (prior spec, §4) in that this document treats it as its own full flow with reason-capture and consequence transparency, since a page architecture entry of its own signals it deserves more than an inline confirm.

## 3. Functional Spec
- `PATCH /api/v1/admin/shops/:id/block` (§3.5, NEW v2.6.1) — sets `status: 'blocked'`, `rejectionReason` (reused field, doubles as block reason per schema, §4.2).
- **Immediate effects** (§3.8, no extra admin action needed): shop disappears from search instantly; auto-cancels its pending requests; excluded from any shortlist even if it already said Yes to an in-flight request, re-checked at selection time (§3.1.1).
- **No unblock endpoint exists in the Master Build Doc** — flagged here as a genuine gap: once blocked, there is no documented path back to `approved` other than direct DB intervention. Recommend as a v1.1 addition (`PATCH /admin/shops/:id/unblock` → `status: 'approved'`) rather than silently assuming one exists.
- Reason is required (mirrors Reject's requirement on the Approval Queue) — stored in the same `rejectionReason` field, shown to the shop owner if/when they check their own dashboard status.

## 4. Where It Lives
Reached from: (a) the All Shops Directory drill-in's "Block this shop" action (prior spec), or (b) directly from the Reports queue (page 3.6) when a filed report against a shop warrants enforcement — this page/flow is the shared destination both entry points lead to.

## 5. Section-by-Section

| Element | Copy | Type role | Notes |
|---|---|---|---|
| Header | "Block {shopName}" | H1 | Ink Navy |
| Consequence summary | "This shop will disappear from customer search immediately. Any requests it's currently part of will be automatically cancelled." | Body | Gray 600 — states §3.8's actual effects plainly, not left implicit |
| Reason field | Label: "Reason for blocking" (required) | Caption/Body input | Free text — becomes `rejectionReason` |
| Linked report (if entry point b) | "Related to report #{id}" | Caption/link | Shown only when reached from Reports queue; links back to that report |
| Primary action | "Block shop" | Button/SemiBold | **Orange #FF5A36 fill** — the one legitimate Orange use per surface rule (§3.3) |
| Cancel | "Cancel" | Button, text-only | Gray 600 |
| Success confirmation | "{shopName} has been blocked." | Toast | — |

## 6. Visual Design
- **Colors:** Orange #FF5A36 for the Block button only (§3.3 destructive-action rule) — everything else Ink Navy/Gray, matching every other admin page in this series. Consequence summary text in Gray 600, **not** Danger red — it's informational, not itself an alarm (the Orange button already signals severity).
- **Background:** White card on Gray 100, centered, generous padding (32px) — mirrors the Admin Login spec's "this should feel deliberate" principle, appropriate for a high-consequence action.
- **Card:** 16px radius, 1px Gray 300 border.

## 7. Graphics / Images / Video
None. No icon, no illustration — text and the consequence summary carry the full weight of this screen; decoration would undercut the seriousness.

## 8. Animation
- Reason field required-state: Block button stays disabled (duration-instant, 100ms crossfade) until text is entered — no shake/alarm styling (calm-tone rule, §1.4, applies even to destructive flows: clarity over alarm).
- On submit: standard button press feedback, then navigate back to the originating page (Directory or Reports) with the success toast.

## 9. States & Edge Cases

| Case | Behavior |
|---|---|
| Reason left empty | Block button stays disabled |
| Shop has an active Rush-Hour or scheduled closure at block time | Irrelevant — both are already moot the instant `status` leaves `approved` (per Rush-Hour spec §9 and Closure spec §9, both already state a blocked shop is excluded from matching regardless) |
| Shop is blocked while a customer has already selected it (`awaiting_selection`/`selected`) | Not retroactively unwound — §3.8's auto-cancel applies to `open` requests; a request already past selection is a completed real-world interaction the block doesn't reach back into |
| Admin wants to reverse a block | **No in-app path** (§3) — must be handled outside this page (DB/support) until an unblock endpoint exists |

## 10. Accessibility
- Required reason field: `aria-required`, error state `aria-invalid` if submit attempted empty.
- Block button `aria-disabled` until valid; explicit confirmation step (this whole page IS the confirmation — no additional double-confirm needed since reason-entry already adds friction).
- 44×44px min touch targets.

## 11. Admin/Data Management
This page **is** the enforcement action itself — no further approval layer above admin. Every block should be treated as final/logged (via `rejectionReason` + implicit `updatedAt`) given the missing-unblock gap (§3).

## 12. Component Inventory
| Component | States |
|---|---|
| BlockShopForm | empty, valid, submitting, error |
| Toast (reused) | blocked, error |

---
*— End of condensed spec —*
