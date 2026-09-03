# NEARBY — Report a Shop Screen Build Specification
### Everything needed to build the `POST /reports` (reportedType: shop) screen — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§4.7, 5.3, FR-3.3/3.4), NEARBY Brand/PWA Guide v1.0, and flag/report-content UX pattern research (the standard "select reason → optional detail → submit" moderation-report pattern, Google Maps' own report-a-problem flow structure)

---

## 0. Scope — what the schema fixes, and the one thing it leaves to sensible defaults

Your `reports` collection (§4.7) fixes most of this screen's shape precisely: `reportedType: 'shop'`, `reportedId` (the shop being reported), `reason` (required, "short category"), `details` (optional free text), and a 5-per-user-per-day rate limit (§5.3) specifically sized to prevent report-spam harassment of a shop. What it does **not** fix is the literal list of reason categories a customer picks from — `reason` is documented as "a short category," not an enumerated set of exact strings. This document proposes a concrete, sensible list (Section 3[C]) built around what a customer filing this report from NEARBY's specific broadcast-and-visit model would actually need to say — flagged clearly as a designed default, not something copied verbatim from the source document, since none exists there to copy.

**Why this screen matters more than its small size suggests:** every report filed here lands directly in the admin's moderation queue (`report:new` → admin room, §5, "Real-time events") and feeds `GET /admin/reports` — this is the one direct channel a customer has to flag something the automated system (reliability scores, ratings) doesn't otherwise catch, like a safety concern or a fraudulent listing.

---

## 1. Page goals, in priority order

1. Make filing a legitimate concern fast and low-friction — per the flag/report design pattern research, a reason-first, detail-optional structure gets a usable report with the least typing required.
2. Discourage frivolous or retaliatory use without being adversarial about it — the 5/day rate limit already does the technical work; the UI's job is to communicate that this is a real moderation channel, not a casual dislike button.
3. Never make the customer feel like *they're* the one being scrutinized for reporting — this is a calm, respectful form, not an interrogation.
4. Close the loop honestly about what happens next, without over-promising a specific outcome or timeline the admin side can't guarantee.

---

## 2. Layout overview

Single screen, reached from the Shop Profile screen's contact/address area (a small "Report this shop" text link, quietly placed rather than prominent — see Section 3 for exact placement) or from a shop's entry in Request History.

```
┌─────────────────────────────┐
│ [A] Header (close + title)    │
├─────────────────────────────┤
│ [B] Shop reminder row          │
├─────────────────────────────┤
│ [C] Reason selection            │
├─────────────────────────────┤
│ [D] Details field                │
├─────────────────────────────┤
│ [E] What happens next strip      │
├─────────────────────────────┤
│ [F] Submit button                 │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, an **X (close) icon** on the left (same reasoning as the Post-Visit Feedback screen — this is an opt-in action the customer can back out of at any point, not a forced step), centered title.
**Copy:** "Report a shop"
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Shop reminder row
**Purpose:** unambiguous confirmation of which shop this report is about — critical given the stakes of a moderation action landing on the wrong listing.
**Layout:** single row, shop photo thumbnail (40×40, rounded 8px, or initials-avatar fallback) + shop name (Body, 600 SemiBold, Text-primary), sits directly below the header — identical component treatment to the Feedback screen's own shop reminder row.
**Animation:** none.

### [C] Reason selection — the primary required field
**Purpose:** the `reason` field (§4.7, required).
**Copy (section label):** "What's the issue?"
**Layout:** vertical list of single-select radio-style rows, each a full-width tappable row (48px height minimum, generous tap target), radio indicator on the left, label text, 1px Border-gray divider between rows — a plain list rather than chips, since these options are mutually exclusive and benefit from being read top-to-bottom rather than scanned as a scattered chip cloud (the flag/report pattern research consistently uses a vertical reason-list structure for exactly this kind of mutually-exclusive categorization).
**Proposed reason set (Section 0's documented default), tailored to NEARBY's specific model rather than a generic "report a business" list:**
1. "Shop was closed when they said they'd be open"
2. "Shop responded Yes but didn't have what I needed" (the broadcast-model-specific equivalent of a "false Yes" complaint — directly relevant to reliability, and something no generic reporting flow like Google's would ever need)
3. "Rude or inappropriate behavior"
4. "Listing looks fake or misleading" (duplicate storefront photo, implausible address, etc.)
5. "Safety concern"
6. "Something else"
**Behavior:** single-select; selecting **"Something else"** reveals a small inline note beneath it: "Please describe the issue below" and makes the [D] details field required for that one reason only (every other reason leaves [D] genuinely optional, since a specific enough category may need no further explanation).
**Color:** selected row's radio indicator filled Primary orange; unselected rows Border-gray outline only; row background stays White/Surface regardless of selection (no full-row color fill, to keep this list visually calm and legible rather than looking like a set of competing buttons).
**Animation:** radio-fill transition on selection, `duration-instant` (100ms); the conditional note/requirement beneath "Something else" expands with a brief height-transition (`duration-quick`, 180ms, `ease-out-soft`) rather than snapping in.

### [D] Details field
**Purpose:** the optional (or conditionally required, per Section [C]) `details` field.
**Copy (placeholder):** "Add any details that might help (optional)" — placeholder text updates to "Please describe what happened" (no "(optional)" suffix) the moment "Something else" is selected in [C], so the field's own copy communicates its changed requirement status without needing a separate error state to explain it.
**Layout:** multi-line text area, 4 visible rows, 12px radius, Surface white background, 1px Border-gray outline, sits below the reason list.
**Character limit:** soft cap ~500 characters (larger than the Feedback screen's optional note, since a report may genuinely need more context than a visit rating does) — counter appears once typing begins, same behavior as the Feedback screen's note field.
**Color:** default border Border-gray; focus border Primary orange 2px, same as every text input in this product.
**Animation:** border-color transition on focus, `duration-instant`, `ease-standard`.

### [E] What happens next strip
**Purpose:** manages expectations honestly, without over-promising — a customer filing a report deserves to know their report goes somewhere real, but the product shouldn't guarantee a specific resolution it can't control.
**Copy:** "Our team reviews every report. We won't share your identity with the shop." — two short, factual sentences: the first confirms this isn't a black hole (directly addresses the single most common anxiety about filing any report — does anyone actually read this), the second states a real privacy guarantee worth stating plainly, since nothing in the reporting flow otherwise tells the customer their identity is protected from the party they're reporting.
**Layout:** thin full-width bar, Gray 100 background, small shield-outline icon (16×16, Text-secondary), Caption text, sits between the details field and the submit button.
**Animation:** none — static, informational, same treatment as every other "what happens next" strip in this series.

### [F] Submit button
**Layout:** fixed to the bottom of the screen, full-width minus 16px margins, 48px height, 12px radius, sits above the safe-area inset.
**Copy:** "Submit report"
**States:** disabled (Gray 300 background) until a reason is selected in [C] (and, if "Something else" was chosen, until [D] has content); enabled (filled Primary orange, White Button-weight text) otherwise; loading (in-flight `POST /reports` call) shows a centered spinner.
**Confirmation before send:** unlike most primary actions in this product, submitting a report against another party benefits from one lightweight confirmation step — a brief inline modal: "Submit this report? Our team will review it." with **Cancel** / **Submit** actions — this is a smaller, calmer version of the same "confirm before an action that affects someone else" pattern already used for the Shortlist screen's shop-selection confirmation, appropriately scaled down since a report is reversible in spirit (the admin can dismiss it) but still deserves one deliberate step before sending.
**On success:** the screen transitions to a brief, quiet confirmation state — "Thanks — we've received your report." (same restrained, non-celebratory treatment as the Feedback screen's own post-submit acknowledgment, per the "tune the celebration to the size of the action" principle already established across this series; filing a report is not something to visually celebrate) — then returns to wherever the customer came from (Shop Profile or Request History) after ~1.5 seconds.
**Rate-limit state (5/user/day, §5.3):** if the limit is already exhausted when the customer opens this screen (rather than discovered only on submit), the entire form is replaced with a plain notice: "You've submitted the maximum number of reports for today. Please try again tomorrow, or contact support directly if this is urgent." — checked proactively on screen load where feasible (a lightweight pre-check, or simply surfaced gracefully if the actual submit attempt returns the rate-limit response) so the customer isn't left filling out a whole form only to hit a wall at the very end.
**Animation:** tap-scale-to-0.97; confirmation modal fades+scales in (`duration-quick`, `ease-standard`); on success, the confirmation-text swap and subsequent screen-exit both use the standard cross-fade pattern (`duration-quick` for the inline swap, `duration-standard` for the exit, `ease-out-soft`).

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where |
|---|---|---|
| Primary | `#FF5A36` | Selected reason radio fill, submit button, "Submit" confirmation-modal action |
| Text primary | `#14213D` | Header title, shop name, reason labels |
| Text secondary | `#5B6472` | What-happens-next strip text, rate-limit notice, field placeholders |
| Border | `#D8DCE3` | Reason-row dividers, unselected radio outline, text-area outline |
| Surface | `#F4F5F7` / `#FFFFFF` | What-happens-next strip background (Gray 100) / header, reason list, details field, page background (White) |

**No Danger red anywhere on this screen** — even though this is a moderation/complaint flow, the form itself should feel calm and procedural, not alarming; Danger stays reserved for validation errors and destructive-action confirmations elsewhere in the product, neither of which applies to a customer calmly describing a concern.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Shop name (reminder row) | Body / 15px | 600 SemiBold |
| Reason-section label | H2 / 18px | 600 SemiBold |
| Reason row labels | Body / 15px | 400 Regular |
| Details field placeholder/text | Body / 15px | 400 Regular |
| Character counter | Caption / 13px | 400 Regular |
| What-happens-next strip | Caption / 13px | 400 Regular |
| Submit button label | Button / 15px | 600 SemiBold |
| Success confirmation text | Button / 15px | 600 SemiBold |
| Rate-limit notice | Body / 15px | 400 Regular |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Radio-fill selection, tap-scale feedback, text-area focus border |
| `duration-quick` | 180ms | Conditional "Something else" note expand, confirmation-modal fade-in, success-text inline swap |
| `duration-standard` | 240ms | Screen-exit cross-fade after successful submission |
| `ease-standard` | `cubic-bezier(0.4,0,0.2,1)` | Selection/focus transitions |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Expand transitions, screen-exit |

**No looping animation anywhere on this screen** — consistent with every form/decision screen in this series; this is a calm, procedural task, not a live state.
**Reduced motion:** covered by the app-wide `prefers-reduced-motion` query, same degradation pattern as every other screen.

---

## 7. Images, icons & video — asset list

**No video, no photography beyond the small shop-reminder thumbnail** (an existing asset, not new).

| Asset | Format/size | Source |
|---|---|---|
| Shop photo thumbnail / initials-avatar fallback | 40×40, rounded 8px | Reused from the shop's existing onboarding photo |
| Close (X) icon (header) | Line icon, 24×24 | Icon library |
| Radio-select indicator | CSS-drawn circle (outline/filled states) | — |
| Shield-outline icon (what-happens-next strip) | Line icon, 16×16, Text-secondary | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B] Shop reminder | Shop name/photo already available from wherever this screen was launched (Shop Profile or Request History) — no new fetch needed |
| [C]/[D] | Client-side form state until submit |
| [F] Submit | `POST /api/v1/reports` with `{ reportedType: 'shop', reportedId, reason, details? }` (§4.7) — rate-limited 5/user/day (§5.3); server fires `report:new` to the admin room on success |
| [F] Rate-limit pre-check | If the app tracks the customer's own daily report count client-side (from prior submissions this session/day), it can show the exhausted-state notice proactively; otherwise, gracefully handle the 429 response from an actual submit attempt with the same notice copy |

**Loading state:** none needed beyond the submit button's own in-flight spinner.
**Error state:** a failed submission (network/500, not rate-limit) uses the shared toast/error-boundary pattern (§6.1) with the form state preserved and the submit button re-enabled for retry.

---

## 9. Responsive & platform notes

- **Viewport:** centered single-column composition, max-width 480px, consistent with every screen in this series; the reason list scrolls normally within the screen if needed, no special treatment required at this length (6 options).
- **Discoverability vs. prominence balance:** the entry link on the Shop Profile screen ("Report this shop") should be genuinely findable — not hidden in a sub-menu — but visually quiet (Caption size, Text-secondary, placed near the Contact row rather than competing with the primary Select/Call/Directions actions) — present, not promoted, matching the tone of a feature that exists for real concerns rather than casual use.
- **PWA standalone mode:** no bottom tab bar (consistent with every opt-in task screen in this series — New Broadcast, Feedback) — the header's close (X) icon returns to wherever the customer launched this from.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 4.7, 5.3, FR-3.3/3.4) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). The specific reason-category list in Section 3[C] is a documented design default, not a literal requirement lifted from the source document — `reason` is specified only as "a short category, required," and this list is built to reflect concerns specific to NEARBY's broadcast-and-visit model (particularly the "said yes but didn't have it" option, which has no equivalent in a generic business-listing report flow) rather than a generic complaint taxonomy.*
