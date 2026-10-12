# OSG Roll Call

A mobile-first Progressive Web App (PWA) for taking roll call at [OSG Communities](https://osgcommunities.org/) events. It integrates with [Neon CRM](https://developer.neoncrm.com/api-v2/) for people and event lookup, and runs on Cloudflare Workers + D1.

> **Status: planning.** The design below has been written up for committee review. No application code exists yet. See [docs/PLAN.md](docs/PLAN.md) for the implementation plan.

## Table of contents
1. [Goals](#goals)
2. [Key decisions](#key-decisions)
3. [Features](#features)
4. [Architecture](#architecture)
5. [Data model](#data-model)
6. [API reference (planned)](#api-reference-planned)
7. [Neon CRM integration notes](#neon-crm-integration-notes)
8. [Privacy and security](#privacy-and-security)
9. [Prerequisites](#prerequisites)
10. [Local development](#local-development)
11. [Configuration and secrets](#configuration-and-secrets)
12. [Database migrations](#database-migrations)
13. [Firebase setup](#firebase-setup)
14. [Deployment](#deployment)
15. [Testing](#testing)
16. [Installing the PWA](#installing-the-pwa)
17. [Backup and export](#backup-and-export)
18. [Troubleshooting](#troubleshooting)
19. [Roadmap](#roadmap)
20. [Contributing](#contributing)

## Goals
- Let authorized users record who attended an event, quickly, on a phone.
- Participants are members, volunteers, parents, or caretakers of members.
- Record the event duration and derive total volunteer hours.
- Frequent participants are suggested as tick-boxes; infrequent ones are found by partial-name search.
- A person's name is typed **only once ever**, the first time they attend.
- Authorized users can view and edit past events, retrieved by date range.

## Key decisions
| Topic | Decision | Why |
|---|---|---|
| Source of truth | Cloudflare **D1** | Neon's API exposes attendee `markedAttended` as read-only and has no per-event hours field. |
| Neon's role | Read-only lookup of accounts and events; optional volunteer time-sheet push (phase 3) | Avoids writing to the CRM; keeps roll call fast and independent of Neon availability. |
| Hours model | Event-level duration. **Total volunteer hours = volunteers present × event duration** | No per-person hour entry. |
| Authentication | **Firebase Auth** (existing Firebase project) | Already available; Worker verifies ID tokens. |
| Roles | Tagged locally once (member / volunteer / parent / caretaker) | Independent of how Neon is configured. |
| New people | Stored in D1 with no Neon account id, flagged "not in Neon" | No Neon writes, no duplicate-account risk. |
| Hosting | Cloudflare Workers (static assets + API) and D1 free tier | Low cost, single deploy unit. |

## Features
- **Roll call screen:** pick today's event; people appear grouped by role with the most frequent attendees pre-listed as one-tap check-boxes.
- **Instant search:** partial name match over everyone known; tap to add to the list.
- **Add new person:** first name, last name, role — used only the first time.
- **Duration and totals:** set the event duration once; the footer shows `N present · M volunteers × H h = T volunteer hours`.
- **History:** choose a date range, list events with per-event and range volunteer-hour totals, open any event to edit its roster or duration.
- **Audit trail:** every add/remove of an attendee is logged with user and timestamp.
- **Offline-tolerant:** submissions are queued on the device and replayed when back online.
- **Admin tools:** sync people from Neon, merge duplicates, export CSV.

## Architecture
```
 Phone / tablet (PWA)
   Preact UI · service worker · IndexedDB offline queue
        │  HTTPS, Firebase ID token
        ▼
 Cloudflare Worker (Hono, TypeScript)
   ├─ /api/*  → auth (Firebase JWT via Google JWKS)
   ├─ D1      → events, people, attendance, audit
   ├─ Neon API proxy (Basic auth, secrets held server-side, cached/throttled)
   └─ Cron    → daily people sync
        │
        ▼
 Neon CRM API v2   https://api.neoncrm.com/v2
```
- The browser never sees Neon credentials.
- Firestore/Firebase is used only for sign-in and user role claims (`recorder` or `admin`).

### Repository layout (planned)
```
worker/            Worker entry, routes, lib (auth, neon, suggest)
app/               PWA source (pages, lib, manifest, service worker)
migrations/        D1 SQL migrations
docs/PLAN.md       Implementation plan
wrangler.jsonc     Cloudflare configuration
```

## Data model
D1 (SQLite). **No date of birth or address is stored.**

| Table | Columns |
|---|---|
| `person` | `id`, `neon_account_id` (nullable, unique), `first_name`, `last_name`, `role` (member\|volunteer\|parent\|caretaker), `email`, `phone`, `active`, `created_at` |
| `person_link` | `parent_id`, `child_id` — links a parent/caretaker to a member for family suggestions |
| `event` | `id`, `neon_event_id` (nullable), `name`, `event_date`, `start_time`, `duration_hours`, `created_by` |
| `attendance` | `event_id`, `person_id`, `role_at_event`, `recorded_by`, `updated_at` — PK `(event_id, person_id)` |
| `attendance_audit` | `id`, `event_id`, `person_id`, `action`, `user`, `ts` |

- `role_at_event` snapshots the person's role so past totals do not change if someone's role changes.
- Volunteer hours are **computed, never stored**: `COUNT(attendance WHERE role_at_event = 'volunteer') × event.duration_hours`.
- Indexes: `attendance(person_id, event_id)`, `event(event_date)`, `person(last_name, first_name)`.

## API reference (planned)
All routes require a valid Firebase ID token (`Authorization: Bearer <token>`). Admin routes require the `admin` claim.

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/events?from=&to=` | Events in a date range with volunteer-hour totals |
| POST | `/api/events` | Create an event (from a Neon event or ad hoc) |
| PATCH | `/api/events/:id` | Edit name, date, duration |
| GET | `/api/events/:id/roster` | Current attendance plus ranked suggestions |
| PUT | `/api/events/:id/attendance` | Body `{ "present": [personId, ...] }`; replaces the set, writes audit rows |
| GET | `/api/neon/events/upcoming` | Upcoming events from Neon (cached) |
| GET | `/api/people/search?q=` | Partial name search |
| POST | `/api/people` | Add a first-time participant (name, role, optional linked parent); checks for duplicates |
| POST | `/api/admin/sync-people` | Pull accounts from Neon and upsert (also runs daily by cron) |
| POST | `/api/events/:id/push-timesheets` | *Phase 3:* send volunteer hours to Neon time sheets |

**Suggestions:** people ranked by attendance count over the last 90 days and by prior attendance at events with the same name, grouped by role.

## Neon CRM integration notes
Verified against the Neon API v2 OpenAPI spec (v2.12).

- **Base URL:** `https://api.neoncrm.com/v2`; HTTP Basic auth with Org ID as username and API key as password.
- **Accounts:** `POST /accounts/search` (searchFields, outputFields, pagination; max page size 200).
- **Events:** `POST /events/search`, `GET /events/{id}`, `GET /events/{id}/attendees`.
- **Limitations that drive the design:**
  - `EventAttendee.markedAttended` is returned but there is no documented write path, so attendance cannot be recorded in Neon through the API.
  - There is no per-event volunteer-hours field.
  - `/timeSheets` items are weekly per account and require `shiftId` and `roleId`, so volunteers must be assigned to an opportunity shift. Phase 3 needs a spike with a sandbox account; CSV export is the fallback.
- **Sync data minimization:** request only Account ID, first name, last name, email, and phone as output fields.
- Use a dedicated Neon **system user with the least privilege** needed (read accounts and events).

## Privacy and security
- Minors' names are stored; access is limited to authorized, signed-in users.
- No date of birth, address, payment, or membership-fee data is stored in D1.
- Neon and Firebase secrets live only in Worker secrets / `.dev.vars` (git-ignored); never commit them.
- All Neon calls are server-side; the client only talks to the Worker.
- Audit rows record who changed attendance and when.
- Review who holds the `recorder` and `admin` claims regularly.

## Prerequisites
- Node.js 20+ and npm
- A Cloudflare account (Workers and D1 free tier) and `wrangler` (`npm i -g wrangler` or via `npx`)
- A Firebase project with Authentication enabled
- A Neon CRM Org ID and an API key for a system user (see Neon's [authentication guide](https://developer.neoncrm.com/authentication/))

## Local development
Application code is not yet committed; these are the intended steps.

```bash
npm install
cp .dev.vars.example .dev.vars        # fill in secrets
npx wrangler d1 migrations apply osg-roll-call --local
npm run dev                            # wrangler dev + Vite
```

## Configuration and secrets
| Name | Where | Description |
|---|---|---|
| `NEON_ORG_ID` | Worker secret | Neon organization id |
| `NEON_API_KEY` | Worker secret | Neon API key (read-scoped system user) |
| `FIREBASE_PROJECT_ID` | Worker var | Used to validate token audience/issuer |
| `VITE_FIREBASE_*` | App build env | Public Firebase web config |

```bash
npx wrangler secret put NEON_ORG_ID
npx wrangler secret put NEON_API_KEY
```

## Database migrations
```bash
npx wrangler d1 create osg-roll-call
# put the database_id into wrangler.jsonc
npx wrangler d1 migrations apply osg-roll-call --remote
```

## Firebase setup
1. Create or reuse a Firebase project and enable a sign-in provider (e.g., Google or email link).
2. Create users, then assign roles with a one-off admin script that sets custom claims, e.g. `{ role: "recorder" }` or `{ role: "admin" }`.
3. The Worker verifies the ID token signature against Google's JWKS and reads the role claim; no Firestore read per request.

## Deployment
```bash
npm run build
npx wrangler deploy
```
Configure a custom domain in the Cloudflare dashboard if desired, and add it to Firebase's authorized domains.

## Testing
- Unit tests (Vitest with Miniflare D1): suggestion ranking, volunteer-hour calculation, attendance upsert and audit, date-range queries, duplicate-name detection.
- Neon sync tests use recorded/mocked responses; one manual run against the real API with a read-only key.
- Manual: record roll call on a phone viewport, go offline, submit, reconnect and confirm replay; edit a past event and verify the audit row and totals.
- Lighthouse PWA audit before release.

## Installing the PWA
Open the deployed URL on a phone, then use *Add to Home Screen* (iOS Safari) or *Install app* (Chrome/Android).

## Backup and export
- CSV export of events and attendance from the Admin page.
- D1 Time Travel and `wrangler d1 export osg-roll-call --remote --output backup.sql` for full backups.

## Troubleshooting
| Symptom | Likely cause |
|---|---|
| 401 from `/api/*` | Expired Firebase token or missing role claim |
| Neon calls fail with 401/403 | Wrong Org ID/API key or system user lacks permission |
| Neon calls return 429 | Rate limit; rely on cache and reduce sync frequency |
| Person appears twice | First-time entry before Neon sync; merge in Admin |
| Changes not visible offline | Queue replays only after the device reconnects |

## Roadmap
1. **MVP:** schema, auth, people search/add, roll call and duration, history by date range with edit, PWA install and offline queue, CSV people import, volunteer-hour totals.
2. **Neon read integration:** scheduled people sync, upcoming events, link local events to Neon events.
3. **Neon time sheets:** push volunteer hours (feasibility spike required).

## Contributing
- Work on feature branches and open pull requests against `main`.
- Never commit secrets or real participant data.
- Keep the plan in [docs/PLAN.md](docs/PLAN.md) and this README in sync with design changes.

## License
To be determined by OSG Communities.
