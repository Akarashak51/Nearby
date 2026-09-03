# NEARBY — Page Architecture, Derived from Blinkit / Zepto / Zomato Analysis
### Companion to Master Build Document v2.6.2 — maps Section 6.1's three route trees (`/app/*`, `/shop/*`, `/admin/*`) into a full page list

---

## 0. Why this isn't a straight copy of Blinkit/Zepto/Zomato

Before listing pages, one thing has to be said clearly: **NEARBY is not a cart-and-checkout app.** Blinkit, Zepto, and Instamart are quick-commerce (cart → checkout → delivery). Zomato is food-delivery (cart → checkout → delivery) plus a separate dining/reservations product. NEARBY has **no cart, no payment, no delivery** — a customer broadcasts a need, shops answer Yes/No, the customer picks one and physically walks in (Section 1, Master Build Doc).

So the useful thing to borrow from those apps isn't their checkout flow — it's:
- **How they structure discovery, search, and the home screen** (customer app)
- **How they structure the merchant/partner side's order-response and business-management screens** (shop dashboard)
- **How they structure operator-side approval, monitoring, and analytics** (admin console)

Below, each NEARBY page is tagged with **[Inspired by: X]** where a competitor pattern applies, and **[NEARBY-specific]** where the broadcast model requires something none of those apps have.

### What was researched
- **Blinkit** — customer app UX (home screen, search, category page, cart, reorder flow)
- **Zepto** — customer app UX (home, categories, buy-again, live order tracking) + **Zepto Franchise app** (dark-store operator app: performance KPIs, earnings, staffing) + general quick-commerce admin-panel patterns (store/driver/customer management)
- **Zomato** — **Restaurant Partner app** (order management, menu/inventory, outlet management, staff, business analytics/payouts, offers, help center, rush-hour, leave scheduling) + Zomato Dining Partner (reservation accept/reject, owner's hub)

---

## 1. Customer App — `/app/*`
*Orange-forward surface (Brand Guide §3.3). Role: `customer`.*

| # | Page | Purpose | Pattern source |
|---|------|---------|-----------------|
| 1.1 | **Landing / Home** | Big search-style CTA ("What are you looking for?"), recent requests, nearby-shop-count teaser, "Buy Again"-style **Repeat Request** shortcuts to past searches | [Inspired by: Blinkit/Zepto home screen — sticky search, "Buy Again" carousel] |
| 1.2 | **Sign up / Log in** | Email+password local auth, Google Sign-In button (Section 7), phone field for post-approval contact | [NEARBY-specific: no OTP-gated signup, admin verifies phone manually] |
| 1.3 | **New Broadcast (Search) screen** | Product/need text field, radius selector, location capture (auto-geolocation with manual-address fallback per Section 6.2), "Broadcast" submit button | [Inspired by: Blinkit search bar + Zepto smart search] but the result isn't a product list — it's a request |
| 1.4 | **Location confirm / manual address** | Shown only on geolocation denial/failure — text input → Nominatim lookup → confirm pin on mini-map | [NEARBY-specific — no competitor has this because they all assume delivery-address-on-file] |
| 1.5 | **Request-in-progress (Live Count) screen** | Countdown timer to `expiresAt`, live "X shops have answered" counter via `response:update`, cancel button | [Inspired by: Zepto/Blinkit live order tracking — "your order is being prepared" state machine] but tracking a *broadcast*, not a delivery |
| 1.6 | **Shortlist / Awaiting Selection screen** | Ranked list of Yes-shops (distance band → rating → reliability, Section 3.1.1), each card shows shop name, distance, avgRating, reliabilityScore, open/closed badge | [Inspired by: Zomato restaurant list cards — name, rating, distance, cuisine tags] mapped onto shops-that-said-yes instead of restaurants |
| 1.7 | **Shop profile (from shortlist)** | Full shop card: photo, address, phone, hours, rating breakdown | [Inspired by: Zomato restaurant detail page / Blinkit store info] |
| 1.8 | **Selection confirmation screen** | "You picked X" + shop contact + address + directions link; other Yes-shops silently notified they weren't picked | [Inspired by: Zepto "order confirmed" screen] |
| 1.9 | **Expired / No-response screen** | Two copy variants per `yesCount` (Section 6.3): zero-Yes → re-broadcast at larger radius CTA; timed-out-with-responses → plain re-search CTA | [NEARBY-specific — no delivery app has a "nobody answered" state] |
| 1.10 | **Cancelled screen** | Distinguishes customer-cancel vs. auto-cancel-on-suspension copy | [NEARBY-specific] |
| 1.11 | **Post-visit Feedback screen** | Found / Not-found toggle + 1–5 star rating, shown once after a `selected` request is presumed visited | [Inspired by: Zomato/Blinkit post-delivery rating prompt] |
| 1.12 | **Request History** | List of past requests with status chips (selected/expired/cancelled), tap-through to detail | [Inspired by: Blinkit/Zepto order history] |
| 1.13 | **Direct Shop Search** (FR-1.15) | Search approved shops directly by name/category without broadcasting — for a customer who already knows who they want | [Inspired by: Zomato's direct restaurant search, as distinct from "order food now" discovery] |
| 1.14 | **Push notification opt-in / settings** | Enable/manage the push subscription that now has a real consumer (v2.6.2 fix) | [Inspired by: standard app notification-permission prompt, generalized] |
| 1.15 | **AI Assistant chat** | Platform-scoped chat, in-scope/out-of-scope guardrail (Section 7) | [NEARBY-specific — none of the three competitors ship a data-scoped chat assistant as a core page] |
| 1.16 | **Report a shop** | File a report (feeds `report:new` → admin room) | [Inspired by: Zomato "report an issue" on order] |
| 1.17 | **Profile / Account settings** | Name, phone, linked-auth-provider display, "Become a shop" entry point (promotes role, Section 3.3) | [Inspired by: standard account page; "become a partner" CTA pattern from Zomato/Blinkit customer apps] |

---

## 2. Shop Dashboard — `/shop/*`
*Teal-forward surface. Role: `shop`. Directly modeled on the Zomato Restaurant Partner app and Zepto Franchise app — this is the strongest 1:1 mapping of the three surfaces, since "manage incoming orders + outlet profile + business stats" is exactly what those two apps do.*

| # | Page | Purpose | Pattern source |
|---|------|---------|-----------------|
| 2.1 | **Shop onboarding / Profile creation** | Shown if `GET /shops/me` returns none (v2.6.1): shop name, category, address (geocoded), phone, storefront photo *or* proof-of-business doc | [Inspired by: Zomato restaurant onboarding — outlet name, address, FSSAI/legal doc upload] |
| 2.2 | **Pending-approval screen** | Static "under review" state, shown while `status: pending` | [Inspired by: Zomato "your outlet is under review"] |
| 2.3 | **Rejected / Re-apply screen** | Shows `rejectionReason`, lets owner resubmit the same field set (v2.6.1 fix) | [NEARBY-specific — Zomato doesn't expose a structured re-apply flow this explicitly, but the pattern of "resubmit corrected info" is standard marketplace-onboarding UX] |
| 2.4 | **Live Dashboard / Home** | Open/closed toggle (front and center, Zepto-franchise-KPI-style), today's Yes/No count, current reliability score, incoming-request feed | [Inspired by: Zomato Partner "live order management" home tab + Zepto Franchise "performance visibility" KPIs] |
| 2.5 | **Incoming Request card / detail** | Product asked, distance, countdown to respond, Yes/No buttons; falls back to 5s polling on socket drop (v2.6.1) | [Inspired by: Zomato Partner "accept/reject order" screen — but answering *availability*, not *fulfillment*] |
| 2.5b | **Rush-hour toggle** | Temporarily pause new incoming requests / extend response time during a busy patch | [Inspired by: Zomato Partner "Rush hour" feature, adapted from kitchen-prep-time to answer-time] |
| 2.6 | **Outlet Management** | Edit shop name, category, phone, opening hours (timezone-aware, Section 3.4) | [Inspired by: Zomato Partner "Outlet management" — name, address, timings] |
| 2.7 | **Opening Hours editor** | Per-day hours grid, feeds the `$nearSphere` + working-hours filter (Section 3.1) | [Inspired by: Zomato Partner outlet-timings screen] |
| 2.8 | **Reliability & Ratings view** | `reliabilityScore`, `avgRating`, `foundCount`/`notFoundCount` breakdown, formula explainer (Section 3.1.2) | [Inspired by: Zomato Partner "business reporting" / Zepto Franchise "Order Breaches, Defects" KPI tiles] |
| 2.9 | **Request History (shop side)** | Past requests the shop answered, outcome (selected/not-selected/expired), no penalty shown for un-selected Yes (Section 3.1.3) | [Inspired by: Zomato Partner order history] |
| 2.10 | **Staff management** *(if multi-user shop accounts are ever needed)* | Add/remove staff who can toggle open/closed or answer requests | [Inspired by: Zomato Partner "manage staff: add/delete/invite"] — flagged as a *possible* future page, not in current FR list; include only if the team wants parity |
| 2.11 | **Push notification settings** | Manage the shop-side push subscription (already has a consumer since Sprint-1) | [Inspired by: standard partner-app notification settings] |
| 2.12 | **Help Centre / Raise a ticket** | Simple contact-admin form | [Inspired by: Zomato Partner "Help centre — raise a ticket"] |
| 2.13 | **Leave / Temporary closure scheduler** | Mark shop closed for a date range (festival, personal) without losing `approved` status | [Inspired by: Zomato Partner "Schedule leaves in advance"] |

---

## 3. Admin Console — `/admin/*`
*Navy-forward surface, orange reserved for destructive actions (Brand Guide §3.3). Role: `admin`. Modeled on the generic quick-commerce/marketplace operator panel (store approval, driver-equivalent-N/A-here, customer management, analytics) since none of the three competitor apps expose their internal admin tools publicly — this section leans on typical hyperlocal-marketplace admin-panel structure plus what NEARBY's own FRs (Section 2, 3.3, 3.8) explicitly require.*

| # | Page | Purpose | Pattern source |
|---|------|---------|-----------------|
| 3.1 | **Admin login** | Separate from customer/shop auth, no public signup (Section 3.5A bootstrap only) | [Standard operator-panel pattern] |
| 3.2 | **Dashboard / Overview** | Pending-shop count, open-report count, today's request volume — the one screen every admin panel leads with | [Inspired by: generic quick-commerce admin home — "stores pending approval," "active orders," "flagged items"] |
| 3.3 | **Shop Approval Queue** (`GET /admin/shops`) | List of `pending` shops, each opens to profile + photo/doc + address-plausibility check (Section 3.3 criteria a–d), Approve/Reject with reason | [Inspired by: Zomato's internal restaurant-onboarding review + generic dark-store approval workflow (Zepto-clone admin panels reviewed above explicitly list "manage stores")] |
| 3.4 | **All Shops directory** | Every shop regardless of status, filter by status/category, drill into any shop's full profile and stats | [Inspired by: standard "manage stores" admin table] |
| 3.5 | **Block / Suspend shop** (`PATCH /admin/shops/:id/block`) | Distinct from reject — for an already-approved shop that needs removing; auto-cancels its pending requests (Section 3.8) | [NEARBY-specific enforcement page, standard marketplace-moderation pattern] |
| 3.6 | **Reports queue** (`GET /admin/reports`) | Customer-filed reports (`report:new`), triage and resolve | [Inspired by: standard trust & safety queue, present in every marketplace admin panel] |
| 3.7 | **Analytics dashboard** (`GET /admin/analytics`) | Read-only, `windowDays`-bounded aggregation (Section 8.4): request volume, fulfillment rate, avg response time, top categories | [Inspired by: Zomato Partner's own "business reporting" tile set, and Zepto Franchise's KPI dashboard — reused at the platform level instead of per-outlet] |
| 3.8 | **Customer directory** *(lightweight)* | Search/view customer accounts for support purposes; no edit beyond support actions | [Inspired by: generic "manage customers" admin table] |
| 3.9 | **Admin bootstrap / first-admin setup** (FR-1.13) | One-time flow to create the first admin account | [NEARBY-specific] |

---

## 4. Shared / cross-cutting pages (all three surfaces)

| Page | Where it lives | Notes |
|---|---|---|
| Toast / error-boundary handling | Global | Section 6.1 — one shared handler for 400/401/403/404/409/429 |
| 404 / route-guard redirect | Global | Non-admin hitting `/admin/*` → redirected to `/app` |
| Offline / socket-reconnect banner | `/app/*` and `/shop/*` | Surfaces the 5s-poll fallback state (Section 6.3) so users know they're on the REST fallback, not just silently stale |
| PWA install prompt | `/app/*` primarily, all surfaces technically installable | Brand/PWA Guide Section 5 |

---

## 5. What was deliberately *not* borrowed

- **Cart / checkout / payment pages** — no cart exists in NEARBY (Section 1: payments explicitly out of scope for v1.0)
- **Delivery-partner / rider app** — NEARBY has no delivery leg; the customer physically visits the shop
- **Menu/inventory-item-level pages** (Zomato's dish-by-dish menu editor, Blinkit's SKU catalog) — NEARBY shops answer Yes/No to a *broadcast query*, they don't maintain a browsable product catalog in v1
- **Coupons / offers / ads management** (Zomato Partner's "Offers & Ads") — no monetization layer in v1.0

---

*This document maps 1:1 against Master Build Document v2.6.2 Section 6 (Frontend Architecture & UX Flows) and the Brand/PWA Guide's per-surface accent system (Section 3.3). Every page above should carry the surface's designated accent color and Inter typography per the Brand Guide.*
