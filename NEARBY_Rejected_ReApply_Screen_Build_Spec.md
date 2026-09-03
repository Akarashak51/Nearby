# NEARBY — Rejected / Re-apply Screen Build Specification
### Everything needed to build the `status: 'rejected'` shop-dashboard screen and its resubmission form — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§3.3 v2.6.1 fix, 4.2, endpoint table §5), NEARBY Brand/PWA Guide v1.0 (§1.4, 3.3), and re-application/form-recovery UX pattern research (the same "never a dead end, explain why" empty-state principles applied throughout this series)

---

## 0. Scope — this screen exists because of one specific v2.6.1 fix, and everything here follows from it

Before v2.6.1, a rejected shop had no documented path back in — this exact gap is what that fix closed. The mechanism it introduced is precise and worth restating exactly, since it shapes this whole screen: **`PATCH /shops/me` now also accepts `{ shopName?, category?, address?, phone?, photo | document? }` whenever the caller's shop is `status: 'rejected'`** — the same field set as the original `POST /shops`. A successful resubmission clears `rejectionReason` and resets `status` back to `'pending'` for another admin review.

**What this means concretely for the screen's design:** this is not a new form — it's **the Onboarding screen's own form, reused, pre-filled with the shopkeeper's previous submission, with one added element (the rejection reason) at the top.** Building a separate, differently-structured "re-apply" form would be both wasted effort and a worse experience — a shopkeeper who already filled this out once shouldn't have to learn a new layout to fix one thing and resubmit.

**Tone note, carried over from Section 0 of the Pending-approval spec:** a rejection in this pilot's process (§3.3's criteria: reachable phone, plausible/non-duplicate address, non-duplicate photo/document, category-context consistency) is much more often an honest mismatch or a fixable mistake than a bad-faith submission — a landmark-only address that didn't geocode cleanly, a blurry photo, a category that didn't match the storefront photo. The screen's whole posture should reflect that: **this is a fixable checkpoint, not a punishment.**

---

## 1. Page goals, in priority order

1. Tell the shopkeeper exactly why their submission wasn't approved, in plain language — `rejectionReason` is a real field your admin side populates specifically so this moment doesn't leave anyone guessing (§3.3: "A rejected shop stores a rejectionReason so the applicant can be told why and reapply").
2. Make fixing the problem as low-effort as possible — pre-fill everything that was already correct, and make the field that actually needs attention easy to find without redoing the parts that were fine.
3. Never let this screen feel like starting over — same familiar layout as the original Onboarding form, not a new, unfamiliar one.
4. Keep the tone respectful and matter-of-fact throughout, per Brand Guide §1.4's "respectful of shopkeepers' time" principle — this is doubly true here, where the shopkeeper has already invested effort once and hit a setback.

---

## 2. Layout overview

Single scrollable screen — structurally identical to the Onboarding screen's layout (Section 2 of that spec), with one new section inserted at the top.

```
┌─────────────────────────────┐
│ [A] Header (title)             │
├─────────────────────────────┤
│ [B] Rejection reason card       │  ← new, only on this screen
├─────────────────────────────┤
│ [C] Shop name field (pre-filled) │
├─────────────────────────────┤
│ [D] Category selector (pre-filled) │
├─────────────────────────────┤
│ [E] Address field + map confirm (pre-filled) │
├─────────────────────────────┤
│ [F] Phone field (pre-filled)     │
├─────────────────────────────┤
│ [G] Photo / document (pre-attached) │
├─────────────────────────────┤
│ [H] Resubmit button                │  ← fixed to bottom
├─────────────────────────────┤
│ [I] Sign out                        │
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, no back-chevron (same reasoning as the Pending-approval screen — per that spec's Section 0, a `role: 'shop'` account with no approved shop has nowhere else in the product to go).
**Copy:** "Update your application"
**Background:** White, no border/shadow.
**Animation:** none.

### [B] Rejection reason card — the one genuinely new section on this screen
**Purpose:** Goal 1, directly.
**Layout:** card, Surface white background, a **Warning amber (`#FFB627`) left accent bar** (4px, same treatment pattern already used for the customer-side Live-Count screen's `awaiting_selection` banner state) rather than a full amber fill — a fixable-checkpoint signal, not an alarm. **Not Danger red** — the same reasoning already established across this series (the Cancelled screen's suspension variant, the Push-Settings blocked state): Danger is reserved for destructive actions and hard validation failures, and a rejection with a documented, actionable reason and a clear path back is closer in spirit to those Warning-level "pay attention, this is fixable" states than to a true error.
**Copy:** "Your application needs a small fix" (H2, Text-primary, 600 SemiBold) directly above the actual stored `rejectionReason` text (Body, Text-primary — rendered verbatim from the admin's note, since that's the one piece of genuinely specific, actionable information on this whole screen).
**Animation:** fades in (`duration-standard`, 240ms, `ease-out-soft`) on mount — same restrained, one-time entrance as every outcome-style element in this series.

### [C]–[G] The reused Onboarding form, pre-filled
**Every field, layout choice, color, and behavior in these five sections is identical to the Onboarding screen's own Sections [C]–[G]** (shop name, category selector, address + map-confirm, phone, photo/document upload) — down to the Signal Teal focus borders, the same category option set, the same reused Leaflet/OSM map-confirm component, and the same 5MB client-side file-size validation. **Only the initial state differs:**
- **[C] Shop name, [D] Category, [F] Phone:** pre-filled with the values from the shop's existing record (`GET /shops/me`) — editable, not read-only, since any of these might be exactly what needs correcting.
- **[E] Address:** pre-filled with the previously-submitted `formattedAddress`, with the map-confirm component already centered on the previously-confirmed point — a shopkeeper whose address was the actual problem can immediately see where the pin landed last time and adjust from there, rather than starting from a blank map.
- **[G] Photo/document:** the previously-uploaded file is shown **already attached** (thumbnail or filename, per Onboarding spec Section 3[G]'s attached-state treatment) — since `photo | document` is optional on this resubmission (§3.3's v2.6.1 fix: `{ ..., photo | document? }`), a shopkeeper whose photo *wasn't* the issue shouldn't be forced to re-upload it just to fix their phone number. The "×" remove control (already part of the Onboarding component) lets them replace it only if that's what actually needs fixing.
**Why no field is highlighted as "the problem" automatically:** `rejectionReason` is stored as free text (§4.2), not a structured field-level flag — the system doesn't know programmatically which specific input caused the rejection, only what the admin wrote in prose. Auto-highlighting a guessed field based on keyword-matching the reason text would be fragile and potentially wrong; instead, [B]'s reason card does the job of directing attention, in the shopkeeper's own reading of it, to whichever field actually needs the fix.

### [H] Resubmit button
**Layout:** identical treatment to the Onboarding screen's Submit button (fixed bottom, full-width minus margins, 48px height, filled **Signal Teal**).
**Copy:** "Resubmit for review" — distinct from the original "Submit for review," a small but meaningful copy difference that acknowledges this is a second attempt, not treating it as if nothing happened before.
**States:** identical logic to the Onboarding screen's Submit button (disabled until required fields are present — noting that since photo/document is optional here rather than strictly required the way it was on first submission, the enablement check should treat "an existing attached file, whether newly replaced or carried over" as satisfying that requirement, not require a fresh upload every time).
**On success:** `PATCH /shops/me` with the full field set, `status` resets to `pending`, `rejectionReason` clears server-side — the screen transitions back to the **Pending-approval screen** (previous spec), which will render its normal "under review" state with no trace of the prior rejection, exactly as if this were a first-time submission now going through review again.
**Animation:** identical to the Onboarding screen's Submit button (tap-scale, progress-bar during the multipart upload if a new file was attached, standard cross-fade on success).

### [I] Sign out
**Layout, copy, and behavior:** identical to the Pending-approval screen's own Sign Out row (Danger red, centered, bottom of screen) — included for the same reason: this account still has nowhere else to go while its shop status remains unresolved.
**Animation:** identical to the Pending-approval/Profile screens' treatment.

---

## 4. Color usage summary (Brand Guide §3.1/3.3 — no new hex introduced)

| Token | Hex | Used where |
|---|---|---|
| **Signal Teal** | `#00B8A9` | Every form-field focus border, selected category radio, map-confirm accents, attached-upload-card border, Resubmit button — identical usage to the Onboarding screen |
| Warning | `#FFB627` | Rejection-reason card's left accent bar (the one screen-specific color choice) |
| Danger | `#E5484D` | Sign-out row (same singular exception as the Pending-approval/Profile screens), field-level validation errors if any required field is left empty on resubmission |
| Text primary | `#14213D` | Headline, rejection-reason text, field labels/values |
| Text secondary | `#5B6472` | Helper text, placeholder text |
| Border | `#D8DCE3` | Default input/card outlines |
| Surface | `#F4F5F7` / `#FFFFFF` | Page background (Gray 100) / cards, inputs, header (White) |

**Still no Primary orange anywhere on this screen** — same Teal-forward rule carried over from the Onboarding and Pending-approval screens; this remains a form/status screen with no Yes/No response action.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Rejection-card heading | H2 / 18px | 600 SemiBold |
| Rejection-reason text | Body / 15px | 400 Regular |
| All form field labels/inputs | Body / 15px | 400 Regular |
| Resubmit button label | Button / 15px | 600 SemiBold |
| Sign-out label | Button / 15px | 600 SemiBold |

*(Every form-field typographic treatment is otherwise identical to the Onboarding screen's own Section 5 table — not repeated in full here to avoid duplicating a spec that hasn't changed.)*

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-standard` | 240ms | Rejection-card fade-in on mount, screen-exit to Pending-approval on resubmit |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Card entrance, screen-exit |

*(All form-interaction animations — focus transitions, category expand, upload cross-fade — are identical to the Onboarding screen's Section 6 table, reused without change.)*

**No looping animation anywhere on this screen** — same reasoning as every calm, procedural screen in this series.
**Reduced motion:** covered by the app-wide `prefers-reduced-motion` query.

---

## 7. Images, icons & video — asset list

**No video, no stock photography.**

| Asset | Format/size | Source |
|---|---|---|
| Rejection-card accent bar | CSS-drawn 4px solid bar, Warning amber | — |
| All form icons (camera, document, remove, map-confirm assets) | Identical to the Onboarding screen's asset list | Reused, no new assets |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B] Rejection reason | `GET /api/v1/shops/me` (§6.1) — `rejectionReason`, only populated/relevant when `status === 'rejected'` |
| [C]–[G] pre-fill | Same `GET /shops/me` response — `shopName`, `category`, `address`/`formattedAddress`, `phone`, `photoUrl`/`documentUrl` |
| [H] Resubmit | `PATCH /api/v1/shops/me` (Bearer, shop role) with `{ shopName?, category?, address?, phone?, photo | document? }` (§3.3's v2.6.1 fix) — server clears `rejectionReason` and resets `status` to `pending` on success |

**Loading state:** brief skeleton (matching the Pending-approval screen's own treatment) while the initial `GET /shops/me` resolves and pre-fills the form.
**Error state:** a failed resubmission uses the shared toast/error-boundary pattern (§6.1) with all form data — including any newly-attached file — preserved for retry, identical reasoning to the Onboarding screen's own error-state note.

---

## 9. Responsive & platform notes

- **Viewport, upload handling, camera capture, and progress-bar behavior:** all identical to the Onboarding screen's Section 9 notes — this screen inherits that spec's platform considerations wholesale, since the form itself is the same component, just pre-filled.
- **No bottom tab bar** — same reasoning as the Pending-approval screen (Section 0 of that spec): nothing else exists for this account yet.
- **Screen-exit destination:** a successful resubmission always leads to the Pending-approval screen, never directly to the Live Dashboard — even a corrected, obviously-fine resubmission still requires a fresh admin review pass (§3.3's re-apply mechanism explicitly resets `status` to `pending`, not `approved`), and this screen shouldn't imply otherwise.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 3.3's v2.6.1 fix, 4.2, and the master endpoint table in Section 5) and NEARBY Brand/PWA Guide v1.0 (Sections 1.4, 3.3). This screen is deliberately specified as a pre-filled reuse of the Onboarding form rather than a new, separately-designed screen — the only genuinely new content here is the rejection-reason card, and every other section inherits its layout, color, and behavior directly from the Onboarding spec, consistent with how this series has handled every other component-reuse opportunity (the Shortlist card, the map-confirm component, the recap cards) throughout.*
