# NEARBY — Shop Onboarding / Profile Creation Screen Build Specification
### Everything needed to build `POST /shops` — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.3, 4.2, endpoint table §5), NEARBY Brand/PWA Guide v1.0 (§1.4, 3.3), and simple-onboarding-form UX research (Stripe/Netflix's short-form signup pattern, merchant-onboarding-checklist research)

---

## 0. Scope — the first screen in an entirely different visual register, and who it's actually built for

This is the first screen in this build series to belong to the **Shop Dashboard surface**, and one specific line from your Brand Guide governs every decision below more than any other: **"shop-facing copy is the shortest and most action-oriented of the three surfaces, since FR requirements assume low technical proficiency"** (§1.4). This isn't a customer casually browsing — it's a shopkeeper, possibly filling this out for the first time on a phone, filling in their own business details to get found by nearby customers. Every field, every instruction, every confirmation on this screen should read like it was written for someone in the middle of a workday, not someone evaluating a product.

**The accent shift, stated plainly since it affects every color decision from here on:** per Brand Guide §3.3, the Shop Dashboard is **Teal-forward** — Signal Teal (`#00B8A9`, the same hex the customer surface uses for "Success") replaces orange as the dominant accent on nav/active states, with **orange reserved exclusively for the Yes/No response buttons** elsewhere in the shop dashboard, so that one specific action never visually competes with anything else. This screen has no Yes/No buttons on it — it's the onboarding form, not the response screen — so **this entire screen should be orange-free**, using Signal Teal for its primary action instead. This is the first genuine color-system departure in this whole build series, worth calling out clearly rather than assuming the customer-side orange convention carries over.

**Backend facts that shape the form directly (§3.3, §4.2, endpoint table):**
- Fields required at creation: `shopName`, `category`, `address`, `phone`, and **exactly one of** `photo` or `document` — never both, never neither.
- `timezone` is a stored field (§4.2, IANA format) but nothing in the endpoint signature shows it as a submitted field — the sensible, low-friction read is that the client should **auto-detect and send it silently** (`Intl.DateTimeFormat().resolvedOptions().timeZone`) rather than asking a shopkeeper to pick their own timezone from a dropdown, which would violate the "low technical proficiency" principle for a value the browser already knows.
- File uploads are capped at 5MB per file (§3, Multer/Cloudinary handling) — enforced client-side before upload, not just server-side, so a shopkeeper on a slow connection isn't left waiting to be told their file was too big.
- Approval criteria (§3.3: reachable phone, plausible/non-duplicate address, non-duplicate photo/document, category-context consistency) are all **admin-side checks**, not things this screen validates itself — but the form can still gently improve the odds of a smooth approval by confirming the address on a map before submission, the same pattern already built for the customer-side Location-Confirm screen.

---

## 1. Page goals, in priority order

1. Get a real shopkeeper through this form quickly, on a phone, without confusion — per the short-form-signup research reviewed (Stripe's own onboarding: as few fields as the task allows, one step, not a wizard).
2. Make the photo-or-document choice unambiguous — this is the one part of the form most likely to trip someone up if it isn't crystal clear that only one is needed.
3. Confirm the address visually before submitting, directly improving the odds of a smooth admin approval (criterion b) without adding a whole separate step.
4. Set honest expectations about what happens next and how long it takes, given approval here is a **manual admin phone callback**, not instant (§3.3) — a shopkeeper should never wonder why nothing happened right after submitting.

---

## 2. Layout overview

Single scrollable screen, one continuous form (not a multi-step wizard) — per Goal 1, this task is short enough that breaking it into several screens would add friction without adding clarity.

```
┌─────────────────────────────┐
│ [A] Header (title)             │
├─────────────────────────────┤
│ [B] Welcome / reassurance line  │
├─────────────────────────────┤
│ [C] Shop name field              │
├─────────────────────────────┤
│ [D] Category selector            │
├─────────────────────────────┤
│ [E] Address field + map confirm  │
├─────────────────────────────┤
│ [F] Phone field                  │
├─────────────────────────────┤
│ [G] Photo / document upload      │
├─────────────────────────────┤
│ [H] Submit button                 │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, no back-chevron on first entry (a brand-new user who just chose "I run a shop nearby" at signup, per the Sign-Up screen's account-type toggle, has nowhere meaningful to go back to yet) — but if this screen is reached via the Home page's "Become a shop" card from an existing customer account, a back-chevron does appear, returning to Home.
**Copy:** "Set up your shop"
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Welcome / reassurance line
**Purpose:** directly implements Goal 4's front-loaded honesty and the "shortest, most action-oriented" tone rule — this single line does the job a longer explanation might otherwise need.
**Copy:** "Takes about 2 minutes. We'll review your details and call to confirm before you go live." — short, sets time expectations, and honestly previews the phone-callback approval step (§3.3 criterion a) so it isn't a surprise later.
**Layout:** plain text, Body size, Text-secondary, sits directly below the header, no card/background treatment — this is a lead-in sentence, not a callout.
**Animation:** none.

### [C] Shop name field
**Copy (label):** "Shop name"
**Placeholder:** "e.g. Sharma General Store"
**Layout:** full-width input, 48px height, 12px radius, Surface white background, 1px Border-gray outline — identical visual treatment to every text input in this series, just re-colored for this surface (see Section 4).
**Validation:** required; empty-submit shows "Enter your shop's name" in Danger red beneath the field, same pattern as the customer-side New Broadcast screen's product-query field.
**Color:** default border Border-gray; **focus border Signal Teal 2px** — the first of many places on this screen where Teal replaces the Orange used for the equivalent customer-side control.
**Animation:** border-color transition on focus, `duration-instant` (100ms), `ease-standard`.

### [D] Category selector
**Copy (label):** "What kind of shop is this?"
**Layout:** a simple single-select list of large, plainly-labeled options (not a dropdown menu, which research on low-technical-proficiency users flags as harder to discover/operate than a visible list, and not small chips, which read as fussier than this form's tone calls for) — each option a full-width row, 48px height, radio indicator, 1px Border-gray divider between rows.
**Option set:** Groceries · Medicines · Electronics · Hardware · Stationery · Mobile Accessories · Bakery · Household · **Other** — the exact same category set already established on the customer-side Home page's Quick Category chips and Direct Shop Search screen, kept deliberately identical so a category picked here is guaranteed to match what a customer might filter by later (§4.2 documents `category` as free text with *examples*, not a strict enum, so this UI choice constrains it to a known-good, consistent set rather than leaving it open to typos or inconsistent phrasing that would silently break the customer-side category-filter matching).
**"Other" behavior:** selecting it reveals a small inline text field, "Tell us what kind of shop" — free text, required only if "Other" is the active selection.
**Color:** selected row's radio indicator filled Signal Teal; unselected Border-gray outline only.
**Animation:** radio-fill transition on selection (`duration-instant`); "Other" text field expands with a brief height-transition (`duration-quick`, 180ms, `ease-out-soft`).

### [E] Address field + map confirm
**Purpose:** the single most consequential field for approval odds (criterion b) — a plausible, geocodable, non-duplicate address.
**Copy (label):** "Shop address"
**Placeholder:** "Street, area, or landmark"
**Layout:** text input identical to [C], with an inline "Confirm on map" text-button (Caption, Signal Teal) appearing once the field has content.
**Behavior:** tapping "Confirm on map" opens the **same Leaflet/OpenStreetMap confirmation component already built for the customer-side Location-Confirm screen** — fixed center-pin, moving map, reverse-geocode-on-pan-settle, `POST /geo/lookup` under the hood — reused directly rather than rebuilt, since the underlying task (turn free text into a confirmed lat/lng) is identical on both sides of the product. The one difference in copy: the confirm button here reads **"This is my shop's location"** rather than the customer-side "Use this location," since the framing is "confirm where your business is," not "confirm where you are right now."
**Why this step earns its place on an otherwise-short form:** it directly, visibly improves criterion (b)'s approval odds — an address the shopkeeper has personally confirmed on a map is far less likely to geocode to an implausible or ambiguous point than raw free text alone, and catching that at submission time is far better for a shopkeeper than discovering it via a rejection days later.
**Color:** input styled identically to [C]; "Confirm on map" link and the map-confirm screen's own pin/UI elements all recolored to **Signal Teal** in place of the customer-side screen's Orange (the map-confirm component itself is otherwise visually identical — same pin shape, same OSM tiles, same "drag the map to adjust" hint copy — only the accent color changes to match this surface).
**Animation:** identical to the customer-side Location-Confirm screen's own motion spec (map pan/zoom via Leaflet's built-in easing, confirmation-card text cross-fade on settle) — reused, not redesigned.

### [F] Phone field
**Copy (label):** "Contact phone number"
**Helper text beneath (Caption, Text-secondary):** "We'll call this number to confirm your shop before approving it" — directly and plainly previews criterion (a)'s manual callback, removing any ambiguity about why a phone call might come later.
**Layout:** standard phone-formatted input, identical visual treatment to [C].
**Validation:** required; basic format check (not exhaustive international validation, given this is a single-locality pilot per Master Build Doc §1) — "Enter a valid phone number" in Danger red on failure.
**Color/animation:** identical pattern to every other input on this screen.

### [G] Photo / document upload — the section most likely to cause confusion if handled poorly
**Purpose:** exactly one of `photo`/`document` is required (§4.2) — the UI's job is to make that "exactly one, either is fine" framing completely unambiguous.
**Copy (label):** "Add a photo of your shop, or a business document"
**Helper text beneath:** "Just one of these is enough — whichever's easier for you." — directly states the "either/or, not both" rule in plain language, addressing Goal 2 head-on.
**Layout:** two large, equal-width tappable upload cards side by side (stack vertically below ~340px width), each a dashed-border rounded rectangle (12px radius, 100px height), an icon + short label inside.
- **Left card — "Take/upload a photo":** camera icon, opens the device's native camera or photo-library picker (`<input type="file" accept="image/*" capture>` — offering the camera directly is a meaningful convenience for a shopkeeper who may not have an existing photo on their phone).
- **Right card — "Upload a document":** document icon, opens a file picker accepting PDF/image formats (`accept="application/pdf,image/*"`) for a proof-of-business document (e.g. a trade license, GST certificate, or similar — the exact accepted document types aren't enumerated in the source doc, so this UI stays deliberately generic ("a business document") rather than listing specific document types the team hasn't confirmed).
**Mutual exclusivity behavior:** once one card has a file attached, the other card visually dims (~50% opacity) and becomes non-interactive, with a small note appearing: "Remove this to upload the other instead" — enforced in the UI itself, not just left to a validation error after the fact, so a shopkeeper never gets partway through attaching both before being told not to.
**Attached-file state:** the active card swaps its icon/label for a small thumbnail preview (photo) or a filename + document icon (document), plus a small "×" remove control.
**Client-side size validation:** any file over 5MB is rejected immediately at selection with a plain inline message: "That file's a bit too large — try a smaller photo or a lower-resolution scan." (Danger red, but phrased helpfully rather than as a technical error like "exceeds 5MB limit") — checked before any upload attempt begins, so the shopkeeper never waits through a failed upload to find out.
**Color:** dashed-border cards in Border-gray by default; **Signal Teal** dashed border once a file is attached (a light "this is filled in correctly" signal, consistent with how filled/valid form sections are marked elsewhere in this product, just recolored for this surface).
**Animation:** file-attach swaps the card's content with a brief cross-fade (`duration-quick`, 180ms); the dimming of the unused card transitions smoothly (`duration-instant`) rather than snapping, so the "why did that just gray out" moment reads as an intentional state change, not a glitch.

### [H] Submit button
**Layout:** fixed to the bottom of the screen, full-width minus 16px margins, 48px height, 12px radius, sits above the safe-area inset.
**Copy:** "Submit for review" — deliberately not "Create shop" or "Go live," since neither is true yet at this step (§3.3: the shop starts `pending` and is invisible to customers until approved) — this copy sets the correct expectation about what tapping it actually does.
**States:** disabled (Gray 300) until `shopName`, a category selection, a confirmed address, `phone`, and exactly one of photo/document are all present; enabled (filled **Signal Teal**, White Button-weight text) otherwise; loading (multipart upload + `POST /shops` in flight) shows a centered spinner — and, given this request includes a file upload, a determinate progress indicator is worth the small extra effort here specifically (a thin progress bar beneath the button, filling as the upload progresses) rather than an indefinite spinner, since file uploads on a possibly-slow mobile connection can take long enough that "is this stuck?" becomes a real concern (a concern basically absent from every prior mostly-text form in this series).
**On success:** transitions to the **Pending-approval screen** (page-architecture item 2.2) — a separate, already-scoped screen in this product's structure, not built out further here.
**Animation:** tap-scale-to-0.97; on success, standard cross-fade (`duration-standard`, `ease-out-soft`) to the Pending-approval screen.

---

## 4. Color usage summary (Brand Guide §3.1/3.3 — no new hex introduced, but usage reassigned for this surface)

| Token | Hex | Used where on this page |
|---|---|---|
| **Signal Teal** (same value as "Success" elsewhere) | `#00B8A9` | Focus borders on every input, selected category radio, "Confirm on map" link, map-confirm component's pin/accents, attached-upload-card border, Submit button |
| Danger | `#E5484D` | Validation errors (empty required fields, file-too-large, invalid phone format) |
| Text primary | `#14213D` | Header title, field labels, input text |
| Text secondary | `#5B6472` | Welcome line, helper text, placeholder text |
| Border | `#D8DCE3` | Default input/card outlines, category-row dividers |
| Surface | `#F4F5F7` / `#FFFFFF` | Page background (Gray 100) / inputs, cards, header (White) |

**No Primary orange anywhere on this screen** — per Section 0's surface-accent rule, orange is reserved exclusively for the Yes/No response buttons elsewhere in the Shop Dashboard, and this screen has none, so it should contain precisely zero orange pixels; every place a customer-side equivalent screen would use orange, this screen uses Signal Teal instead.

---

## 5. Typography summary (Brand Guide §4.2 — unchanged type scale, just applied here)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Welcome/reassurance line | Body / 15px | 400 Regular |
| Field labels | Body / 15px | 400 Regular |
| Category row labels | Body / 15px | 400 Regular |
| Helper text (phone, upload section) | Caption / 13px | 400 Regular |
| Upload-card labels | Body / 15px | 400 Regular |
| Submit button label | Button / 15px | 600 SemiBold |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Input focus borders, radio selection, card-dimming on mutual-exclusivity, tap-scale feedback |
| `duration-quick` | 180ms | "Other" category field expand, upload-card content cross-fade on attach |
| `duration-standard` | 240ms | Screen-exit cross-fade to Pending-approval on submit |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Focus/selection transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Expand transitions, screen-exit |

**No looping animation anywhere on this screen** — this is a calm, procedural form, same reasoning as every non-live screen in this series.
**Reduced motion:** covered by the app-wide `prefers-reduced-motion` query.

---

## 7. Images, icons & video — asset list

**No video, no stock photography** — the only "photo" on this screen is whatever the shopkeeper uploads themselves.

| Asset | Format/size | Source |
|---|---|---|
| Camera icon (upload card) | Line icon, 24×24 | Icon library (same one used app-wide) |
| Document icon (upload card) | Line icon, 24×24 | Icon library |
| Remove ("×") icon | Line icon, 16×16 | Icon library |
| Map-confirm component assets | Same as the customer-side Location-Confirm screen's asset list (OSM tiles, pin shape derived from `nearby-mark.svg`), recolored to Signal Teal | Reused |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| All fields | Client-side form state until submit |
| Timezone | Auto-detected via `Intl.DateTimeFormat().resolvedOptions().timeZone`, sent silently alongside the form (Section 0) — **flagged as a documented assumption**, since the endpoint's documented body (`{ shopName, category, address, phone, photo | document }`) doesn't explicitly list `timezone` as a submitted field even though the schema stores one; the team should confirm whether the server expects the client to send this or derives/defaults it another way |
| [E] Address confirm | `POST /api/v1/geo/lookup` (Public, rate-limited 10/IP/hour, §5.4) — identical usage to the customer-side Location-Confirm screen |
| [H] Submit | `POST /api/v1/shops` (Bearer, customer or shop role — §3.3/endpoint table) as `multipart/form-data`: `{ shopName, category, address, phone, photo }` or `{ ..., document }` — never both. On success from a `customer`-role caller, the server promotes that account's role to `shop` in the same transaction (§3.3's v2.6.1 fix) |

**Loading state:** the Submit button's own progress-bar treatment (Section 3[H]) is this screen's only loading state — no skeletons needed, since nothing on this screen depends on a prior fetch (it's a pure creation form).
**Error state:** a failed submission (network/500, not a validation issue) uses the shared toast/error-boundary pattern (§6.1) with all entered form data — including the attached file — preserved for retry, since asking a shopkeeper to re-select a photo/document after a failed submit would be a genuinely frustrating regression on a mobile upload.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with the rest of this product; the two upload cards in [G] sit side-by-side down to a reasonably narrow width before stacking (Section 3[G]).
- **Low-bandwidth/slow-connection consideration:** the determinate upload progress bar (Section 3[H]) matters more on this screen than almost anywhere else in the product, given file uploads are meaningfully more bandwidth-sensitive than the text-only submissions everywhere else — worth the small extra implementation effort specifically here.
- **Camera capture:** offering the device camera directly (Section 3[G]'s `capture` attribute) rather than only a gallery picker is a small but genuinely helpful convenience for a shopkeeper photographing their storefront in the moment, rather than needing an existing photo already saved.
- **PWA standalone mode:** no bottom tab bar (this is a first-run/onboarding task, not a standing destination) — once approved, the shop owner lands in the Shop Dashboard's normal tab-bar-equipped structure for the first time.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.3, 4.2, and the master endpoint table in Section 5) and NEARBY Brand/PWA Guide v1.0 (Sections 1.4, 3.3). This is the first screen in the build series to operate under the Teal-forward accent rule rather than the customer surface's Orange-forward one — every color decision above follows that rule deliberately, and the map-confirmation component is a direct, recolored reuse of the customer-side Location-Confirm screen rather than a new build, since the underlying task (confirm a point on a map) is identical on both sides of the product.*
