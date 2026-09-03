# NEARBY — AI Assistant Chat Screen Build Specification
### Everything needed to build the customer-facing `/api/v1/ai/ask` chat interface — copy, layout, cards, color, motion, assets, states
### Derived from: NEARBY Master Build Doc v2.6.2 (§4.8–4.9, 5.3, Section 7), NEARBY Brand/PWA Guide v1.0, and 2026 chatbot-UI trust/scope research (scope-setting first messages, AI-disclosure patterns, progressive-disclosure for uncertainty, feedback-control conventions)

---

## 0. Scope — what this assistant is, architecturally, before any UI decision makes sense

Section 7 of your Master Build Document draws one line that shapes this entire screen: **"The assistant answers only from NEARBY's own data — never the open web."** Every message a customer sends passes through a **Groq guardrail/router agent first**, which classifies it in-scope or out-of-scope before anything reaches the larger model that actually answers (Gemini Flash-Lite, for the Customer Assistant specifically, per the four-agent table in §7). This is a **retrieval-scoped assistant, not a general chatbot** — and the UI has to make that boundary legible to the customer from the first message, not discover it awkwardly three exchanges in when a question about, say, the weather gets deflected.

2026 chatbot-UX research is emphatic on exactly this point: **"users forgive a limited assistant that declared its limits. They don't forgive one that pretended."** Every section below is built around stating NEARBY's assistant's scope honestly, up front, and handling out-of-scope questions as a normal, expected part of the conversation rather than an error.

**What the Customer Assistant is actually for (per §7's agent table):** request progress ("where's my search at"), radius/product suggestions, and FAQ — not general chat, not open-web search, not anything about a different shop's internal operations or another customer's data.

---

## 1. Page goals, in priority order

1. Set scope honestly in the very first thing the customer sees — per the research, this is "the highest-traffic screen in the whole feature," and the first message should render instantly as static copy, not as an inference the customer has to wait for.
2. Make it easy to ask a good first question — suggested-prompt chips tied to what this assistant is actually good at (§7's job list), not a blank text field and a blinking cursor.
3. Handle out-of-scope questions gracefully and quickly — the Groq guardrail is specifically chosen for speed (§ "small, fast model"), so an out-of-scope deflection should feel近-instant, not like the customer waited for a slow "no."
4. Close the loop on quality — `POST /ai/feedback`'s `helpful`/`not_helpful` rating exists for a reason (it feeds the admin's weekly AI-review digest, §4.9/UC-16), so every assistant message should make giving that feedback trivially easy.

---

## 2. Layout overview

Single screen, reached via the bottom tab bar's "AI Assistant" tab (Home page spec, Section [I]) — a standing "place" in the app, not a modal.

```
┌─────────────────────────────┐
│ [A] Header (title + disclosure) │
├─────────────────────────────┤
│ [B] Message thread              │  ← scrolls
│     (or [C] empty/first-message state) │
├─────────────────────────────┤
│ [D] Typing indicator            │  ← conditional, appends to thread
├─────────────────────────────┤
│ [E] Composer (input + send)     │  ← fixed to bottom
└─────────────────────────────┘
```

---

## 3. Section-by-section spec

### [A] Header
**Layout:** 56px row, back-chevron left (returns to Home — though since this is a tab-bar destination, this may instead be omitted in favor of standard tab navigation, consistent with how the Request History screen also has no back-chevron), centered title, a small clock/history icon on the right.
**Copy:** "NEARBY Assistant"
**Disclosure line, directly below the title (small, persistent, not a one-time toast):** "Powered by AI · Answers only about NEARBY" (Caption, Text-secondary) — this is the single most research-backed line on the whole screen: a persistent, low-key AI disclosure plus an explicit scope statement, both in one line, both visible for the entire session rather than shown once and scrolled away.
**History icon (right side):** opens a simple list of past conversations (`GET /api/v1/ai/history`, paginated per §5.2) — a lightweight drill-through, not built out in full here since it's a simple reused list pattern (rows: first message preview + relative date, tap to reopen that conversation's transcript read-only or resume it).
**Background:** White, 1px Border-gray bottom border on scroll.
**Animation:** none.

### [B] Message thread
**Layout:** standard chat-thread scroll view, messages stack bottom-up, auto-scrolls to the latest message on send/receive.
**User message bubble:** right-aligned, filled Primary orange background, White text, rounded 16px with a slightly squared bottom-right corner (a common, subtle "this bubble points to the sender" convention), max-width capped at **75% of the viewport** on mobile (directly per the research's explicit width-cap guidance, which exists specifically to keep long messages readable rather than stretching edge-to-edge).
**Assistant message bubble:** left-aligned, Surface white background, 1px Border-gray outline, Text-primary text, rounded 16px with a squared bottom-left corner, same 75%-width cap. A small avatar (24×24, the NEARBY logomark, not a cartoon face — research warns avatars "can feel gimmicky," and reusing the existing brand mark instead of inventing a bot-persona character keeps this consistent with the product's existing visual identity rather than introducing a new character) sits to the left of the first bubble in a consecutive run of assistant messages only, not repeated on every bubble.
**Timestamps:** small Caption-size, Text-secondary, shown only on the last message in a consecutive run from the same sender, tucked below the bubble — avoids cluttering every single message with a repeated timestamp.
**Long content (tables, lists) inside an assistant bubble:** since this assistant answers from NEARBY's own structured data (request status, reliability-score explanations, FAQ), occasional short lists or a small data callout are plausible — render these as plain formatted text within the bubble (bullet points, bold key terms) rather than a nested card-within-a-bubble, keeping the bubble itself the atomic unit of the interface.
**Animation:** each new message fades in + translateY(6px→0) over `duration-quick` (180ms, `ease-out-soft`) as it's added to the thread — a light, quick entrance, not a bouncy or attention-grabbing one, consistent with the calm-tone rule applied throughout this product.

### [C] Empty state / first message — the highest-priority section on this screen
**Purpose:** per Section 1's Goal 1 and the research's explicit framing of this as the highest-traffic moment in the whole feature.
**Critical implementation detail:** this first message is **static copy, rendered instantly on screen mount — never a model call, never a loading state.** The research is explicit about this ("it's copy, not inference, and the trust clock starts the moment the window opens") and it also happens to align with cost/latency reality: no reason to spend a Gemini call on a fixed greeting.
**Copy (assistant bubble, styled identically to a normal assistant message):** "Hi! I can help with your current search, suggest a good radius or how to word what you're looking for, and answer questions about how NEARBY works. I can't help with anything outside NEARBY, though — no general web search here." (kept to two short sentences plus the explicit limitation, directly implementing the research's "sets scope... sets expectations... says what it can't do" structure, and deliberately under the "two sentences and a suggestion is the ceiling" guidance, with the limitation folded into the second sentence rather than a third).
**Suggested-prompt chips, directly below the first message (the "hands the user a first move" step):** 3–4 rounded-pill chips, tappable, pre-filling and auto-sending the composer:
- "What's happening with my current search?"
- "What radius should I use?"
- "How does the reliability score work?"
- "How do I become a shop on NEARBY?"
**Chip styling:** rounded-full, 36px height, Gray-100 background, Text-primary text — identical visual treatment to every other suggestion-chip pattern in this product (Home's category chips, New Broadcast's radius quick-picks), so this doesn't introduce a new chip style.
**Behavior:** if the customer has an active request (`open`/`awaiting_selection`/`selected`), the first chip's copy can dynamically reflect that ("What's happening with my search for **[productQuery]**?") rather than the generic version — a small but meaningful personalization that costs nothing extra to implement since the client already has this data from Home.
**Animation:** first message renders with no animation at all (appears instantly, per the "don't stream, don't make users wait" rule); chips fade in with a light `stagger-step` (40ms) immediately after, `duration-standard` (240ms).

### [D] Typing/thinking indicator
**Purpose:** the guardrail-then-model pipeline (Section 0) means there's a real, if usually brief, latency gap between send and response — the research is clear this needs a visible waiting state, not a frozen composer.
**Layout:** appears as a temporary assistant-side bubble, same position/styling as a real assistant message, containing three small dots animating in a simple sequential bounce (a near-universal "typing" convention, kept here rather than invented as something more branded, since this specific pattern is instantly legible without any learning curve).
**Copy:** none — just the dot animation; no "Thinking…" label needed, the dots alone are sufficient and a text label would be redundant.
**Duration expectation:** the Groq guardrail step is fast by design (§ "small, fast model"); if a message turns out to be out-of-scope, the customer should see the deflection response (Section [B]/[C] treatment, Section 4) resolve *faster* than an in-scope, Gemini-routed answer — the UI doesn't need to distinguish these two cases visually, since the underlying latency difference alone communicates it naturally.
**Animation:** three dots, sequential scale-pulse (each dot: 1.0→1.3→1.0, offset by ~150ms from the next), looping only for the duration this indicator is shown — removed and replaced by the real message the instant a response arrives, never left running alongside a rendered answer.

### [E] Out-of-scope response handling *(a message-content pattern within [B], not a separate visual component)*
**Purpose:** implements Section 0/Goal 3 directly — this is the moment the guardrail did its job, and the UI's only responsibility is to make that feel like a normal, unembarrassing part of the conversation.
**Copy pattern (rendered as a completely standard assistant bubble, no special color, icon, or "error" framing):** "That's outside what I can help with here — I only know about NEARBY itself. If you're looking for **[the closest in-scope topic, inferred where reasonably possible]**, I can help with that instead." — the bracketed redirect is a soft, best-effort attempt (e.g. a question about "which pharmacy has the best prices" might redirect toward "I can't compare prices, but I can tell you which nearby pharmacies are open right now" via the Direct Shop Search screen) but falls back to a plain "What would you like to know about your search, a shop, or how NEARBY works?" when no sensible redirect exists.
**Why no special visual treatment:** per the research's "users forgive a limited assistant... they don't forgive one that pretended" — dressing up a scope-boundary message in Warning-amber or a distinct icon would frame an entirely normal, expected outcome as if something went wrong, which contradicts the calm, non-alarmist tone this entire product maintains for every other "this is a normal outcome, not an error" case (the Expired screen, the Cancelled screen, the Feedback screen's "Not found" option — all deliberately un-alarmed, and this is the same principle applied to the AI layer).
**Feedback controls still apply** (Section [F] below) — an out-of-scope deflection is still a real response worth rating, in case the guardrail mis-classified something that was actually answerable.

### [F] Feedback controls (thumbs up/down)
**Purpose:** directly implements `POST /api/v1/ai/feedback` (`rating: helpful | not_helpful`, §4.9) — this is real product telemetry, not decoration, since it feeds the admin's `GET /admin/ai/review` weekly digest (UC-16).
**Layout:** a small row of two icon-only buttons (thumbs-up / thumbs-down, 20×20 each), appears below every assistant message, subtly (low-opacity, Text-secondary) until hovered/tapped-near on desktop, or simply always faintly present on mobile (touch has no hover state to rely on).
**Behavior:** tapping either fills that icon (Success teal for thumbs-up, Text-secondary-filled for thumbs-down — deliberately not Danger red for a negative rating, same "don't over-dramatize a normal input" principle as everywhere else) and sends `POST /ai/feedback` with `{ conversationId, rating }`; a rating can be changed by tapping the other icon (switches the fill), but once submitted at least once, the pair stays visibly rated rather than resetting to neutral — no confirmation dialog, no toast, this should be as frictionless as the research's "thumbs up or 'was this helpful?'" convention implies.
**No comment field:** kept deliberately binary per the documented schema (`helpful`/`not_helpful` only, no free-text note field on this endpoint) — a comment box here would imply a capability the backend doesn't capture.
**Animation:** icon fill transitions on tap, `duration-instant` (100ms).

### [G] Composer (input + send)
**Layout:** fixed to the bottom of the screen, full-width text input (multi-line, expands up to ~4 lines before scrolling internally) + a circular send button (40×40) to its right, sits above the safe-area inset.
**Placeholder:** "Ask about your search, a shop, or how NEARBY works" — a compact restatement of scope, doing double duty as both a text-entry cue and a lightweight scope reminder for a customer who's scrolled past the header's disclosure line.
**Send button:** disabled (Gray 300) when the input is empty; filled Primary orange, White paper-plane icon, once text is present; tap-scale-to-0.97 on send.
**Behavior:** Enter/Return sends on desktop (Shift+Enter for a newline); on mobile, the send button is the primary submission path. Sending clears the input immediately and appends the user's bubble to the thread before the response arrives (optimistic UI — the customer's own message never needs to wait on a network round-trip to appear).
**Rate-limit wall (20 requests/user/hour, §5.3):** if hit, the composer disables and a plain inline note replaces the placeholder: "You've reached the hourly limit for assistant questions — try again in **[time until reset]**." (Text-secondary, no Danger red — a rate limit is a normal operating boundary, not a fault of the customer's) — per the research's explicit call-out that "rate-limit walls with reset times" belong in a well-designed failure-state set, not a bare, unexplained block.
**Animation:** border-color transition on focus (`duration-instant`, Primary orange 2px, same as every other text input in this product).

---

## 4. Color usage summary (Brand Guide §3.1/3.2 — no new hex introduced)

| Token | Hex | Used where |
|---|---|---|
| Primary | `#FF5A36` | User message bubbles, send button, focus border, suggested-prompt chip taps |
| Success | `#00B8A9` | Thumbs-up filled state |
| Text primary | `#14213D` | Assistant bubble text, header title |
| Text secondary | `#5B6472` | Disclosure line, timestamps, placeholder text, thumbs-down filled state, rate-limit note |
| Border | `#D8DCE3` | Assistant bubble outline, composer input outline |
| Surface | `#F4F5F7` / `#FFFFFF` | Suggestion-chip background, page background (Gray 100) / assistant bubbles, header, composer background (White) |

**No Danger red anywhere on this screen** — out-of-scope deflections, negative feedback, and rate-limit walls are all treated as normal operating states, never errors, consistent with the pattern established across every other screen in this series.

---

## 5. Typography summary (Brand Guide §4.2)

| Element | Role/size | Weight |
|---|---|---|
| Header title | H1 / 24px | 600 SemiBold |
| Disclosure line | Caption / 13px | 400 Regular |
| Message bubble text (both sender types) | Body / 15px | 400 Regular |
| Message timestamps | Caption / 13px | 400 Regular |
| Suggested-prompt chip labels | Caption / 13px | 400 Regular |
| Composer placeholder/input text | Body / 15px | 400 Regular |
| Rate-limit note | Caption / 13px | 400 Regular |

---

## 6. Animation & motion summary (Brand Guide §9.1–9.5 — reused tokens only)

| Motion token | Value | Applied to |
|---|---|---|
| `duration-instant` | 100ms | Composer focus border, feedback-icon fill, send-button tap-scale |
| `duration-quick` | 180ms | New message fade+slide-in |
| `duration-standard` | 240ms | Suggested-prompt chip entrance stagger |
| `stagger-step` | 40ms | Delay between suggestion chips on first render |
| `ease-out-soft` | `cubic-bezier(0,0,0.2,1)` | Message and chip entrances |
| Typing-indicator dot pulse | ~150ms offset per dot, looping only while active | [D], removed the instant a real response arrives |

**No broadcast-pulse motif anywhere on this screen** — the typing indicator uses the universal three-dot convention instead, deliberately not repurposing the brand's own "live broadcast" animation for a different kind of waiting state; conflating the two would blur a visual signal this product otherwise keeps very consistent (Section 6 of the Live-Count spec).
**Reduced motion:** message entrance and chip stagger degrade to instant appearance; the typing-indicator dots switch to a static "···" if `prefers-reduced-motion` is set, still communicating "waiting" without motion.

---

## 7. Images, icons & video — asset list

**No video, no photography.**

| Asset | Format/size | Source |
|---|---|---|
| Assistant avatar | NEARBY logomark, 24×24 | Reused from `nearby-mark.svg` — no new asset |
| Thumbs-up / thumbs-down icons | Line icons, 20×20, fillable | Icon library |
| Send (paper-plane) icon | Line icon, 20×20 | Icon library |
| History/clock icon (header) | Line icon, 20×20 | Icon library |
| Back-chevron (if included per Section 3[A]'s note) | Line icon, 24×24 | Icon library |

---

## 8. Data & state requirements

| Section | Data / endpoint |
|---|---|
| [B]/[D] | `POST /api/v1/ai/ask` — sends the customer's message (and implicitly, `agent: 'customer'` context server-side); response includes the assistant's reply text and a `conversationId` for feedback association |
| [A] History icon | `GET /api/v1/ai/history`, paginated (§5.2) — this customer's own past conversations only |
| [F] Feedback | `POST /api/v1/ai/feedback` with `{ conversationId, rating }` |
| [G] Rate limit | Enforced server-side (20/user/hour, §5.3) — the 429 response should carry a reset-time value the client can render directly in the composer's disabled-state copy, rather than the client guessing or hard-coding "an hour" |

**Loading state:** the typing indicator (Section [D]) is this screen's only loading state — no skeleton needed, since a chat thread's own message-by-message loading is inherently sequential rather than a batch of data to placeholder.
**Error state:** a genuine network/server failure (distinct from a rate-limit wall or an out-of-scope deflection, both of which are normal successful responses with specific content) uses the shared toast/error-boundary pattern (§6.1), with the failed user message remaining visible in the thread and a small inline "Retry" affordance directly beneath it, rather than silently vanishing.

---

## 9. Responsive & platform notes

- **Viewport:** message bubbles cap at 75% width on mobile per Section 3[B]'s explicit research-backed rule; on wider viewports (tablet/desktop, within this product's existing 480px-max-width centered layout convention), the same proportional cap still applies rather than switching to a fixed pixel width, keeping the thread's proportions consistent across breakpoints.
- **Keyboard handling:** the composer should behave like every other text-input screen in this product — scrolls the thread to keep the latest message visible above the on-screen keyboard, standard behavior.
- **90-day data retention (§4.8/4.9, §11):** conversations and their feedback both auto-expire after 90 days — the History list (Section 3[A]) should simply reflect whatever's still within that window without the UI needing any special "this conversation expired" messaging; older conversations just won't appear, the same way any TTL-governed data quietly ages out elsewhere in this system.
- **PWA standalone mode:** carries the bottom tab bar (this is a permanent destination, per Section 2), consistent with Home, History, and Profile.

---

*This document is internally consistent with NEARBY Master Build Document v2.6.2 (Sections 4.8, 4.9, 5.3, and Section 7 in full) and NEARBY Brand/PWA Guide v1.0 (Sections 3–5, 9). Every design decision here follows from one architectural fact stated plainly in your own Section 7: this is a retrieval-scoped assistant answering only from NEARBY's own data, guarded by a fast classification step before any larger model runs — the UI's entire job is to make that boundary honest and unembarrassing rather than something the customer discovers by hitting it unexpectedly.*
