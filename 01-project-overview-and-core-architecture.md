# Topic 1 — Project Overview and Core Business Architecture

## 1. The one-sentence explanation

Uny10 is a team-based sponsorship-operations workspace for festival or organization committees. It combines a sponsorship CRM, team onboarding and approval, meeting governance, calendar and visit planning, map-based company discovery, notifications, support, and proposal/document tools in one Next.js application.

The important architectural idea is not simply “a dashboard backed by MongoDB.” It is a multi-tenant application: many teams use the same deployed application and database, while every team’s operational records must be isolated by `teamId` and every request must be tied to a real authenticated user with membership in that team.

## 2. The business problem

The product exists to replace fragmented spreadsheets, chat messages, personal notes, and manually prepared documents. A sponsorship committee needs to answer questions such as:

- Which companies are being approached, by whom, and with what status?
- What contacts, calls, visits, and follow-ups already exist?
- Which team members are allowed to act for this team?
- Which meetings require coordinator approval?
- What companies are near a planned offline visit?
- What proposal or MOU material can be generated from the collected data?

The code reflects those needs through the main domain objects and routes:

| Business concern | Main UI | Main API/data code |
| --- | --- | --- |
| Team access and governance | `app/login`, `app/onboarding`, `app/waiting-room`, dashboard team switcher | `app/api/auth/*`, `lib/db/users-auth.ts`, `lib/db/teams.ts`, `lib/db/team-members.ts` |
| Sponsorship CRM | `app/companies`, `components/companies/*`, company detail page | `app/api/companies/*`, `lib/db/companies.ts`, `lib/models/company.ts` |
| Meetings | `app/meetings`, `components/meetings/*` | `app/api/meetings/route.ts`, `lib/db/meetings.ts`, `lib/models/meeting.ts` |
| Calendar | `app/calendar`, `components/calendar/calendar-content.tsx` | `app/api/calendar/route.ts`, `lib/db/calendar.ts`, `lib/models/calendar.ts` |
| Geospatial visits | `app/map`, `components/map/*` | `app/api/visit-plans/*`, `app/api/expand-url`, `lib/models/visitplan.ts` |
| Documents | `app/documents`, `components/documents/*` | `app/api/documents/convert-to-pdf`, `lib/docx-processor.ts` |
| Collaboration signals | dashboard, notifications, support center | `app/api/notifications`, `app/api/support`, `lib/notifications-helper.ts`, `lib/db/support.ts` |

## 3. High-level architecture

Uny10 uses a unified Next.js App Router architecture:

```text
Browser
  ├─ App Router pages and React client components
  ├─ AuthProvider + AuthRouteGuard
  └─ authFetch / APIClient / feature-specific fetch calls
          │ same-origin JSON requests
          ▼
Next.js server
  ├─ middleware.ts: API request-shape/status checks
  ├─ app/api/**/route.ts: HTTP handlers and business orchestration
  ├─ lib/team-scope.ts: server session + active team authorization
  ├─ lib/validate-session.ts: refresh-token lookup
  └─ lib/db/* and lib/models/*: persistence/query layer
          ▼
MongoDB
  ├─ Users, Teams, TeamMembers
  ├─ companies, meetings, calendar, VisitPlans
  └─ Notifications, Support
```

There is no separate Express service in this repository. The UI and REST-like API handlers are deployed together as one Next.js application. This is a monolith in repository and deployment shape, but it is internally layered:

1. **Presentation/routing:** `app/**/page.tsx` and `components/**`.
2. **Client integration/state:** `lib/api-client.ts`, `lib/auth-fetch.ts`, `lib/auth-context.tsx`, `lib/company-store.ts`.
3. **HTTP boundary:** `app/api/**/route.ts`.
4. **Cross-cutting security:** `middleware.ts`, `lib/validate-session.ts`, `lib/team-scope.ts`, rate limiting.
5. **Persistence:** native MongoDB repositories in `lib/db/*` and Mongoose models in `lib/models/*`.

### Why this shape fits the product

- The frontend and backend are on one origin, simplifying cookie-based sessions.
- TypeScript types can be shared between components, API clients, and repositories.
- A small committee product can deploy one application instead of coordinating separate frontend and API releases.
- Next.js provides App Router navigation, Server/Client Component boundaries, route handlers, metadata, and deployment support in one framework.

The trade-off is coupling: UI, API, and persistence evolve together, and long-running jobs, WebSockets, and heavy server-side document processing are less natural in serverless route handlers.

## 4. Repository structure: how the modules connect

### `app/`: route tree and HTTP entry points

The page routes define the user-facing product surface: `/`, `/companies`, `/meetings`, `/calendar`, `/map`, `/documents`, `/profile`, `/settings`, `/support-center`, `/login`, `/onboarding`, and `/waiting-room`.

The API routes are colocated under `app/api/`. For example, `app/api/companies/route.ts` owns collection-level company `GET` and `POST`, while `app/api/companies/[id]/route.ts` owns item-level `GET`, `PATCH`, and `DELETE`.

### `components/`: feature composition

Pages are generally thin wrappers around feature components. For example:

```tsx
// app/companies/page.tsx — route entry
<CompaniesContent />
```

The actual interaction is in `components/companies/companies-content.tsx` and its child components such as `all-companies.tsx`, `online-companies.tsx`, and `offline-companies.tsx`. This keeps route-level concerns separate from filters, dialogs, forms, tables, and responsive presentation.

The dashboard is assembled from `AppSidebar`, `TopNavbar`, `StatsCards`, `ActivityFeed`, `TodaysSchedule`, `UpcomingFollowups`, `AnalyticsSection`, and `MapPreview`.

### `lib/`: shared application and data logic

- `lib/auth-context.tsx`: client session context and auth transitions.
- `lib/auth-fetch.ts`: reads normalized local auth state and adds team/user headers.
- `lib/api-client.ts`: reusable company/follow-up API wrapper.
- `lib/team-scope.ts`: resolves a team and verifies server-side access.
- `lib/validate-session.ts`: validates the HttpOnly refresh-token cookie.
- `lib/db/*`: feature repositories and MongoDB query functions.
- `lib/models/*`: Mongoose schemas used by selected domains.
- `lib/actions/*`: older/alternate domain action/normalization helpers, especially company and user operations.
- `lib/docx-processor.ts`: document-template/XML processing.
- `lib/rate-limit.ts`: in-memory request throttling.

### `public/`, root configuration, and scripts

`public/` contains logos, the manifest, icons, and `sw.js`. `app/layout.tsx` registers the PWA and production analytics. `package.json` defines development, build, lint, and seed commands. `scripts/` contains data setup and diagnostic utilities.

## 5. Users, teams, and tenancy model

### Actors

The source supports these roles:

- **Coordinator:** creates/owns a team and approves member requests; coordinator-specific operations also govern meetings and team data.
- **Subcoordinator:** joins using a code and normally begins pending; can perform operational work after approval.
- **Admin:** represented in the user/member types, although the most visible workflows primarily distinguish coordinator and subcoordinator.

Statuses also matter independently of role: `pending`, `active`, and `rejected`. Membership has its own `approvalStatus`, `isActive`, and status fields in `TeamMembers`.

### Shared-database logical isolation

The application uses a shared MongoDB deployment and logically separates tenants with `teamId`. Company, meeting, visit-plan, and team-member records are queried with team filters. A user may have multiple `teamIds`; `activeTeamId` determines the current working context.

This is cheaper and simpler than database-per-team isolation, which is reasonable for many small, temporary committee teams. The risk is that one missing `teamId` predicate can leak data. Uny10 reduces that risk through reusable team-scoping helpers, but the design still depends on every repository and route preserving the invariant.

## 6. Complete flow A: first-time Google sign-in to an active dashboard

This is the most useful end-to-end flow for explaining the architecture.

### Step 1 — User starts at the login page

`app/login/page.tsx` renders Google sign-in through `GoogleOAuthProvider` and `GoogleLogin` from `@react-oauth/google`. The browser receives a Google credential and sends the application a JSON request to `POST /api/auth/google-callback`.

The browser request contains profile data such as `googleId`, `email`, `name`, and optionally `avatar`. The server does not treat the browser’s localStorage object as the source of truth for identity; it creates a server-side session as part of this request.

### Step 2 — The callback reconciles the user

`app/api/auth/google-callback/route.ts`:

1. Parses and validates the required OAuth fields.
2. Looks up the user by Google ID using `findUserByGoogleId`.
3. Falls back to `findUserByEmail` to avoid creating a duplicate when the email already exists.
4. Updates the Google profile when an existing email record lacks the current Google ID.
5. Creates a new user with `createGoogleUser` when no record exists.
6. Generates a UUID refresh token and stores it with a 30-day expiry using `updateUserRefreshToken`.
7. Calls `reconcileUserActiveTeam` to determine the current team context.
8. Sets `refresh_token` as an HttpOnly, SameSite=Lax cookie; it is `secure` in production.
9. Returns a state such as `onboarding_required`, `authenticated`, `waiting_for_approval`, or `rejected` plus a sanitized user object.

The user record is created with a `default-team` placeholder and pending status when no real team exists. `default-team` is a sentinel, not a valid tenant for protected business data.

### Step 3 — The frontend stores display/session metadata

The login page uses `setAuthUser`/`setStoredAuthUser` from `lib/auth-fetch.ts` and dispatches the `auth_user_updated` browser event through `lib/auth-context.tsx`. The normalized local object makes the UI responsive and allows request helpers to know the active team.

Important distinction:

- `localStorage.auth_user` is a client-side cache for rendering and request metadata.
- The HttpOnly `refresh_token` cookie is the server-side credential used by protected routes.

The local cache improves startup UX, but it must not be trusted for authorization.

### Step 4 — Auth routing chooses the next screen

`AuthProvider` is mounted by `app/layout.tsx`. `AuthRouteGuard` is also mounted there and classifies the current route as auth, onboarding, waiting, or app.

- Active user on `/login` is redirected to `/`.
- Pending user is sent to `/waiting-room`.
- Unauthenticated user trying to access app routes is sent to `/login`.
- A pending onboarding context is represented in `sessionStorage.pending_user` and routes to `/onboarding`.

`AuthProvider` first restores a cached user if one exists, then calls `GET /api/auth/session` in the background. That gives fast first paint while still allowing server state to correct stale local state.

### Step 5 — User creates or joins a team

`app/onboarding/page.tsx` offers two paths and posts to `POST /api/auth/onboarding`.

#### Create-team path

1. User enters a team name.
2. The page sends `{ userId, teamName, action: "create_team" }`.
3. The route rate-limits attempts by forwarded IP using `checkRateLimit`.
4. It loads the user and checks `MAX_TEAMS` from `lib/constants.ts` (currently `5`).
5. `createTeam` in `lib/db/teams.ts` normalizes the name, checks uniqueness, generates a random team ID and join code, creates the coordinator record, and creates the team document.
6. `onboardAsCoordinator` updates the user’s active/team/role/status fields.
7. `upsertTeamMember` creates an approved active coordinator membership.
8. The route returns the team and user; the page stores the user and navigates to `/`.

#### Join-team path

1. User enters a join code; the UI uppercases it.
2. The page sends `{ userId, joinCode, action: "join_team" }`.
3. The route finds the team with `getTeamByJoinCode`.
4. It checks the team limit and handles already-active, already-pending, and rejected/retry cases.
5. `onboardAsSubcoordinator` associates the user with the team but sets pending state for a first real team.
6. `upsertTeamMember` creates a pending inactive membership.
7. `notifyTeam` sends a notification to coordinator(s).
8. The page stores the returned user and navigates to `/waiting-room`.

### Step 6 — Waiting-room approval

`app/waiting-room/page.tsx` reads the cached user and polls `GET /api/auth/session` after 2.5 seconds and then roughly every 12 seconds while the page is visible.

- If the server reports `active`, it stores the updated user and redirects to `/`.
- If it reports `rejected`, it stores pending context and redirects to onboarding.
- If the browser is hidden or a request fails, it retries later.

This is deliberately simple polling. It avoids requiring WebSockets for a low-frequency approval event, at the cost of delayed updates and repeated requests.

## 7. Complete flow B: a protected company-list request

This flow demonstrates the normal application path after login.

### Step 1 — User opens Companies

The `/companies` page renders `CompaniesContent`. Feature components obtain the normalized cached user through `authFetch` or use the `apiClient` methods in `lib/api-client.ts` such as `getAllCompanies()`.

`APIClient.request`:

1. Builds a same-origin `/api/companies` URL.
2. Reads the cached user.
3. Adds `Content-Type`, `x-team-id`, `x-user-id`, and `x-user-status` headers.
4. Also appends `?teamId=...` as a fallback.
5. Sends credentials so the browser includes the HttpOnly cookie.

The header and query parameter are routing/context hints. They do not replace server authentication.

### Step 2 — `middleware.ts` performs an early check

The middleware matcher covers `/api/:path*`. For a team-scoped route it:

- Allows the public Google callback and onboarding routes through.
- Rejects a request whose supplied `x-user-status` is `rejected` with 403.
- Requires either `x-team-id` or `?teamId=` for non-auth API paths, returning 400 otherwise.

This is a cheap early guard. It does not query MongoDB and it does not establish the authenticated identity. In fact, client-supplied `x-user-status` and `x-user-id` are not authoritative; a malicious caller can forge them. The deeper route-level gate is essential.

### Step 3 — The route resolves and authorizes the team

`app/api/companies/route.ts`:

1. Calls `ensureIndexes()`.
2. Resolves the team using `resolveTeamIdFromRequest`, preferring the team header, then body for writes, then query parameters; the helper can fall back to `default-team`.
3. Calls `enforceTeamAccess(request, teamId)`.

`enforceTeamAccess` in `lib/team-scope.ts`:

1. Calls `getSessionUserId` in `lib/validate-session.ts`.
2. Reads the `refresh_token` cookie.
3. Uses the Mongoose `User` model to find a non-expired token and returns only the server-derived user ID.
4. Calls `hasActiveTeamAccess`.
5. Looks up the team with `getTeamById`.
6. Allows the team coordinator directly, otherwise loads the `TeamMembers` record.
7. Requires active membership plus approved/active status.
8. Returns `401`, `403`, or `404` on failure; returns `null` on success.

This “return a response or null” convention lets each route stay explicit:

```ts
const accessResponse = await enforceTeamAccess(request, teamId)
if (accessResponse) return accessResponse
```

### Step 4 — The repository applies the tenant filter

After the gate, `getAllCompanies(teamId)` in `lib/db/companies.ts` queries the companies collection using `teamId`. The returned records are serialized to JSON by `NextResponse.json`.

The most important invariant is therefore:

```text
authenticated session user
  + active membership in requested team
  + repository query filtered by same teamId
  = team-scoped response
```

### Step 5 — The frontend updates

The component receives the JSON list, updates local/component or Zustand state, and renders filters/cards/tables. If a 401 is returned, `APIClient`/`authFetch` clears local auth and redirects to `/login`; this prevents the user from remaining on a page whose server session is no longer valid.

## 8. Complete flow C: a team-scoped business mutation

A company creation request follows the same boundary but adds write validation:

```text
Form in CompaniesContent
  → apiClient.createCompany(data)
  → POST /api/companies
  → middleware requires team context
  → route parses JSON and resolves teamId from body/header/query
  → enforceTeamAccess validates cookie + membership
  → route validates required company fields
  → lib/db/companies.createCompany(data, teamId)
  → MongoDB insert with teamId
  → JSON company response
  → component closes form / refreshes list / updates store
```

The same pattern appears in follow-ups, meetings, calendar entries, and visit plans. Meetings add domain authorization and state transitions: the model carries `approvalStatus`, `requestType`, RSVP responses, cancellation details, minutes, and outcome so a subcoordinator’s proposed action can be governed by a coordinator.

## 9. Major feature architecture

### Sponsorship CRM

The company model stores the team, company type, company and sponsorship points of contact, location coordinates, reference/proposal/MOU links, amount, outreach status, special deliverables, remarks, visit history, and embedded follow-ups. Embedding follow-ups keeps a company’s interaction timeline close to the company and avoids another read for the common detail view.

`components/companies/*` provides list/filter/detail/edit behavior. `lib/db/companies.ts` applies MongoDB CRUD and team filters; `lib/actions/companies.ts` contains normalization, coordinate extraction, URL expansion, and user/company-assignment helpers.

### Meetings and governance

`Meeting` supports online/offline type, date/time/duration, link or venue, agenda, creator and role, attendees, RSVP responses, approval state, company association, minutes, cancellation, outcome, and notes. `app/api/meetings/route.ts` orchestrates list/create/update/delete and delegates persistence to `lib/db/meetings.ts`.

The design makes governance visible in data rather than only in UI: a meeting can remain pending, be approved/rejected, be cancelled, receive RSVP responses, and later be marked visited.

### Calendar and visits

Calendar entries belong to a user/date and contain note, event schedule, visit IDs, and meeting IDs. Visit plans belong to a team/user/date and contain ordered assigned companies with statuses such as assigned, visited, skipped, or rescheduled. The map UI in `components/map/*` is browser-oriented because Leaflet requires the DOM; the API persists the operational result.

### Dashboard and collaboration

The dashboard composes operational summaries rather than being a separate data subsystem. It surfaces stats, activity, today’s schedule, upcoming follow-ups, analytics, and a map preview. Notifications and support use their own APIs and persistence, while auth/team events can trigger refreshes in the client.

### Documents and external links

The documents UI includes proposal generation and quick links. `lib/docx-processor.ts` handles Word-template concerns, while `app/api/documents/convert-to-pdf/route.ts` is the conversion boundary and uses `isomorphic-dompurify` where HTML sanitization is required. `app/api/expand-url/route.ts` expands/inspects URLs and includes SSRF defenses because server-side URL fetching can otherwise reach internal resources.

## 10. Why the important technologies exist

- **Next.js App Router:** unified routing, page rendering, route handlers, layouts, and deployment; reduces cross-origin/API coordination.
- **React Client Components:** required for forms, map interaction, charts, dialogs, browser storage, and event listeners.
- **MongoDB:** flexible documents match evolving committee data such as embedded contacts, follow-ups, RSVP arrays, and meeting metadata.
- **Mongoose:** schemas/models and the `User` session validation path provide structured access for selected collections.
- **Native MongoDB driver:** direct collection operations are used by several repositories where the code wants explicit filters, updates, indexes, and collection names.
- **HttpOnly cookie sessions:** the browser sends the credential automatically while JavaScript cannot read the refresh token; the database-backed token can be revoked or expired server-side.
- **Leaflet:** browser map/visit planning without making the core application depend on a proprietary map SDK.
- **Context + localStorage + custom event:** quick session restoration plus cross-component synchronization; server revalidation corrects stale state.
- **Polling:** adequate for infrequent approval changes and simpler than real-time infrastructure.

## 11. Design decisions, trade-offs, and security review

### Logical tenancy versus stronger isolation

Shared collections reduce operational cost and make global queries/indexes easy. The trade-off is the need for disciplined `teamId` filters. A future hardening step would centralize tenant-aware repository wrappers or add automated tests that prove every team-scoped query rejects cross-team IDs.

### Two-stage authorization

`middleware.ts` is fast and general but only checks request shape and a client-provided rejected status. `enforceTeamAccess` is the real security boundary because it derives identity from the cookie and checks membership in MongoDB. Interviewers may ask why middleware does not fully authenticate: doing database work in middleware can add latency and runtime constraints, but the route handler still must authorize every protected operation.

### Client cache versus secure identity

LocalStorage enables instant rendering and supplies active-team context to helpers. It is readable and forgeable, so it must never be used alone to authorize a request. The secure cookie and server lookup provide the authoritative identity. A stronger design could use a server-rendered session bootstrap or avoid duplicating user metadata in localStorage.

### Mongoose plus native driver

The hybrid persistence layer can be justified by gradual evolution and the need for both schema-oriented models and direct collection operations. It also increases cognitive and maintenance cost: different serialization conventions, model names, collection names, and connection paths must remain consistent. A future refactor could standardize on one driver or clearly define repository ownership per collection.

### Polling versus real-time approvals

Waiting-room polling is simple, resilient, and sufficient for human approval latency. It produces periodic traffic and approval changes are not instant. WebSockets, Server-Sent Events, or a managed pub/sub service would improve immediacy but add serverless complexity.

### Important edge cases

- Missing/expired cookie → 401.
- Missing team context → middleware 400.
- Unknown team → 404 from the team access helper.
- Rejected membership → 403 and client rerouting.
- Stale localStorage user → background `/api/auth/session` correction.
- Duplicate email/Google identity → callback reconciliation instead of blindly inserting.
- Join-code guessing → onboarding rate limit.
- Default placeholder team → must not be treated as a real tenant.
- Client-forged `x-user-id` or `x-user-status` → harmless only if route handlers always use cookie-derived identity and membership checks.
- Team limit → enforced in both onboarding UI behavior and API (`MAX_TEAMS = 5`); the API is authoritative.
- Database/network failure → route-level 500 responses and client retry/redirect behavior vary by feature.

### Improvements worth proposing

- Make all API routes use a single shared authorization wrapper and typed payload schemas.
- Remove authorization reliance on client-supplied status headers entirely; use them only as hints or remove them.
- Add integration tests for cross-team read/write/delete attempts and missing tenant filters.
- Move rate limiting to Redis or a provider-backed distributed limiter for multi-instance/serverless deployment.
- Add a background job/queue for heavy document conversion and batch generation.
- Replace polling with a managed real-time channel if approval latency becomes important.
- Define one persistence abstraction and consistent ID/date serialization rules.
- Add audit logs for approvals, team membership changes, company ownership, and sensitive mutations.
- Use stricter validation and security headers/CSP before handling sensitive production data at scale.

## 12. How I should explain this topic to an interviewer

“Uny10 is a multi-tenant sponsorship-management SaaS for committee teams. It is implemented as a unified Next.js App Router application: React pages and feature components are colocated with REST-like route handlers under `app/api`. The browser maintains a cached user object for fast UI state, but authorization is server-side: a refresh-token cookie is looked up in MongoDB, then `enforceTeamAccess` verifies that the derived user is the coordinator or an approved active member of the requested team. Domain repositories then include `teamId` in their MongoDB filters. The main workflows are Google sign-in and onboarding, team approval, company/follow-up CRM, meeting governance, calendar/visit planning, maps, notifications, and documents. We chose shared logical tenancy and MongoDB for cost and flexibility, accepting the trade-off that every query must preserve tenant isolation. The main future improvements would be distributed rate limiting, stronger centralized validation/auditing, consistent persistence abstractions, and asynchronous document processing.”

For a specific request, continue with: “For example, opening Companies goes from `CompaniesContent` through `apiClient.getAllCompanies`, to `GET /api/companies`, through `middleware.ts`, then `enforceTeamAccess` validates the cookie and membership, `getAllCompanies(teamId)` queries MongoDB with the tenant filter, and the JSON response updates the component/store.”

## 13. Likely interviewer questions with answers

### Q1. What architecture does Uny10 use?

It is a unified Next.js App Router monolith with React UI, Next.js route handlers, shared TypeScript modules, and MongoDB persistence. It is deployed as one application but has clear presentation, HTTP, authorization, and repository layers.

### Q2. What does multi-tenancy mean here?

Multiple teams share the same application and database. Team-owned records carry `teamId`; requests carry the active team context; server-side session and membership checks authorize access; repositories filter by that same team ID. It is logical isolation, not database-per-team isolation.

### Q3. Why is `localStorage.auth_user` not enough for security?

Users can edit localStorage in developer tools. It is only a rendering/request-context cache. The server trusts the non-readable `refresh_token` cookie, looks it up in the `Users` collection, and derives the actual user ID before checking team membership.

### Q4. What happens if a request has a valid team header but belongs to another team?

Middleware may allow it if the shape is valid, but the route’s `enforceTeamAccess` checks the cookie-derived user against that team. If the user is not an active approved member or coordinator, the route returns a 403. The repository should also query by that team ID.

### Q5. Why use Next.js instead of separate React and Express applications?

Same-origin UI/API deployment simplifies cookies and avoids CORS coordination; types and feature code can be shared; App Router provides layouts and navigation; and one deployment is operationally simpler for the product’s scale. The trade-off is tighter coupling and less suitability for long-running or real-time workloads.

### Q6. Why MongoDB?

The domain contains naturally document-shaped and evolving data: embedded contacts/follow-ups, meeting RSVP/approval arrays, and visit-plan assignments. MongoDB supports that shape and indexed team-scoped queries with low schema migration overhead. The trade-off is weaker relational guarantees and the need to design carefully around document growth and consistent filtering.

### Q7. What is the difference between middleware and `enforceTeamAccess`?

Middleware runs broadly before API handlers and performs cheap request-shape/status checks. `enforceTeamAccess` runs inside the handler and performs the authoritative database-backed session and membership authorization. Both exist for defense in depth, but the latter is the real identity boundary.

### Q8. How does a new user become a coordinator?

Google callback creates or finds the user and sets a session. Onboarding posts `create_team`; the route creates a unique team/join code, updates the user to coordinator/active, upserts an approved coordinator membership, and returns the user/team so the frontend can enter the dashboard.

### Q9. How does a joining user become active?

The user submits a valid join code. The route creates or updates a pending `TeamMembers` record, marks the user pending/inactive for that team, notifies the coordinator, and sends the user to the waiting room. The coordinator approves the request through the approval APIs; the waiting room polls `/api/auth/session` until the server reports active.

### Q10. What is the biggest security risk in this architecture?

A missing or incorrect `teamId` filter could expose another team’s data. Client-provided headers are also forgeable. Mitigation is to derive identity from the secure cookie, centralize membership checks, apply tenant filters in every repository operation, add cross-team integration tests, and eventually enforce tenant-aware data access through a stronger abstraction.

## 14. Important things I must remember

- Uny10 is a multi-tenant sponsorship operations product, not just a generic dashboard.
- The browser cache improves UX; the HttpOnly refresh-token cookie is the server-side credential.
- `middleware.ts` is an early request guard, not the complete authentication system.
- `enforceTeamAccess` combines session validation and team membership authorization.
- `teamId` must be present in both authorization and database query scope.
- `default-team` is a placeholder and must not be treated as a real tenant.
- User role/status and team-membership role/status are related but distinct pieces of state.
- Core flow: user action → page/component → client helper → API route → middleware → session/team gate → repository/model → MongoDB → JSON → UI state.
- The codebase uses both Mongoose and the native MongoDB driver.
- The main product flows are onboarding/approval, CRM, meetings, calendar/visits, maps, documents, notifications, and support.
- Be ready to defend trade-offs: shared tenancy, hybrid persistence, polling, localStorage cache, and serverless limits.
