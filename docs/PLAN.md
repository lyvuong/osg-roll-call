# OSG Roll Call PWA — Implementation Plan

## Context
OSG Communities (Neon CRM) needs a fast roll-call app for event attendance and hours. Neon's API v2 (spec v2.12, verified) can look up accounts/events but `EventAttendee.markedAttended` is read-only and there is no per-event "hours" field, so Neon cannot be the record for roll call. Decisions made with the user:
- **Hours model**: hours are an **event-level duration**; total volunteer hours = (# volunteers present) × event duration. No per-person hour entry.
- **Record**: Cloudflare D1. **Neon**: read lookups + (phase 3) volunteer time sheets for volunteers' hours.
- **Auth**: Firebase Auth (existing Firebase project); Worker verifies ID tokens; role allow-list in Firestore.
- **Roles** (member / volunteer / parent / caretaker): tagged locally once, at first entry.
- **New people**: stored locally in D1 with `neon_account_id = NULL` ("not in Neon"); no Neon writes for accounts.

## Architecture
- **Cloudflare Worker** (Hono, TypeScript) serves `/api/*` and static PWA assets (Workers Static Assets; Pages not needed, one deploy unit).
- **PWA**: Vite + Preact (small bundle), service worker via vite-plugin-pwa, offline cache of the shell + people directory; roll-call submissions queued in IndexedDB and replayed when online.
- **D1** (free tier is ample): attendance data and cached people directory.
- **Neon proxy**: Worker holds `NEON_ORG_ID` / `NEON_API_KEY` as secrets (Basic auth to `https://api.neoncrm.com/v2`); the browser never sees them. Throttle + cache (events list cached in D1/KV for minutes; people directory synced on schedule).
- **Firebase**: client SDK for sign-in; Worker verifies JWT with Google JWKS (no Admin SDK needed on Workers). Firestore `users/{uid}` doc holds `role: "recorder" | "admin"`; read via REST with a service account, or simpler, use Firebase custom claims set once by admin script (preferred — no Firestore reads per request).

## D1 schema (migrations/0001_init.sql)
- `person(id, neon_account_id NULL UNIQUE, first_name, last_name, role, email, phone, active, created_at)` — role ∈ member|volunteer|parent|caretaker
- `person_link(parent_id, child_id)` — parent/caretaker ↔ member (for "bring the family" suggestions)
- `event(id, neon_event_id NULL, name, event_date, start_time, duration_hours, created_by)` — local copy; may be created from Neon event or ad hoc
- `attendance(event_id, person_id, role_at_event, recorded_by, updated_at, PRIMARY KEY(event_id, person_id))` — `role_at_event` snapshots the role so past totals stay stable
- Derived: `volunteer_hours(event) = COUNT(attendance where role_at_event='volunteer') × event.duration_hours` (computed in query, never stored)
- `attendance_audit(id, event_id, person_id, action, user, ts)`
- Indexes: `attendance(person_id, event_id)`, `event(event_date)`, `person(last_name, first_name)`.

## API (Worker)
- `GET /api/events?from=&to=` — date-range history (D1); `POST /api/events` — create from Neon event pick or ad hoc
- `GET /api/neon/events/upcoming` — proxy to `POST /events/search` (cached)
- `GET /api/events/:id/roster` — attendance + **suggestions**: people ranked by attendance count in the last 90 days (and same event series/name), pre-grouped by role
- `PUT /api/events/:id/attendance` — body `{present: personId[]}` replaces the set (diffed server-side); writes audit rows
- `PATCH /api/events/:id` — edit name/date/`duration_hours`
- `GET /api/people/search?q=` — partial match on name (D1 `LIKE`, directory cached from Neon + local-only)
- `POST /api/people` — first-time participant (name, role, optional linked parent); dedupe check on normalized name before insert
- `POST /api/admin/sync-people` + cron trigger (daily) — pull Neon accounts via `POST /accounts/search` (outputFields: Account ID, First/Last Name, Email; paginate 200) and upsert by `neon_account_id`
- Phase 3: `POST /api/events/:id/push-timesheets` — for each volunteer present with `neon_account_id`, create a `/timeSheets` item of `duration_hours` (requires shiftId/roleId; see Risks)

## UI (mobile-first, one-hand use)
1. **Roll call screen** (default): pick event (today's pre-selected) → chips grouped by role, **frequent people pre-listed with checkboxes**; one tap = present; event **duration (hours)** is set once at the top; sticky footer shows "N present · M volunteers × H h = T volunteer hours"; Save.
2. **Search box** at top: instant partial match; tap to add to the list. "Not found → Add new" opens a 3-field sheet (first, last, role) — only used the first time ever.
3. **History**: date-range picker → events list (with per-event and range-total volunteer hours) → open to view/edit roster and duration (same component as roll call; edits audited).
4. Settings/Admin (admin only): sync from Neon, merge duplicate people, export CSV.

## Files to create (project root `C:\Users\lyvuo\Documents\Projects\OSG\Roll Call`)
- `wrangler.jsonc`, `package.json`, `tsconfig.json`
- `worker/index.ts`, `worker/routes/{events,attendance,people,neon,admin}.ts`, `worker/lib/{auth,neon,suggest}.ts`
- `migrations/0001_init.sql`
- `app/` — `main.tsx`, `pages/{RollCall,History,Admin}.tsx`, `lib/{api,offlineQueue,firebase}.ts`, `manifest.webmanifest`, `sw.ts`
- `README.md` — comprehensive project documentation (overview, features, architecture diagram, data model, API reference, Neon integration notes incl. read-only `markedAttended` and time-sheet limits, privacy stance: no DOB/address in D1, prerequisites, local dev, secrets/`wrangler secret`, D1 migrations, Firebase setup + admin claims script, deploy, testing, PWA install, backup/export, troubleshooting, roadmap/phases, contributing). Plus `.gitignore` (node_modules, .wrangler, .dev.vars, dist). Both are committed along with this plan.

## Phases
1. **MVP**: schema, auth, people search/add, roll call + hours, history by date range/edit, PWA install + offline queue. (No Neon yet — seed people via CSV import.) Includes totals of volunteer hours per range.
2. **Neon read integration**: people directory sync, upcoming events, Neon event linking.
3. **Neon time sheets** for volunteers (optional; validate feasibility first).

## Risks / open items
- Neon time sheets are weekly per account and items require `shiftId`/`roleId` — volunteers must be assigned to an opportunity shift. Phase 3 needs a spike with a sandbox account; fallback is CSV export.
- Neon rate limits: confirm current limit and keep all Neon calls server-side with caching.
- Need from user: Neon Org ID + API key for a read-scoped system user (set via `wrangler secret`, never committed), Firebase project config, and a Neon sandbox if available.
- Child data: names of minors stored in D1 — restrict to authorized users. **D1 stores no DOB or address** (confirmed by user); the Neon sync's `outputFields` must request only Account ID, first/last name, email, phone.

## Verification
- `wrangler dev` + Miniflare D1: unit tests (vitest) for suggestion ranking, volunteer-hours computation, upsert/audit, date-range queries, name dedupe.
- Mock Neon responses (recorded from spec shapes) for sync tests; one manual run against the real Neon API with read-only key.
- Browser test via preview: sign in, record roll call on mobile viewport, go offline → submit → reconnect and confirm replay; edit past event (roster and duration) and verify audit row and recomputed volunteer-hour totals.
- Lighthouse PWA check; deploy with `wrangler deploy`, smoke test on a phone.
