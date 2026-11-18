# Arts Access Miami — Grantee Portal

A single-file, self-contained prototype of a grantee management portal for Arts Access Miami, a nonprofit arts education collective. It simulates how Arts Access Miami staff ("admin" accounts) and grantee organizations ("grantee" accounts) would manage reporting, finance, events, community updates, and resources in one shared platform.

**Live demo:** https://claude.ai/code/artifact/2b8a2416-33df-481a-abfd-a7aa3484f2a9

## What this is (and isn't)

This is a **front-end-only prototype**. Everything — accounts, reports, finance records, posts, documents — lives in the browser's `localStorage`. There is no server, no database, no real authentication, and no real email delivery. It's built to demonstrate and iterate on UX/workflow ideas quickly, not to run as a production system.

- All of `index.html` is one file: markup, CSS, and a single vanilla-JS (ES5-style) script, wrapped in an IIFE.
- State is a single `db` object, seeded on first load (`seedDB()`) and persisted via `localStorage.setItem` after every change (`saveDB()`).
- "Sending an email" or "uploading a file" is simulated with a toast notification unless explicitly wired to a real asset (see **Real file hosting** below).

## Running it

Just open `index.html` in a browser — no build step, no dependencies, no server. Or open the published Artifact link above.

## Demo logins

| Role | Email | Password |
|---|---|---|
| Admin (super admin) | `operations@artsaccessmiami.org` | `demo-password` |
| Admin (other staff) | `director@` / `impact@` / `programs@` / `marketing@` / `info@artsaccessmiami.org` | `demo-password` |
| Grantee (org admin) | `fabiola@littlehaitiarts.org` (or any seeded org's contact email — see `ORG_PROFILES`) | `demo-password` |

The login screen has "Fill demo credentials" buttons for both the Grantee and Staff tabs.

**Admin login is restricted** to the six `@artsaccessmiami.org` addresses above (see `ADMIN_ALLOWED_EMAILS`) — no other email can log in or be granted the admin role, even via invite.

## Roles & permission tiers

- **Grantee** — a staff member at a partner organization. Sees their own org's data only.
  - **Org admin** (`orgAdmin: true`) — a subset of grantee users who can manage their org's contacts, finance, and reporting. Every org has at least one.
- **Admin** — Arts Access Miami staff. Sees all organizations.
  - **Super admin** (`operations@artsaccessmiami.org` only) — the only login that can access **Accounts** (user management, including removing/restoring organizations).
  - **Finance managers** (super admin + `director@artsaccessmiami.org`) — the only logins that can edit grant amounts, add/delete disbursements, mark payments sent, or delete finance documents. Other admins can view Finance but not modify it.

Permission checks are implemented as small helper functions (`isSuperAdmin()`, `canManageFinance()`) and enforced both by hiding the relevant buttons/nav items and by re-checking inside the action handlers.

## Feature tour

**Grantee view**
- **Dashboard** — upcoming deadlines, events/meetings, and a feed preview. Stat tiles link to their respective sections.
- **Calendar** — embedded Google Calendar, plus an "Add to calendar" flow for deadlines/events/meetings tracked in-app.
- **Community Feed** — org-level posts with a required subject line, optional photo, likes, and comments. Orgs can edit/delete their own posts.
- **Photo Gallery** — admin-created "events"; each links out to an externally-hosted photo album (no in-app photo uploads).
- **Directory** — browse every grantee org's public profile (contact, programs, disciplines, schools, feeder patterns).
- **Reporting** — Pre-Report / Mid-Term / Final stages, each with a checklist of required items; admins review and approve/reject individual items (not the whole submission).
- **Finance** — grant amount and disbursement schedule (read-only for grantees), finance document folder.
- **Resources** — "Arts Access Resources" (org-wide info, marketing toolkit, the **More Information** document category) and "My Organization" (contacts, team accounts, grant documents, School Photoshoot album links, invite requests).
- **Settings** — theme, notification category toggles (meetings/deadlines, feed posts, disbursements, albums — grant disbursements only shown to admins/org admins), privacy (contact visibility in the Directory), account.
- **Notifications bell** (top of sidebar) — in-app notifications, e.g. when an admin rejects a report item.

**Admin view**
- **Admin Dashboard** — portfolio-wide stats (all clickable, deep-linking to the relevant section) and a "Grantee compliance" list.
- **Grantees Directory** — click any org card to open its full profile: public directory info, outstanding Reporting/Attendance/Posts status, and (super admin only) a **Danger Zone** to remove the organization from the portal.
- **Accounts** *(super admin only)* — manage every login (add/edit/remove, change role/org), with search + filter by name/role/org. Includes the **Approvals** sub-tab (pending calendar entries, invite requests, invite management) and a **Removed Organizations** list with one-click restore.
- **Manage Contacts & Documents** — program contacts, grant documents, School Photoshoot album links per org/season/region.
- **Finance** — grant amounts & disbursements per org (edit gated to super admin + director), finance documents.
- **Attendance** — track which orgs attended each meeting.
- **Reporting review** — per-org, per-stage submission review with per-item approve/reject and reviewer notes.

## Data model

Everything hangs off one `db` object (see `seedDB()` for the full shape), the main collections being:

`events`, `photos`, `posts`, `comments`, `documents`, `contacts`, `reports`, `albums` (Gallery events), `photoshootAlbums` (School Photoshoot links), `accounts`, `invites`, `financeDocuments`, `orgContacts`, `grantAmounts`, `disbursements`, `meetingAttendance`, `inviteRequests`, `marketingMaterials`, `meetingMaterials`, `orgProfiles`, `notifications`, `removedOrgs`.

Static reference data (not persisted, defined as JS constants) includes `GRANTEE_ORGS`, `ORG_PROFILES`, `ORG_TIERS`, `ORG_REGIONS`, `COHORTS`, `DISCIPLINES`, `REPORT_STAGES`.

**Removing an organization** (super admin only) hides it from every active list in the app and revokes its grantee logins/pending invites, but keeps historical records (past reports, disbursements, posts) intact — "soft delete." Restoring an org makes it active again; logins/invites are not automatically recreated.

### Schema migrations

Because state lives in each browser's `localStorage`, every new field or reshaped collection added during development needs a one-time migration guard (pattern: `if (!db.someField) { db.someField = ...; saveDB(db); }`) near the top of the script, so existing saved sessions pick up new features without losing data.

## Real file hosting

Most "documents" in the app are mock metadata with a simulated download. One exception: the **New Partners Information Packet** PDF is a real file, uploaded as an Artifact asset and linked directly — see the `assets` capability declared on the published Artifact.

## Known limitations

- No real backend — no real authentication, email delivery, or push notifications. Settings toggles and "email" notifications are UI-only / simulated.
- State is per-browser (`localStorage`), not shared across devices or synced between admin and grantee sessions in real time.
- A handful of org-selector variables (e.g., the currently-selected org in Finance/Reporting/Photoshoots) don't automatically reset if that exact org is removed mid-session.
