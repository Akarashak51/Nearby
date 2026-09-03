# NEARBY — Admin Bootstrap / First-Admin Setup
### Full Specification — Content, Format, Design, and Data (Not a Web Page — See §0)

**Version 1.0**
**Companion to:** NEARBY Master Build Document v2.6.2 (§3.5A, §9, Deliverables) · NEARBY Brand & PWA Guide v1.0 · NEARBY Page Architecture & Competitor Analysis (§3.9)
**Surface:** None — CLI/terminal only, never `/admin/*` or any HTTP route · **Role:** N/A (pre-authentication, operator-run)

---

## 0. Critical Framing — Please Read Before Anything Else

The Page Architecture & Competitor Analysis document lists this as page 3.9, "one-time flow to create the first admin account." **This is not a web page, and it must never become one.** The Master Build Document is explicit and unambiguous on this point (§3.5A):

> *"The first and only admin account is created by running `node scripts/createAdmin.js` locally against the production database connection string, or as a one-off Render Shell command — **never over HTTP**. The script reads `ADMIN_EMAIL` / `ADMIN_PASSWORD` from environment variables (not committed), hashes the password with bcrypt, and inserts a single `role: 'admin'` user."*

This constraint exists because of a **security fix already made once in this product's history**. The Master Build Document's own changelog notes that an earlier version of `POST /auth/signup` accepted a client-supplied `role` field, meaning anyone could self-register as an admin over the public API — this was closed by making `role` server-assigned only on public signup (never trusting client input, §3.5A) and by moving admin creation entirely out of the HTTP surface into a manually-run, environment-gated script.

**Building a browser-based "Create first admin" wizard — even one gated behind a secret URL, an invite token, or a "only works if zero admins exist yet" check — reopens exactly the class of vulnerability this design already closed once.** Any HTTP-reachable endpoint that can mint a `role: 'admin'` user is an attack surface regardless of how cleverly it's gated, because gates can be bypassed, tokens can leak, and "only works once" checks can race. The single existing admin account already fully governs every other admin-creation-adjacent action in this product (Shop Approval, Block/Suspend, Reports, Customer moderation) — there is no product requirement anywhere in the Master Build Document for a second, self-service admin-creation path, and this document does not invent one.

**This document therefore specifies the actual artifact: the terminal/CLI experience of running the script**, not a web UI mockup. Every section below (§4 onward) is adapted to that reality — "screen," "card," and "button" are replaced by "terminal prompt," "output block," and "keypress," because those are the real interface elements here.

---

## 1. Overview & Purpose

### 1.1 — What it is

A single Node.js script (`scripts/createAdmin.js`, NEW v2.6, Master Build Document Deliverables list) that an operator — not a customer, not a shop owner, not even a logged-in admin (since none exists yet) — runs directly against the production database, either from their own machine or via a one-off Render Shell command, to create the platform's first and only admin account.

### 1.2 — Why it's a script and not a page (restated for emphasis)

Every other account-creation path in NEARBY (customer signup, shop signup) runs over HTTP because the person creating the account and the account being created are the same low-privilege actor — there's no escalation risk in letting a customer self-register as a customer. An admin account is categorically different: it can approve or reject any shop, block any shop, suspend or ban any customer, and view aggregate platform data. No self-service HTTP path should ever be able to mint that level of access, which is precisely the lesson the product's own earlier vulnerability (§0) already taught.

### 1.3 — What this document does not do

It does not propose an in-app "Setup Wizard" screen, does not propose a secret admin-invite URL, and does not propose a "first user becomes admin automatically" convenience pattern — all three are common developer shortcuts, and all three are exactly the shape of thing §0 rules out. If the team later needs a second or third admin account, the correct pattern (consistent with §3.5A) is either running the same script again with different environment values, or building a proper **in-app "invite another admin" feature that only an already-authenticated admin can trigger** (analogous to the Staff Management feature specified elsewhere in this series, which similarly requires an already-privileged actor to extend access) — not a pre-authentication bootstrap path.

### 1.4 — Operator story

> *"As the person deploying NEARBY for the first time, I need one command I can run against the production database, one time, that creates the admin account I'll use to log into the admin console — without exposing any way for someone else to do the same thing."*

### 1.5 — Pattern source

The Page Architecture document lists this as "[NEARBY-specific]," and correctly so — none of Blinkit, Zepto, or Zomato's public-facing apps expose their internal admin-bootstrap mechanism at all (Page Architecture §3, intro), so there is no competitor UI pattern to draw from here. The pattern this document actually follows is the standard, widely-used software-engineering convention of an **environment-gated seed script**, common to nearly every web application that needs exactly one privileged account to exist before its own admin UI can be used to create more.

---

## 2. Functional Specification

### 2.1 — Required environment variables (existing, from `.env.example`, §9)

| Variable | Purpose | Committed to source control? |
|---|---|---|
| `ADMIN_EMAIL` | The email address for the new admin account | **Never** — set only in the local/Render shell environment at run time |
| `ADMIN_PASSWORD` | The plaintext password, hashed by the script before storage | **Never** — same as above |
| `MONGODB_URI` | Already required for any script/server process to reach the database | Never (standard secret) |

### 2.2 — What the script does, step by step

1. Reads `ADMIN_EMAIL` and `ADMIN_PASSWORD` from the process environment — **not** from a command-line argument (which could leak into shell history) and **not** from an interactive prompt with no confirmation step (§4.3 adds one for safety).
2. Connects to the database using the same `MONGODB_URI` the main server uses.
3. Hashes `ADMIN_PASSWORD` with bcrypt (§3.6's existing password-hashing standard — no separate hashing scheme for admins).
4. Inserts one `users` document with `role: 'admin'`, `status: 'active'`, `authProvider: 'local'`, and the hashed password — reusing the exact same `users` schema (§4.1) every other account type uses, with no admin-specific schema fork.
5. Exits with a clear success or failure message (§4.4, §4.5) — never partial/silent failure.

### 2.3 — Idempotency and re-run safety

The script should check whether a user with `ADMIN_EMAIL` already exists before inserting, and fail loudly rather than creating a duplicate or silently overwriting an existing account's password — an accidental re-run (e.g. a deploy script mistakenly re-triggering it) must never be a silent password-reset vector for an existing admin (§9).

### 2.4 — What this script explicitly does not do

- It does not create more than one admin per invocation — one run, one account, matching "the first and only admin account" framing in §3.5A precisely.
- It does not send a welcome email, does not set up 2FA, does not do anything beyond the single account-creation write described in §2.2 — keeping the script's blast radius as small as the single job it has.
- It does not run automatically on deploy — it must always be a deliberate, manually-triggered action by whoever is standing up the environment, never wired into a CI/CD pipeline where it could fire unintentionally.

---

## 3. Where It Lives

Nowhere in the deployed application. It lives in the codebase's `scripts/` directory (alongside any other one-off operational scripts the project has), run via:

```
node scripts/createAdmin.js
```

either on a developer's own machine with a production `MONGODB_URI` temporarily exported, or as a one-off command in Render's Shell feature against the live production service — both explicitly named as the two supported invocation methods in §3.5A. It is never linked from, referenced by, or reachable through any URL the deployed frontend or backend serves.

---

## 4. The Actual Interface: Terminal Output Specification

Since there is no visual page, this section specifies the **text, structure, and tone of the CLI's own output** — the actual "content" a person building or running this script will see, treated with the same care this whole documentation series gives to on-screen copy elsewhere.

### 4.1 — On successful run

```
NEARBY — Admin Bootstrap

Checking environment variables...        ✓
Connecting to database...                ✓
Checking for existing admin account...   ✓ none found
Hashing password...                      ✓
Creating admin account...                ✓

Admin account created successfully.
  Email: admin@example.com

You can now log in at /admin/login with this email and the
password you set in ADMIN_PASSWORD.

Remember to unset ADMIN_EMAIL / ADMIN_PASSWORD from your shell
environment now that this account exists.
```

### 4.2 — Design rationale for this output's structure

- **Step-by-step checklist with checkmarks**, not a single "Done!" — because this script runs against production and touches the one account that governs the entire platform's trust and safety tooling, an operator should be able to see exactly which step succeeded, mirroring the same "make the invisible legible" principle this documentation series applies to end-user pages (e.g. the fallback-polling banner on the Live Dashboard, or the consequence summaries on the Block/Suspend flows) — just expressed here as plain terminal text instead of UI copy.
- **Explicit reminder to unset the environment variables** at the end — a small but real security habit this script's own output should actively encourage, since a leftover `ADMIN_PASSWORD` in a shell's exported environment is an easy, avoidable leak.
- **No password is ever echoed back**, not even partially — only the email, which is not a secret.

### 4.3 — Confirmation prompt before writing (safety, not a UI feature)

Before the "Creating admin account..." step, the script should print a short confirmation and require an explicit `y` keypress, unless run with a `--yes` flag for scripted/non-interactive invocation (e.g. inside a one-off Render Shell session where a human is still present and reading the output, but where piping input might be inconvenient):

```
About to create an admin account for admin@example.com
against this database:
  mongodb+srv://***@cluster0.mongodb.net/nearby-prod

Continue? (y/N)
```

The connection string is shown with credentials masked (`***`) but the host/database name visible — enough for the operator to visually confirm they're pointed at the environment they think they are (a common real-world mistake this confirmation step is designed to catch), without ever printing a database credential in full.

### 4.4 — On failure: admin already exists

```
NEARBY — Admin Bootstrap

Checking environment variables...        ✓
Connecting to database...                ✓
Checking for existing admin account...   ✗

An admin account already exists (admin@example.com).
This script only creates the first admin account.

To add another admin, use the in-app admin-invite feature
once logged in (if available), or contact the engineering team.
```

Exits with a non-zero status code, consistent with standard CLI tooling conventions — so this failure is detectable by anything scripting around it (e.g. a deploy runbook checking the exit code).

### 4.5 — On failure: missing environment variables

```
NEARBY — Admin Bootstrap

Checking environment variables...        ✗

Missing required environment variable(s): ADMIN_PASSWORD

Set ADMIN_EMAIL and ADMIN_PASSWORD before running this script.
See .env.example for the full list of required variables.
```

### 4.6 — On failure: weak password

```
NEARBY — Admin Bootstrap

Checking environment variables...        ✓
Connecting to database...                ✓
Checking for existing admin account...   ✓ none found
Hashing password...                      ✗

ADMIN_PASSWORD is too short (minimum 12 characters for an
admin account). Set a stronger password and try again.
```

A stricter minimum than whatever the customer/shop signup password rule is (§7 of the Master Build Document, not restated here) is a deliberate, explicitly-flagged addition (**NEW**, not in the current Master Build Document) — an admin account's password strength deserves a higher bar than a customer's, given the scope of what it controls.

---

## 5. "Design System" — What Applies and What Doesn't

Since this artifact has no visual surface, the Brand & PWA Guide's color, typography, and motion tokens (§3, §4, §9) **do not apply** — there is no Ink Navy, no Signal Teal, no card, no animation, because there is no rendered UI for any of those to style. This section exists specifically to state that plainly, rather than leaving the absence unexplained.

- **Text formatting:** plain monospace terminal output, using simple ASCII checkmarks/crosses (`✓`/`✗`) and indentation for hierarchy — no color-coding is assumed, since terminal color support varies by environment (local machine vs. Render Shell) and the message content must be equally clear in either case.
- **Graphics, images, video:** none, and none should ever be added — a bootstrap script has no reason to render anything beyond text.
- **Cards, animations, backgrounds:** not applicable — there is no card metaphor, no motion, no background color in a terminal script's output.

---

## 6. States & Edge Cases

| Case | Behavior |
|---|---|
| Script run with all correct env vars, no existing admin | Success path (§4.1) |
| Admin already exists | Fails loudly (§4.4), no duplicate created, non-zero exit code |
| Missing env var(s) | Fails before touching the database at all (§4.5) — no partial connection attempt |
| Database unreachable | Fails at the "Connecting to database..." step with the underlying connection error surfaced, not swallowed |
| Password too short | Fails after existence-check but before hashing/writing (§4.6) — no account is left in a partial state |
| Operator declines the confirmation prompt (§4.3) | Script exits cleanly with "Aborted — no changes made." and a zero exit code (this isn't an error, it's an intentional cancel) |
| Script run twice in a row with the same env vars | Second run correctly detects the existing admin and fails per §4.4 — this is the idempotency guarantee from §2.3, not a bug |

---

## 7. Accessibility

Terminal output should avoid relying on color alone even where a given terminal does support it (e.g. green checkmarks) — the ASCII `✓`/`✗` symbols themselves already carry the pass/fail meaning in text, so the output remains equally legible in a plain-text log file, a CI pipeline's captured output, or a color-blind operator's terminal, without any additional design work required.

---

## 8. Admin/Data Management

This document **is** the admin/data-management specification for this feature — there is no further admin-facing configuration screen for it, since the moment an admin account exists, this script has finished its entire job permanently for that environment (§2.3 idempotency). Ongoing admin-account management (adding a second admin, revoking one) is out of scope for this document and belongs to whatever in-app admin-invite feature the team builds later (§1.3) — not to this bootstrap script, which should remain a single-purpose, first-run-only tool.

---

## 9. "Component" Inventory — Developer Handoff

Reframed for a CLI artifact rather than a UI:

| Unit | Type | Responsibility |
|---|---|---|
| `scripts/createAdmin.js` | Node.js script (entry point) | Orchestrates §2.2's steps in order, prints §4's output at each stage |
| Env-var validation | Function/module | Checks `ADMIN_EMAIL`, `ADMIN_PASSWORD`, `MONGODB_URI` present before proceeding (§4.5) |
| Existing-admin check | DB query | `users.findOne({ role: 'admin' })` — powers both §2.3's idempotency and §4.4's failure message |
| Password strength check | Function | Enforces the 12-character admin-specific minimum (§4.6, NEW) |
| Confirmation prompt | CLI interaction | Implements §4.3, skippable via `--yes` flag |

No shared UI component library, no Toast, no design tokens — this script is a standalone, dependency-light utility by design.

---

## Appendix A — Quick Reference: What's New vs. Reused

| New for this feature | Reused as-is from existing docs |
|---|---|
| Admin-specific 12-character password minimum (§4.6) — stricter than customer/shop signup, explicitly flagged as new | `scripts/createAdmin.js` itself, its env-var contract, and its "never over HTTP" constraint — all already defined (§3.5A, §9, Deliverables list, Master Build Document) |
| Exact terminal output copy and structure (§4 of this document) | `users` schema, unchanged — no admin-specific schema fork (§4.1) |
| Confirmation-prompt step with masked-connection-string display (§4.3) | bcrypt password hashing, unchanged (§3.6) |
| Idempotency check and its specific failure message (§2.3, §4.4) | `.env.example`'s existing `ADMIN_EMAIL`/`ADMIN_PASSWORD` variables, unchanged (§9) |

---

## Appendix B — Explicit Non-Recommendation

For the avoidance of doubt, having now specified this feature in full: **do not build a web-reachable version of this flow**, regardless of how it might be gated (secret token, "only if zero admins exist" check, IP allowlist, or otherwise). If a future requirement genuinely needs a friendlier bootstrap experience, the safer evolution is a slightly nicer **interactive CLI** (e.g. prompting for the email/password interactively instead of requiring env vars, still run manually, still never over HTTP) — not a browser form. This appendix exists so that a future reader skimming only the section headers still encounters the one instruction in this whole document most important not to miss.

---

*— End of specification —*
