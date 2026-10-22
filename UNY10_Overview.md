# UNY10 Project Overview

## Project purpose

Uny10 is a team-based sponsorship and CRM workspace. It helps teams manage companies, contacts, follow-ups, meetings, calendar events, visit plans, documents, notifications, and support requests.

The application is organized around teams. Each user has an identity, a role, a team relationship, and an approval state. Business data is stored with team context so one team should not see another team's records.

---

## 1. Simple architecture

~~~text
Browser
  ↓
React pages and components
  ↓
fetch / authFetch / API helpers
  ↓
Next.js route handlers and server functions
  ↓
middleware + session validation + team authorization
  ↓
database repositories or Mongoose models
  ↓
MongoDB
  ↓
JSON response
  ↓
React state and UI
~~~

Uny10 is a full-stack Next.js application. The frontend and backend are in the same project, but they still have different responsibilities:

- The browser displays the interface and captures user actions.
- The server validates requests, checks permissions, and talks to MongoDB.
- MongoDB stores the durable application data.

---

## 2. Technology stack

### Frontend

- React 19
- Next.js App Router
- TypeScript
- Tailwind CSS
- Radix-based UI components
- React Context for authentication state
- Zustand for some company state
- Recharts for charts
- Leaflet and OpenStreetMap for maps
- React Hook Form and validation helpers in form areas

### Backend

- Next.js Route Handlers
- API endpoints under the app/api folder
- Next.js middleware
- Server-side functions under lib/actions
- Google Login
- Cookie-based database-backed sessions
- Team and role-based authorization

### Database

- MongoDB
- Mongoose schemas and models
- Native MongoDB repositories
- Cached MongoDB connections
- Indexes for common queries and uniqueness rules

### External integrations

- Google Identity Services
- MongoDB or MongoDB Atlas
- OpenStreetMap map tiles
- Google Maps/reference URLs
- Analytics services
- Document processing and PDF conversion libraries
- PWA service worker

---

## 3. How the browser and server communicate

A browser action creates an HTTP request.

A request contains:

- Method: GET, POST, PATCH, or DELETE
- URL
- Headers
- Cookies
- Query parameters
- Optional JSON body

A response contains:

- Status code
- Headers
- JSON data or an error

Example:

~~~text
POST /api/meetings?teamId=team-123
Cookie: refresh_token=...
Content-Type: application/json

{"title":"Sponsor meeting","date":"2026-10-22"}
~~~

The server processes the request and returns JSON.

Uny10 uses several client request styles:

- authFetch: adds cookies, team headers, query context, and common 401 behavior.
- apiClient: a class with domain methods such as company CRUD.
- Feature helpers such as the meetings API helper.
- Direct fetch for login, onboarding, and authentication bootstrap.

The project has more than one client helper style, but they all serve the same broad purpose: prepare a request and process a response.

---

## 4. Login and authentication

The user starts on the login page.

The login page runs in the browser and displays the Google Login control. After Google authentication succeeds, the browser sends the login information to the server endpoint:

~~~text
POST /api/auth/google-callback
~~~

The server route then:

1. Validates the submitted values.
2. Finds an existing user by Google ID or email.
3. Updates the user's profile if needed.
4. Creates a new user if necessary.
5. Checks team and approval state.
6. Creates an application session.
7. Returns a status to the browser.

The important separation is:

~~~text
Login page
  → browser-side user interaction

Google callback route
  → server-side user lookup, session creation, and cookie setting
~~~

### Application session

Uny10 creates its own random opaque session token. It stores the token and expiry on the user record in MongoDB.

The token is placed in an HttpOnly cookie called refresh_token.

These terms are different:

- Token: the random value representing the session.
- Cookie: the browser mechanism that stores and sends the token.
- Session: the server-side relationship between that token and the user.
- localStorage: a browser cache used for frontend state, not a trusted security source.

The browser also stores a normalized user object in localStorage so the UI can render quickly. The server does not trust this object for identity.

---

## 5. Onboarding and team access

A new user may need onboarding before entering the main application.

### Creating a team

The user can create a team. The server creates:

- A unique team ID
- A team name
- A join code
- Coordinator information
- Membership information

The creator becomes the coordinator.

### Joining a team

A user can provide a join code. The server:

1. Finds the team.
2. Creates a TeamMembers record.
3. Marks the membership pending.
4. Notifies the coordinator or team.
5. Sends the user to the waiting room.

### Approval

The coordinator can approve or reject a membership.

A membership has values such as:

- Role
- Status
- Approval status
- Active flag
- Team ID
- User ID

The waiting room checks the current session periodically. When the user becomes approved and active, the user can enter the dashboard.

---

## 6. What happens when the app reloads

The global authentication provider initializes when the app starts.

~~~text
App loads
  ↓
Read cached auth_user from localStorage
  ↓
Call GET /api/auth/session
  ↓
Server reads refresh_token cookie
  ↓
Find matching non-expired user in MongoDB
  ↓
Check active team and membership
  ↓
Return current user
  ↓
Update AuthProvider and localStorage
  ↓
Route guard chooses the allowed page
~~~

If the token is missing, expired, or revoked, the server returns an unauthenticated response. The frontend clears stale local data and redirects to login.

If the session is valid but the user no longer has team access, the frontend can send the user back to onboarding or an approval-related screen.

---

## 7. Protected request lifecycle

Suppose the user opens the Companies page.

The component calls:

~~~text
authFetch('/api/companies')
~~~

The helper reads the cached user, adds the current team context, includes the session cookie, and sends the request.

The server flow is:

~~~text
Browser
  ↓
GET /api/companies
  ↓
middleware
  ↓
companies route handler
  ↓
resolve team ID
  ↓
validate session cookie
  ↓
check team membership
  ↓
query MongoDB
  ↓
return JSON
  ↓
update React state
~~~

### Middleware

Middleware is an early checkpoint. In Uny10 it checks API request shape, public authentication routes, team context, and some status information.

Middleware is not the complete authorization system. Passing middleware only allows the request to continue.

### Session validation

The session helper reads refresh_token from the request cookie. It searches the User model for:

- The matching stored token
- An expiry time later than the current time

If there is no matching user, the request receives HTTP 401.

### Team authorization

The team access helper checks:

1. The authenticated user exists.
2. The requested team exists.
3. The user is the coordinator or an allowed member.
4. The membership is active and approved when required.

If the user is authenticated but not allowed, the server returns HTTP 403.

---

## 8. Database operation flow

After authorization, the route calls a database layer.

Uny10 has two database styles.

### Mongoose path

Mongoose schemas and models are stored under the lib/models folder.

Models include:

- User
- Company
- Meeting
- Calendar
- VisitPlan
- Notification

A Mongoose schema describes fields, defaults, allowed values, nested structures, and indexes.

### Native repository path

Native MongoDB repositories are stored under lib/db.

Examples include:

- Companies
- Meetings
- Calendar
- Teams
- Team members
- Support
- Authentication-related user operations

The route calls meaningful functions such as:

~~~text
getAllCompanies(teamId)
createMeeting(data, teamId)
getTeamMember(teamId, userId)
~~~

The repository hides MongoDB query syntax from the route.

### Team-scoped queries

Business records usually include teamId.

A safe query is conceptually:

~~~text
find the document where:
  document ID matches
  AND teamId matches the authorized team
~~~

This gives two protections:

- Authorization checks whether the user may access the team.
- The database filter checks whether the document belongs to that team.

---

## 9. Core modules

### Companies and follow-ups

Companies store sponsorship information, contacts, POCs, amounts, statuses, locations, documents, and follow-ups.

Follow-ups are embedded inside the company document. They are reused by:

- Company detail pages
- Calendar views
- Profile reminders
- Activity and notification displays

The current company UI often uses server-side functions from lib/actions. Dedicated API routes also exist for company and follow-up operations.

### Meetings

Meetings store dates, times, organizers, attendees, RSVP responses, approval workflow, minutes, outcome, and optional company links.

Subcoordinator requests can remain pending until a coordinator approves them.

### Calendar

Calendar entries store daily notes and scheduled events. They can reference meeting IDs and visit-plan IDs.

Calendar combines different types of work into a time-based view.

### Visit plans and maps

Visit plans store a date, assigned user, team, ordered companies, and visit state.

The map module uses company locations, coordinates, Leaflet, OpenStreetMap, and browser geolocation.

### Teams and team members

Teams store coordinator and team-level information. TeamMembers stores individual membership, role, approval, active status, and statistics.

The project uses both an embedded team members array and a separate TeamMembers collection. The embedded form is convenient for team display; the separate collection is useful for membership queries and approval workflows.

### Notifications

Notifications belong to users and can be generated for meetings, approvals, events, and other team activity.

### Documents

The document module supports proposals, MOU-related links, templates, document processing, and PDF conversion.

### Support

Support records contain issue type, title, description, status, user information, and team context.

---

## 10. Example: creating a company

A user fills the company form and clicks Create.

~~~text
User submits form
  ↓
React component builds CompanyData
  ↓
createCompany is called
  ↓
server connects to MongoDB
  ↓
company fields and coordinates are normalized
  ↓
Company document is saved
  ↓
user/team statistics are synchronized
  ↓
plain company object is returned
  ↓
React adds it to local state
  ↓
new company appears in the list
~~~

Online companies usually use a reference URL. Offline companies use a physical location.

The server may derive coordinates from a recognized map URL or address. It also adds team ID and timestamps.

The current visible company screens often call lib/actions/companies.ts directly. The separate HTTP route path is:

~~~text
POST /api/companies
  → app/api/companies/route.ts
  → enforce team access
  → lib/db/companies.ts
  → native MongoDB collection
~~~

---

## 11. Example: adding a follow-up

The user opens a company detail page and clicks Add Follow Up.

The current detail-page flow is:

~~~text
Open follow-up modal
  ↓
Select approved subcoordinator
  ↓
Choose next follow-up date
  ↓
Enter remarks
  ↓
Update local company state immediately
  ↓
Convert UI fields to database fields
  ↓
Save the complete company
  ↓
Show success or restore previous state
~~~

The UI uses fields such as:

~~~text
poc
scheduledFollowUp
remarks
~~~

Before saving, they are converted to:

~~~text
id
date
note
source
addedBy
~~~

The current detail screen saves the full company through updateCompanyDetails. It does not normally call the dedicated follow-up endpoint.

The dedicated API alternative is:

~~~text
POST /api/companies/:id/followups
PATCH /api/companies/:id/followups
DELETE /api/companies/:id/followups?followUpId=...
~~~

The native repository uses:

- $push to add a follow-up
- arrayFilters and $set to update one
- $pull to delete one

The targeted API path is better for small, independent updates. The whole-company save is simpler but can overwrite another user's concurrent changes.

---

## 12. Database connections and indexes

Uny10 has separate cached connection helpers for Mongoose and the native MongoDB driver.

The goal is:

~~~text
First request
  → create connection and cache it

Later requests in the same runtime
  → reuse cached connection
~~~

This is important in serverless deployment because opening a new connection for every request is expensive and can exhaust MongoDB connection limits.

Indexes support common queries and uniqueness rules.

Examples:

- Unique user email
- Unique Google ID when present
- Team and role lookups
- Company team/name lookup
- Meeting date/status lookup
- Calendar user/date lookup
- Unique user/team membership
- Unique visit plan for date/user/team
- Unique team name and join code

Indexes make frequent searches faster. Unique indexes also enforce rules even when two requests arrive at almost the same time.

---

## 13. Logout

Logout is both a browser and server operation.

~~~text
User clicks Logout
  ↓
AuthProvider removes auth_user from localStorage
  ↓
POST /api/auth/logout
  ↓
Server reads refresh_token
  ↓
Server clears stored token and expiry in User document
  ↓
Server expires the browser cookie
  ↓
Browser redirects to login
~~~

Removing only localStorage would not be enough. The server must revoke the database session so the old cookie cannot continue to authenticate requests.

---

## 14. Deployment

Uny10 is designed for a Vercel-style Next.js deployment.

The deployment contains:

- Frontend pages and client components
- Server-rendered or dynamically rendered pages where needed
- Serverless API route handlers
- MongoDB as an external database
- Static assets and PWA files

Important environment values include:

- MongoDB connection URI
- MongoDB database name
- Google authentication configuration
- Public Google client ID
- Analytics IDs
- Optional collection names
- DNS settings where required

Secrets must remain server-side. Only values intentionally marked as public should be exposed to the browser.

Production concerns include:

- Secure cookies
- MongoDB connection pooling
- Cached connections
- Client-only map/document libraries
- File and payload size limits
- External service failures
- Database indexes
- Error logging
- PWA assets

---

## 15. Main design decisions

### One Next.js full-stack application

This reduces deployment and communication complexity and allows shared TypeScript code.

The trade-off is tighter coupling between UI, API, server functions, and database code.

### Team ID data isolation

Most business data is scoped by team ID.

This supports multiple teams in one database, but every query must apply the correct team filter.

### Database-backed opaque sessions

The server can revoke a session immediately by clearing its stored token.

The trade-off is a database lookup when validating a session.

### Embedded and referenced data

Small related data such as follow-ups and RSVP responses is embedded for convenient reads. Other data is linked by IDs.

This is simple, but embedded arrays can grow and concurrent updates need care.

### Mongoose and native MongoDB

Mongoose gives schemas and model features. Native repositories provide direct collection control.

The trade-off is that the project must keep validation, team filters, serialization, and authorization consistent across both approaches.

---

## 16. One connected story

A user logs in with Google. The browser sends the login result to a Next.js server route. The server finds or creates the user, creates a random application session token, stores it in MongoDB, and sets it in an HttpOnly cookie. The frontend stores a convenient user-context copy locally and sends the user to onboarding, the waiting room, or the dashboard.

When the application reloads, the frontend asks the server to verify the session. The server reads the cookie, finds the non-expired user, and checks the user's active team and membership.

When the user performs an action, such as creating a company or meeting, a React component calls a request helper or server-side function. The request includes the session cookie and team context. Middleware performs early checks. The route or server function validates input, identifies the session user, checks authorization, and calls a repository or Mongoose model.

The database layer reuses a cached MongoDB connection. It performs a team-scoped query or update. Companies can contain embedded follow-ups. Meetings can contain RSVP and approval information. Calendar entries connect meetings and visit plans. Visit plans connect users, dates, teams, and companies. Notifications and support records connect back to users and teams.

The server converts database values into JSON-safe values and returns the response. The frontend parses it, updates React state, displays a toast, redirects, or restores previous state if an optimistic update failed.

When the user logs out, the frontend clears its cached state, while the server invalidates the database session and expires the cookie.

---

## 17. Natural 5-minute interview explanation

> “Uny10 is a full-stack Next.js application for managing sponsorship and team operations. The frontend uses React and TypeScript, the backend uses Next.js route handlers and server-side functions, and MongoDB stores the application data. The project uses both Mongoose models and native MongoDB repositories.
>
> The application is team-based, so a major requirement is keeping one team's data separate from another team's data.
>
> Authentication starts with Google Login. The login page sends the login result to a server route. The server finds or creates the user and then creates its own random opaque session token. That token is stored with an expiry time in MongoDB and sent to the browser in an HttpOnly cookie. The frontend also stores a user-context object in localStorage for faster rendering, but the server does not trust localStorage for security.
>
> When the app reloads or the user makes a protected request, the browser sends the cookie. The server reads it, finds the matching non-expired user, and checks the user's team membership, role, approval status, and active state. Middleware performs early request checks, but the final permission decision is made by server-side session and team authorization helpers.
>
> For a normal feature request, a component calls authFetch or another API helper. The helper adds JSON headers, includes the cookie, and attaches team context. Next.js maps the URL to a route handler under app/api. The route reads the query parameters, path parameters, or JSON body. It validates the input, checks session and team access, and then calls a database repository or Mongoose model.
>
> For example, when a user creates a company, the server normalizes contacts, status, location, and coordinates, saves the company with its team ID and timestamps, synchronizes related statistics, and returns a plain company object. Follow-ups are embedded inside the company document. Meetings contain RSVP and approval information. Calendar entries connect dates with meetings and visits. Visit plans connect users, teams, dates, and companies. Notifications and support requests are also stored with user and team context.
>
> MongoDB queries are team-scoped. Indexes support common lookups such as email, team membership, status, dates, and user/team combinations. Connection helpers cache MongoDB connections so every request does not create a new connection, which is important for serverless deployment.
>
> After a database operation, the server converts IDs and dates into JSON-safe values and returns a response. The frontend parses it and updates React state, shows a toast, redirects, or rolls back an optimistic update when necessary.
>
> On logout, the frontend removes local user state and calls the logout route. The server clears the stored session token and expiry and expires the browser cookie. This makes the old session unusable.
>
> The main strengths are the unified Next.js architecture, database-backed sessions, team-scoped data, reusable request helpers, and feature modules. The main improvement areas are standardizing the Mongoose and native MongoDB paths, applying server-side authorization consistently to every server function, and using targeted updates for operations such as follow-ups.”

