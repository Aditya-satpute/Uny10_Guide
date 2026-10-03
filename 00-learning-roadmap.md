# Uny10 Technical Interview Learning Roadmap

Concise study order for understanding the codebase. This file is a roadmap only; detailed notes come next.

## 1. Product and Architecture

- Business purpose: team-based sponsorship/CRM workspace for companies, contacts, outreach, visits, meetings, documents, and coordination.
- High-level shape: Next.js 16 App Router monolith; React 19 UI; Next.js Route Handlers; MongoDB; browser-managed session state backed by server validation.
- Primary boundary: browser page/component → shared client helper or direct `fetch` → `/api/*` route handler → auth/team gate → DB/repository layer → MongoDB.
- Study files: `app/`, `components/`, `lib/`, `middleware.ts`, `package.json`, `next.config.mjs`.

## 2. Folder and Module Structure

- `app/`: route tree, pages, root layout, global styles, API handlers.
- `components/`: feature components (`dashboard`, `companies`, `meetings`, `calendar`, `map`, `documents`, `support`, `profile`) and reusable `ui/` primitives.
- `lib/`: auth context/fetching, API client, state stores, actions, DB repositories, Mongoose models, validation, scoping, notifications, document processing.
- `scripts/`: seed/check/update utilities for users, meetings, and database maintenance.
- `public/`: brand assets, PWA manifest/service worker, icons.
- Root Markdown files and `docs/interview/`: existing implementation/setup/reference material to cross-check against source.

## 3. Frontend and Next.js Concepts

- App Router page map: dashboard, companies/detail, calendar, meetings, map, documents, mail, profile, settings, support, login, onboarding, waiting room, sponsors.
- Server/Client Component split; client interactivity for forms, dashboards, maps, charts, dialogs, uploads, and auth state.
- Root providers/layout: theme, auth, analytics, Microsoft Clarity, development agentation, PWA registration.
- UI system: Tailwind/PostCSS styling, Radix-based primitives, Lucide icons, Recharts, date utilities, responsive/mobile layouts.
- State: React local state, `AuthProvider`, localStorage/sessionStorage, custom `auth_user_updated` event, Zustand company store.
- Important frontend utilities: `lib/api-client.ts`, `lib/auth-fetch.ts`, `lib/company-store.ts`, `components/auth/auth-route-guard.tsx`.

## 4. Backend and API Architecture

- Route handlers under `app/api/` and their HTTP contracts/status codes.
- API groups: `auth`, `companies`, `meetings`, `calendar`, `visit-plans`, `teams`, `team-members`, `users`, `notifications`, `support`, `documents`, `expand-url`.
- Shared backend gates: `middleware.ts`, `getSessionUserId`, `resolveTeamIdFromRequest`, `hasActiveTeamAccess`, `enforceTeamAccess`.
- Error and input patterns: JSON responses, route-level validation/normalization, ObjectId handling, role/status checks, sanitization and SSRF protection where applicable.
- Compare the generic `apiClient`/`authFetch` path with feature-specific direct `fetch` calls.

## 5. Database and Data Model

- MongoDB database (default `omnispons`) and environment-configurable collection names.
- Core entities: `Users`, `Teams`, `TeamMembers`, `companies`, `meetings`, `calendar`, `VisitPlans`, `Notifications`, `Support`.
- Mongoose models: user, company, meeting, calendar, visit plan, notification; native MongoDB repositories for several feature domains.
- Embedded data: company contacts/follow-ups; meeting attendees/RSVPs/approval metadata; visit-plan assigned companies; calendar event arrays.
- Referenced-by-ID data: meetings/visits linked to companies and calendar entries; users linked to teams/team-memberships.
- Indexes, compound team/date/status queries, soft-active flags, unique visit-plan constraint, and serverless connection caching.
- Read: `lib/models/*`, `lib/db/*`, `lib/mongodb.ts`, `lib/db/mongodb.ts`, `MONGODB_SETUP.md`.

## 6. Authentication, Authorization, and Multi-Tenancy

- Google credential flow: Google sign-in → `/api/auth/google-callback` → find/create/reconcile user → onboarding or app redirect.
- Session model: random refresh token stored in an HttpOnly cookie and persisted with expiry on the user record; `/api/auth/session` validates it server-side.
- Lifecycle: login, logout, session refresh, expiry/redirect, pending approval, rejection, and team switching.
- Team onboarding: create team as coordinator or join by code as pending member; coordinator approval/rejection; waiting-room polling.
- Roles/statuses: coordinator, subcoordinator, admin; pending, active, rejected; membership approval and active flags.
- Tenant isolation: team ID in headers/query/body plus server-side session identity and active membership verification; study limitations of client-provided headers and the authoritative server gate.
- Files: `app/api/auth/*`, `lib/db/users-auth.ts`, `lib/db/teams.ts`, `lib/db/team-members.ts`, `lib/validate-session.ts`, `lib/team-scope.ts`, `lib/auth-context.tsx`.

## 7. Major Feature Flows to Trace

### Companies and Follow-ups

- List/filter online/offline companies → team-scoped GET → normalize/serialize company data.
- Create/update/delete company → role/team authorization → company repository → UI refresh/store update.
- Add/edit/delete follow-up → embedded company array update; inspect coordinate/reference URL normalization and duplicate/assignment logic.

### Meetings

- Create online/offline meeting → validate fields/company/team → persist meeting and approval state.
- List/filter upcoming, past, company-linked, or pending-approval meetings.
- Subcoordinator create/edit/delete request → coordinator approval/rejection workflow.
- RSVP, cancel, mark visited, minutes/outcome updates → targeted PATCH/DB transitions → notifications/UI state.

### Calendar and Visit Planning

- Calendar entry by user/date/range → event schedule plus linked visit/meeting IDs.
- Map/search companies → plan visit → create/update team/user/date visit plan → assign/order companies → mark visited/skipped/rescheduled.
- Trace calendar/visit synchronization and member visit statistics.

### Dashboard, Notifications, and Support

- Dashboard aggregates stats, activity, schedule, follow-ups, analytics, and map preview from API data.
- Notifications list/read updates; profile/team events feed the UI.
- Support create/list/status update with team/user visibility rules.

### Documents and Links

- Upload/template proposal flow → DOCX processing/template replacement → download or conversion endpoint.
- DOCX-to-PDF endpoint → sanitization/conversion pipeline; inspect client/server responsibilities and size/error handling.
- URL expansion endpoint → fetch/redirect handling with SSRF protections; coordinates can be derived from reference URLs.

## 8. External Services and Browser Integrations

- Google OAuth / Google Identity Services.
- MongoDB/MongoDB Atlas.
- Leaflet with OpenStreetMap tiles and browser geolocation for maps/visit planning.
- Vercel Analytics, Google Analytics, Microsoft Clarity.
- Client document libraries: Mammoth, Docxtemplater, PizZip/JSZip, LibreOffice/ConvertAPI-related conversion path, DOMPurify.
- PWA service worker, manifest, install/settings experience.
- Identify public vs secret environment variables, redirect URI configuration, and third-party failure behavior.

## 9. Security and Reliability Review

- HttpOnly/SameSite/secure cookie settings and session revocation.
- Team isolation and role-based authorization at route/repository boundaries.
- In-memory rate limiting and its serverless scaling limitation.
- XSS sanitization, SSRF blocking, payload/file-size validation, ObjectId validation, and error handling.
- MongoDB connection reuse, pool sizing, disabled buffering, index creation, and behavior during database/network failure.

## 10. Deployment and Operations

- Vercel-style Next.js deployment: static/client pages plus serverless route handlers.
- Required configuration: `MONGODB_URI`, `MONGODB_DB_NAME`, Google credentials/redirect URI, public Google client ID, analytics IDs, optional collection names/DNS settings.
- Build/runtime concerns: server-only secrets, `NEXT_PUBLIC_*`, dynamic Leaflet/client features, environment-specific cookie security, and PWA assets.
- Development workflow: `npm run dev`, `npm run build`, `npm run start`, `npm run lint`, seed scripts; verify deployment with testing/setup guides.

## 11. Important Design Decisions and Interview Trade-offs

- Next.js monolith versus separate frontend/backend services.
- Shared MongoDB collections with `teamId` scoping versus database-per-tenant.
- Mongoose models plus native MongoDB driver instead of one persistence abstraction.
- Database-backed opaque sessions versus stateless JWT/localStorage tokens.
- Embedded subdocuments versus referenced collections and MongoDB document growth limits.
- Middleware request-shape checks plus deeper server-side authorization.
- React Context/localStorage/custom events plus Zustand rather than one global state library.
- Client-heavy map/document/PDF work versus server-side processing; dynamic loading for browser-only libraries.
- In-memory rate limiting and current connection caching versus distributed Redis/serverless-native alternatives.

## 12. Suggested Study Sequence

1. Product architecture and folder map
2. Next.js rendering and frontend component/state flow
3. API route lifecycle and shared request helpers
4. MongoDB schemas, repositories, indexes, and connection strategy
5. Google OAuth, sessions, onboarding, team authorization
6. Companies/follow-ups feature flow
7. Meetings/approval/RSVP feature flow
8. Calendar/map/visit-plan flow
9. Documents, links, notifications, and support
10. Security, external integrations, performance, deployment, and trade-offs
