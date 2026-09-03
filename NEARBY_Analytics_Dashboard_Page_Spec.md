# NEARBY — Admin Console: Analytics Dashboard
### Condensed Page Spec — v1.0
**Refs:** Master Build Doc v2.6.2 (§5.4, §6.4, §8.4) · Brand & PWA Guide v1.0 · Page Architecture (§3.7)
**Surface:** `/admin/*` — Navy-forward · **Role:** admin

---

## 1. Overview
Read-only aggregate pilot-health metrics (FR-3.8, UC-18) — "so admin can see pilot health at a glance rather than inferring it manually from the reports and shops lists" (§3.5). Pattern-sourced from Zomato Partner's "business reporting" tile set and Zepto Franchise's KPI dashboard, reused at platform level instead of per-outlet (Page Architecture §3.7).

**Architecturally unique among admin pages:** "the one dashboard screen with no direct 1:1 write-side counterpart" (§6.4) — pure read-only, **refreshed on manual pull, not sockets** (§6.4), consistent with its 30-req/hour rate limit (§8.4). No real-time updates here by design — this page must never imply live-ness it doesn't have.

## 2. Functional Spec — exact 4 metrics, no more, no less (§5.4)
`GET /api/v1/admin/analytics?windowDays=7` (default 7, max 90 — bounds cost on free-tier M0, §8.4). All computed on read via aggregation, **no new collection** (§5.4 note).

| Metric | Definition | Notes |
|---|---|---|
| **fulfilmentRate** | % of requests in window ending `selected`, out of requests that closed with ≥1 Yes | Single headline % |
| **activeShops** | Count `approved` + `isOpen: true` shops right now, **plus** sub-count that got ≥1 response in the window | Two numbers, not one — "right now" vs. "active in window" are different claims and must be labeled distinctly |
| **avgBroadcastReach** | Avg `shopResponses` per request in window — reported **with p50/p90 alongside the mean**, not mean alone | Three numbers per this metric — the spec explicitly requires percentiles, not just an average |
| **topRequestedProducts** | Top 10 `productQuery` values by count, case-folded/trimmed | List, paginated beyond 10 (§5.2) |

- Rate limit: 30/admin/hour (§8.4) — this page's manual-refresh button must respect it; no auto-poll ever.
- Performance target: <2s response at 90-day window, pilot scale (§8.4 non-functional req) — no special loading choreography needed beyond a standard spinner given this bound.
- **Admin Insights (optional, Gemini Flash, §7.4):** plain-language summary of the payload — e.g. "Fulfilment is up 8% this week, mostly driven by faster shop responses." Shown as a small card, clearly labeled AI-generated, never replacing the raw numbers.

## 3. Where It Lives
Admin nav → "Analytics." No sub-navigation — one scrollable page, windowDays selector at top affects everything below it.

## 4. Section-by-Section

| Element | Copy | Type role | Notes |
|---|---|---|---|
| Page header | "Analytics" | H1 | Ink Navy |
| Window selector | "Last 7 days" / "30 days" / "90 days" (chips) | Button/SemiBold | Default 7d; re-fetches all metrics on change |
| Manual refresh | "Refresh" (icon + label) | Button, outline | **No auto-refresh, no live socket** — must be an explicit tap (§6.4) |
| Last-updated stamp | "Updated {time}" | Caption | Gray 600 — reinforces this is a snapshot, not live |
| **Card — Fulfilment rate** | "{X}% fulfilled" + "{N} of {M} requests with ≥1 Yes ended in a selection" | Display numeral, Caption sub-line | Headline metric, largest visual weight |
| **Card — Active shops** | "{N} open right now" + "{M} responded to a request this window" | Body numeral ×2, Caption labels | Two numbers, explicitly separate labels (§2) — never merged into one figure |
| **Card — Broadcast reach** | "{X} shops per request on average" + "Median {p50} · 90th percentile {p90}" | Body numeral, Caption sub-line | All three numbers shown together, not mean-only |
| **Card — Top requested products** | Ranked list, "{product} — {count} requests" | Body list | Top 10, "See more" paginates beyond |
| AI Insights card (optional) | "{plain-language summary}" | Body, small "AI-generated" tag | Gemini Flash output — Gray 100 background to visually distinguish from hard data cards |
| Rate-limit notice (if hit) | "You've refreshed a lot — try again in a few minutes." | Caption, Danger | Only shown on 429 |

## 5. Visual Design
- **Colors:** Ink Navy (header, active window-chip, card accents), Gray 900 (numerals), Gray 600 (labels/captions). **No Orange, no Teal, no Warning/Danger** except the rate-limit notice — this is a neutral reporting page, no status/enforcement color language needed since nothing here is actionable or urgent.
- **Background:** White cards on Gray 100 — same admin-console pattern as every other page in this series.
- **Cards:** 16px radius, 1px Gray 300 border, generous internal padding (20px) since numerals need room to breathe — this is the one admin page where Display-scale type (Brand Guide §4.2, "rare") is justified, on the Fulfilment Rate headline number specifically, mirroring how the shop-side Reliability & Ratings view uses Display for its own headline score.
- **Layout:** Single column on mobile, 2-column grid on wider viewports — cards don't need to be equal-weighted (Fulfilment Rate can be visually largest, matching its role as the one number "every admin panel leads with" framing from the Dashboard/Overview page, §3.2).

## 6. Graphics / Images / Video
- **Simple bar or sparkline chart is appropriate for Top Requested Products** (ranked bars) — the one place a chart adds real value over a plain list, since relative magnitude between the top few products is genuinely useful at a glance.
- No illustration, photo, or video anywhere else — this is a numbers page.
- p50/p90 for broadcast reach can optionally be shown as a small range/distribution bar rather than plain text, but plain numerals are sufficient and lower-risk to build correctly — recommend text-first, chart as a v1.1 enhancement.

## 7. Animation
- Window-chip switch: duration-instant (100ms), triggers full re-fetch (loading state on all cards simultaneously, not staggered).
- Refresh tap: standard button press + spinner replacing the "Refresh" label until data returns.
- Numeral updates on refresh: **no animated count-up** — instant value swap, same explicit rule as every score/numeral across this series (Reliability & Ratings, Live Dashboard) — dramatizing movement in a metrics dashboard risks implying a trend that a single refresh can't actually establish.
- Top-products list re-order (if ranks shift between refreshes): instant reflow, no slide/reorder animation — avoids implying more precision/drama than a weekly aggregate warrants.

## 8. States & Edge Cases

| Case | Behavior |
|---|---|
| Pilot has too little data (e.g. day 1) | Cards show real zeros/low numbers plainly — no fake "not enough data yet" empty state, since these are legitimate (if small) aggregates, not a broken feature |
| 429 rate limit hit on refresh | Refresh button disables temporarily, rate-limit notice shown (§4) |
| Query exceeds 2s target (large window, degraded free-tier performance) | Standard loading spinner persists; no special timeout messaging needed unless it hard-fails, in which case standard error toast ("Couldn't load analytics. Try again.") |
| AI Insights fails/unavailable | Card is simply omitted — the hard-data cards are the source of truth and function fully without it; never blocking on the AI summary |
| windowDays changed to 90 on slow connection | Same loading treatment, just potentially longer — no special UI needed given the 2s target already bounds this at pilot scale |

## 9. Accessibility
- Numerals paired with their text labels always — a screen reader must get "87 percent fulfilment rate," not a bare "87."
- Window-selector chips use `role="tab"`/`aria-selected`.
- Manual-refresh button clearly labeled, not icon-only.
- Chart (top products) has an accessible text-table equivalent alongside it, not chart-only data.
- 44×44px min touch targets.

## 10. Admin/Data Management
This page has no write path at all (§1) — it is purely a consumption surface over data admin already governs elsewhere (Shop Approval, Reports, Block/Suspend). No admin-of-admin layer applies.

## 11. Component Inventory
| Component | States |
|---|---|
| AnalyticsWindowSelector | 7d/30d/90d active |
| MetricCard (×3: fulfilment, active shops, broadcast reach) | loaded, loading, error |
| TopProductsCard | loaded, loading, paginated |
| AiInsightsCard | visible, hidden (unavailable) |
| Toast (reused) | error, rate-limited |

---
*— End of condensed spec —*
