# Review brief — Arts Access Miami Grantee Portal

This file is written for another AI (or a human reviewer in a hurry) to read `index.html` cold and give useful feedback. It explains what the code is trying to be, how it's structured, which rough edges are *known and accepted* vs. which are open questions, and what kind of feedback would actually be useful.

## What you're looking at

A single ~5,400-line `index.html` — markup, CSS, and one vanilla-JS (ES5-style) script in an IIFE — implementing a prototype grantee-management portal for a (fictional-context) arts-education nonprofit, "Arts Access Miami." It has two roles (grantee org staff, and Arts Access Miami staff/admin) and covers reporting, finance, a community feed, a directory, calendar/meetings, attendance, and account/org administration.

**It is explicitly a front-end-only prototype**, built in one long iterative session as a series of small, specific feature requests (think: "add X," "change Y," "remove Z") rather than a single up-front design. There is no backend: all state lives in one `db` object in `localStorage`, seeded on first load. This matters for calibrating feedback — see "What not to flag" below.

## How to navigate the file efficiently

Don't read linearly. The script is organized into clearly marked sections (`/* ===== NAME ===== */`); use these as your map:

| Section comment | Roughly | What's there |
|---|---|---|
| `TOKENS` / `BASE` / `SHELL` | lines 5–280 | CSS variables, resets, layout shell |
| `DATA` | ~1808–1987 | Static reference constants (`GRANTEE_ORGS`, `ORG_PROFILES`, `REPORT_STAGES`, `DISCIPLINES`, etc.) |
| `seedDB()` | 1987–2195 | The entire initial dataset |
| `loadDB()` / `saveDB()` | 2195–2204 | The only two functions that touch `localStorage` |
| ~2204–2380 | | **Migration guards** — one `if (!db.x) {...}` per schema change made over the project's life (see below) |
| `NAV` | 2644– | `isSuperAdmin()`, `canManageFinance()`, `activeGranteeOrgs()`, `navigate(view)` — the permission and routing core |
| `REPORTING` (render) | 3541– | Per-stage, per-item submission review |
| `GRANTEES DIRECTORY` | 4131– | `openGranteeProfile()` — the admin org-detail modal, including org removal |
| `FINANCE` | 4792– | Grant amounts/disbursements, gated by `canManageFinance()` |
| `ADMIN ACCOUNTS` | 5252– | User management + removed-org restore, gated by `isSuperAdmin()` |

Grep for `function renderX` to find any view's render function; grep for `data-` to find the attribute a render function's output uses to wire up its own click handlers (see pattern below).

## Core patterns (so you don't mistake them for bugs)

1. **One `db` object, one `saveDB(db)` after every mutation.** No diffing, no transactions — a function that changes state always re-renders itself or calls the relevant `renderX()` afterward.
2. **Render-then-bind.** Every `renderX()` sets `innerHTML` from a data array, then immediately does `$all("[data-foo]", container).forEach(el => el.addEventListener(...))` to re-wire handlers, because `innerHTML` destroys previous listeners. This is the single most repeated pattern in the file.
3. **Migration guards.** Because real users' `localStorage` persists across code changes, every new/reshaped field got a one-time guard like `if (!db.removedOrgs) { db.removedOrgs = []; saveDB(db); }` placed near the others, rather than a versioned migration system. There are ~20 of these by now.
4. **Permission = a boolean helper + hide the control + re-check in the handler.** E.g. `canManageFinance()` is called to (a) decide whether to render a button at all, and (b) guard the top of the click handler as a backstop. There's no capability/ACL abstraction — every gated feature re-derives its own check inline or via one of a handful of named helper functions (`isSuperAdmin`, `canManageFinance`, `canSeeDisbursementNotif`).
5. **No `confirm()`.** The deployment environment's dialogs don't fire reliably, so destructive actions either have no confirmation (most deletes — "Remove account," "Remove disbursement," etc., matching how the rest of the app already behaved) or a two-click "click again to confirm" pattern with a timeout (only used once so far, for removing an organization — see `pendingRemoveOrgConfirm` in `openGranteeProfile`).
6. **Soft delete for organizations, hard delete for most other list items.** Removing an org (`removeOrgFromPortal`) keeps its historical records (reports, posts, disbursements) in `db` but hides it from every enumeration via `activeGranteeOrgs()`, and hard-deletes its grantee login accounts and pending invites. Restoring (`restoreOrgToPortal`) does not recreate those accounts/invites. Almost everything else (contacts, documents, disbursements, accounts) is a straight `.filter()` hard delete.

## What not to flag (known, deliberate)

- **No real auth, backend, or email/push delivery.** Login is a `localStorage` account list checked client-side; "admin" email allowlisting (`ADMIN_ALLOWED_EMAILS`) is a UI gate, not security. Notification preferences and "email sent" toasts are simulated.
- **Plaintext passwords in `db.accounts`.** Known, not a target for fixing in this prototype.
- **`localStorage`-only persistence.** No cross-device sync, no multi-user concurrency handling. Two admins in two tabs can clobber each other's changes silently.
- **One giant file, one IIFE, no build step, no modules.** Deliberate, for a single-file Claude Artifact that has to be copy-pasteable/publishable as-is.
- **ES5-ish style** (`var`, `function(){}`) even though `.map/.filter/.forEach` are used freely — a stylistic choice made early in the project and kept for consistency, not a technical constraint.

## Open questions — this is where feedback would actually help

1. **Consistency drift.** Because features were added incrementally over many separate requests, similar problems were sometimes solved slightly differently each time (e.g., different destructive-action UX between "remove account" with no confirmation vs. "remove organization" with a two-click confirm; different filter-toolbar markup reused in some places but hand-rolled in others). Worth a pass for places where the same UI problem has two different solutions in the codebase.
2. **The permission model is purely additive and ad hoc.** `isSuperAdmin()`, `canManageFinance()`, `canSeeDisbursementNotif()`, `data-admin-only`, `data-super-admin-only`, `nav-admin`/`nav-grantee` CSS classes — five or six different gating mechanisms exist side by side. Is this a problem worth consolidating, or fine for a file this size?
3. **`activeGranteeOrgs()` is called dozens of times**, often inside render functions that run on every keystroke/filter change (`.filter()` over a small constant array each time) — fine at this scale, but flag if you see it compounding somewhere expensive.
4. **Known stale-state edge case:** a handful of module-level `var adminFinanceOrg = activeGranteeOrgs()[0]`-style defaults don't get reset if that exact org is later removed mid-session. Low priority, but is there a cheap general fix (e.g., re-derive on every render instead of caching) that's worth applying everywhere rather than patching case-by-case?
5. **Is the migration-guard approach ((~20 and growing) going to become unmanageable, and if so, what's the smallest change that would help** (a single versioned migration list? a schema version number with a switch statement?) without a real backend to lean on?
6. **Accessibility** hasn't been deliberately audited at any point — icons are mostly inline SVGs with `aria-label` on icon-only buttons, but no systematic review has been done (focus management on modal open/close, keyboard nav through the many `season-tab` bars, color contrast on status chips, etc.).
7. **Anything that looks like a real bug**, obviously — this file has grown through ~100+ incremental edits across one long session, each individually tested with a Node.js syntax check and a grep-based consistency sweep, but never through an end-to-end pass of the whole app.

## What would be useful back

- A short, prioritized list (not an exhaustive line-by-line review) — correctness issues first, then consistency/maintainability, then anything else.
- If you flag something, point to the specific function or `data-` attribute rather than a line number alone (line numbers will drift as the file keeps changing).
- Assume the author already knows about the items in "What not to flag" above — feedback repeating those isn't useful; feedback on the open questions, or on something not listed here, is.
