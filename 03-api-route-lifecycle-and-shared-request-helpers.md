# Topic 3 — API Route Lifecycle and Shared Request Helpers in Uny10

This note explains how a request travels through Uny10 from a browser action to a Next.js API route, authentication and team checks, database logic, and finally back to the UI.

The central mental model is:

~~~text
User action
  → client helper
  → HTTP request
  → Next.js route.ts
  → middleware
  → session and team authorization
  → database helper/model
  → JSON response
  → frontend state and UI
~~~

---

## 1. Basic concepts

### What is an API?

An API is a defined way for one part of a system to ask another part of the system for data or an operation.

In Uny10, the browser is usually the client and Next.js is the server. The browser sends an HTTP request and the server sends an HTTP response.

### What is HTTP?

HTTP is the communication protocol used by browsers and servers. It defines how requests and responses are structured.

A request contains:

- Method: the kind of operation, such as GET or POST.
- URL: the endpoint being requested.
- Headers: metadata such as content type, team context, and cookies.
- Query parameters: values after ? in the URL.
- Body: data sent with the request, normally JSON for Uny10 mutations.

A response contains:

- Status code: whether the operation succeeded or failed.
- Headers: metadata, including cookies.
- Body: returned JSON or another format.

Example:

~~~text
POST /api/meetings?teamId=team-123
Content-Type: application/json
Cookie: refresh_token=...

{"title":"Sponsor meeting","date":"2026-10-20"}
~~~

### What is an API route handler?

An API route handler is a server function that receives a request and returns a response.

In the Next.js App Router, a route handler is a file named route.ts:

~~~text
app/api/companies/route.ts
~~~

maps to:

~~~text
/api/companies
~~~

The exported function determines the HTTP method:

~~~ts
export async function GET(request: NextRequest) { ... }
export async function POST(request: NextRequest) { ... }
~~~

A page.tsx renders a UI page. A route.ts handles a data/API request.

---

## 2. How Uny10 URLs map to files

| HTTP URL | Actual file | Purpose |
|---|---|---|
| GET /api/companies | app/api/companies/route.ts | List companies |
| POST /api/companies | app/api/companies/route.ts | Create a company |
| GET /api/companies/:id | app/api/companies/[id]/route.ts | Get one company |
| PATCH /api/companies/:id | app/api/companies/[id]/route.ts | Update one company |
| DELETE /api/companies/:id | app/api/companies/[id]/route.ts | Delete one company |
| POST /api/companies/:id/followups | app/api/companies/[id]/followups/route.ts | Add a follow-up |
| GET /api/meetings?scope=upcoming | app/api/meetings/route.ts | Read upcoming meetings |
| PATCH /api/meetings | app/api/meetings/route.ts | Update, RSVP, approval, or minutes |
| GET /api/calendar?date=... | app/api/calendar/route.ts | Read calendar data |
| GET /api/auth/session | app/api/auth/session/route.ts | Verify current session |

### Dynamic route segments

The folder [id] means that the URL value is variable.

~~~text
/api/companies/company-123
      ↓
app/api/companies/[id]/route.ts
~~~

The handler receives it through params:

~~~ts
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params
}
~~~

This id is then passed to the database helper.

---

## 3. The complete route lifecycle

For a normal protected request, Uny10 follows this sequence:

1. A user action or component effect starts the request.
2. The client uses direct fetch, authFetch, apiClient, or a feature helper.
3. The helper adds headers, the team context, and cookies.
4. The browser sends an HTTP request.
5. Next.js maps the URL and method to a route.ts handler.
6. middleware.ts performs early request checks.
7. The route reads headers, query parameters, path params, or JSON body.
8. The route resolves the requested team.
9. The route validates the session cookie.
10. The route checks team membership and permission.
11. The route validates and normalizes input.
12. A database helper or model reads/writes MongoDB.
13. The route serializes database values into JSON-safe values.
14. The route returns JSON and an HTTP status.
15. The client parses the response.
16. React state, a toast, a redirect, or a refresh updates the UI.

This separation keeps components from directly knowing MongoDB query details.

---

## 4. Shared helper: lib/auth-fetch.ts

### What is a shared request helper?

A shared request helper wraps a repeated low-level operation, such as fetch, and applies common rules in one place.

Without authFetch, every component would repeat code for:

- JSON headers
- Including cookies
- Adding the current team
- Handling HTTP 401
- Redirecting to login
- Adding common request context

The main exported function is:

~~~ts
export async function authFetch(
  input: RequestInfo | URL,
  init: RequestInit = {}
) { ... }
~~~

### What authFetch does

#### 1. Reads the cached user

~~~ts
const user = getStoredAuthUser()
~~~

getStoredAuthUser reads localStorage['auth_user'], parses it, and normalizes it.

This cached value helps build the request, but it is not proof of identity. The server must trust the cookie-backed session instead.

#### 2. Creates headers

~~~ts
const headers = new Headers(init.headers)
headers.set('Content-Type', 'application/json')
~~~

A header is metadata attached to the request. Content-Type tells the server how to interpret the body.

#### 3. Adds team and user context

~~~ts
if (user?.id) {
  const teamId = user.activeTeamId || user.teamId || 'default-team'
  headers.set('x-team-id', teamId)
  headers.set('x-user-id', user.id)
  headers.set('x-user-status', user.status)
  finalUrl = appendTeamId(url, teamId)
}
~~~

Uny10 sends the team in both:

- x-team-id header
- teamId query parameter

This supports routes that read team context from different places.

Important: x-user-id and x-user-status are supplied by the browser. They can be changed by a malicious client. They are context, not trustworthy identity proof.

#### 4. Includes the cookie

~~~ts
const response = await fetch(finalUrl, {
  ...init,
  headers,
  credentials: init.credentials ?? 'include',
})
~~~

credentials: include tells the browser to include cookies when browser rules allow it. This is how the refresh_token cookie reaches the server.

#### 5. Handles HTTP 401

~~~ts
if (response.status === 401) {
  localStorage.removeItem('auth_user')
  window.location.href = '/login'
}
~~~

401 means the server considers the request unauthenticated. The helper removes stale local state and redirects from private pages.

### What authFetch does not do

It does not prove identity, decide team permission, replace backend authorization, or validate every response body. Those responsibilities belong to server-side code.

---

## 5. Shared helper: lib/api-client.ts

Uny10 also contains a class-based API client:

~~~text
lib/api-client.ts
~~~

It exposes domain methods:

~~~ts
apiClient.getAllCompanies()
apiClient.createCompany(company)
apiClient.updateCompany(id, updates)
apiClient.deleteCompany(id)
apiClient.addFollowUp(companyId, followUp)
~~~

### Why use apiClient?

A component can call a meaningful method instead of repeating URL and fetch details:

~~~ts
await apiClient.createCompany(company)
~~~

Internally, every method calls:

~~~ts
private async request<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T>
~~~

For getAllCompanies, the flow is:

~~~text
getAllCompanies()
  → request('/companies')
  → /api/companies
  → read local auth user
  → add team/user headers and teamId query
  → fetch with credentials
  → check response.ok
  → parse JSON
~~~

apiClient differs from authFetch:

| Concern | authFetch | apiClient |
|---|---|---|
| Style | Generic function | Domain-specific class |
| Return | Raw Response | Parsed JSON |
| Error parsing | Caller or feature helper | Central inside request |
| Main examples | Meetings, calendar, dashboard | Company CRUD and follow-ups |

This overlap is an important Uny10 design detail. The project has not completely standardized on one client request abstraction.

---

## 6. Direct fetch and feature helpers

Some flows intentionally use direct fetch:

- app/login/page.tsx calls /api/auth/google-callback.
- app/onboarding/page.tsx calls /api/auth/onboarding.
- lib/auth-context.tsx calls /api/auth/session and /api/auth/logout.
- lib/api/meetings.ts calls authFetch and then uses parseResponse.

Login and session bootstrap are special because the user may not yet have a valid session. AuthProvider also needs custom behavior for 401, 403, and network errors.

The project therefore contains several client-to-server styles:

~~~text
Component
  ├─ direct fetch('/api/...')
  ├─ authFetch('/api/...')
  ├─ apiClient.method(...)
  └─ server action from lib/actions/*
~~~

This is not automatically incorrect, but it increases the chance of inconsistent error handling or authorization behavior.

---

## 7. middleware.ts

### What is middleware?

Middleware is code that runs before the final route handler. It is an early checkpoint.

Uny10 configures it for API routes:

~~~ts
export const config = {
  matcher: ['/api/:path*'],
}
~~~

### Current middleware behavior

middleware.ts:

1. Allows public routes such as Google callback and onboarding.
2. Reads x-team-id and teamId.
3. Rejects a client-supplied x-user-status of rejected with 403.
4. Requires team context for most non-auth API routes.
5. Calls NextResponse.next() when the request may continue.

NextResponse.next() only means “continue to the next processing stage.” It does not prove the user is authorized.

The comments describe pending and rejected users as denied, but the runtime condition explicitly rejects only rejected. Pending users can pass this middleware stage; later route logic still matters.

### Middleware versus authoritative authorization

~~~text
middleware.ts       → early request-shape/filter check
enforceTeamAccess   → server-side identity and team authorization
database query      → team-scoped data access
~~~

---

## 8. Shared server-side gates in lib/team-scope.ts

### resolveTeamIdFromRequest

This function resolves team ID from:

1. x-team-id header
2. teamId/teamID/teamid/team in the body
3. teamId query parameter
4. default-team fallback

The route uses this to agree on one team value before authorization and database work.

### getSessionUserId in lib/validate-session.ts

This function reads the refresh_token cookie and searches MongoDB:

~~~ts
const refreshToken = request.cookies.get('refresh_token')?.value
if (!refreshToken) return null

const user = await User.findOne({
  refreshToken,
  refreshTokenExpires: { $gt: new Date() },
}).select('_id').lean()
~~~

This is the authoritative identity source. The server does not need to trust x-user-id.

### hasActiveTeamAccess

This function:

1. Rejects missing or unsuitable team IDs.
2. Loads the team.
3. Allows the coordinator when the coordinator ID matches.
4. Otherwise loads TeamMembers.
5. Requires an active and approved/active membership.

### enforceTeamAccess

This combines authentication and team authorization:

~~~ts
const sessionUserId = await getSessionUserId(request)
if (!sessionUserId) {
  return NextResponse.json(
    { error: 'Authentication required' },
    { status: 401 },
  )
}

const access = await hasActiveTeamAccess(resolvedTeamId, sessionUserId)
~~~

It returns null on success or a NextResponse on failure.

The common route pattern is:

~~~ts
const accessResponse = await enforceTeamAccess(request, teamId)
if (accessResponse) return accessResponse
~~~

If the result is null, execution continues. If it is a response, the route returns immediately.

---

## 9. Complete flow: loading companies with authFetch

### Flow diagram

~~~text
Companies/dashboard component
  → authFetch('/api/companies')
  → localStorage user creates team context
  → cookies and headers are sent
  → middleware.ts
  → app/api/companies/route.ts GET
  → resolveTeamIdFromRequest
  → enforceTeamAccess
  → getAllCompanies(teamId)
  → MongoDB companies collection
  → NextResponse.json(companies)
  → response.json()
  → setCompanies(companies)
  → UI renders companies
~~~

### Step 1: component starts the request

A dashboard or calendar component calls:

~~~ts
authFetch('/api/companies')
~~~

The input is the API URL. The component is asking for companies for the selected team.

### Step 2: authFetch prepares the request

Suppose localStorage contains:

~~~json
{
  "id": "user-1",
  "activeTeamId": "team-123",
  "status": "active"
}
~~~

The request is conceptually:

~~~text
GET /api/companies?teamId=team-123
x-team-id: team-123
x-user-id: user-1
x-user-status: active
Cookie: refresh_token=...
~~~

### Step 3: Next.js chooses the route

GET /api/companies maps to app/api/companies/route.ts and calls its GET function.

### Step 4: middleware checks the request

The middleware sees an API request with team context and a status that is not rejected. It calls NextResponse.next().

### Step 5: route resolves the team

~~~ts
const teamId = resolveTeamIdFromRequest(request)
~~~

The x-team-id header is preferred, so teamId becomes team-123.

### Step 6: route authenticates and authorizes

~~~ts
const accessResponse = await enforceTeamAccess(request, teamId)
if (accessResponse) return accessResponse
~~~

Internally:

1. The cookie is read.
2. MongoDB finds the non-expired matching session.
3. The user ID is obtained from that database record.
4. Team team-123 is loaded.
5. Coordinator or active approved TeamMembers access is checked.
6. Failure returns 401, 403, or 404 immediately.

### Step 7: database helper runs

~~~ts
const companies = await getAllCompanies(teamId)
~~~

This function is in lib/db/companies.ts. It reuses the MongoDB connection, chooses the collection, and filters by team.

For default-team, it includes older records without a teamId:

~~~ts
{ $or: [{ teamId }, { teamId: { $exists: false } }] }
~~~

### Step 8: server returns JSON

~~~ts
return NextResponse.json(companies)
~~~

### Step 9: frontend updates React state

The caller parses the body and updates state:

~~~ts
const response = await authFetch('/api/companies')
const companies = await response.json()
setCompanies(companies)
~~~

React re-renders the list.

---

## 10. Complete flow: creating a company and the two real paths

Uny10 has two company data paths. This must be understood from imports.

### Path A: apiClient route path

lib/api-client.ts contains:

~~~ts
async createCompany(company) {
  return this.request<Company>('/companies', {
    method: 'POST',
    body: JSON.stringify(company),
  })
}
~~~

This becomes:

~~~text
POST /api/companies
  → app/api/companies/route.ts
  → createCompany() in lib/db/companies.ts
  → native MongoDB collection
~~~

The route validates required fields, enforces team access, and passes teamId to the repository.

### Path B: current CompaniesContent path

components/companies/companies-content.tsx imports:

~~~ts
import {
  createCompany,
  getAllCompanies
} from '@/lib/actions/companies'
~~~

That means its createCompany call goes to lib/actions/companies.ts, which contains use server and directly uses the Mongoose Company model.

It does not follow the same URL-based path as apiClient.

~~~text
apiClient.createCompany()
  → HTTP route → lib/db/companies.ts → native MongoDB

lib/actions/companies.ts createCompany()
  → server-side function → Mongoose Company model
~~~

Both functions are named createCompany, but they are different functions.

This is why tracing the actual import is more reliable than assuming every feature uses the API route.

---

## 11. Complete flow: creating a meeting

Meetings use a clear shared-helper and route-handler pattern.

### Client helper

lib/api/meetings.ts contains:

~~~ts
export async function createMeeting(meeting: any) {
  const response = await authFetch('/api/meetings', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(meeting),
  })
  return parseResponse<any>(response)
}
~~~

parseResponse checks response.ok, parses a useful error message, or returns response.json().

### Route handler

app/api/meetings/route.ts does:

~~~ts
const body = await request.json()
const teamId = resolveTeamIdFromRequest(request, body)
const accessResponse = await enforceTeamAccess(request, teamId)
if (accessResponse) return accessResponse
const meeting = await createMeeting(body, teamId)
~~~

### Exact execution

1. A meeting component calls createMeeting(meeting).
2. JSON.stringify converts the object into a request body.
3. authFetch adds the cookie and team context.
4. The browser sends POST /api/meetings.
5. middleware performs early checks.
6. Next.js calls POST in app/api/meetings/route.ts.
7. request.json() creates a JavaScript body object.
8. The route resolves teamId.
9. enforceTeamAccess validates the cookie and membership.
10. lib/db/meetings.ts writes the meeting.
11. The route checks approvalStatus.
12. notifyTeam may create notifications.
13. serializeMeeting converts MongoDB _id to a string.
14. The route returns HTTP 201 and JSON.
15. parseResponse parses the response.
16. The component updates state or refreshes its data.

HTTP 201 means a new resource was created.

---

## 12. Meeting queries and action-based PATCH

GET /api/meetings?scope=upcoming reads scope and chooses getUpcomingMeetings(teamId). Other supported scopes include past and pending.

For mutations, the same PATCH route uses an action field:

~~~json
{
  "action": "rsvp",
  "meetingId": "meeting-1",
  "userId": "user-1",
  "status": "yes"
}
~~~

The route does:

~~~ts
const { action, ...payload } = body

if (action === 'rsvp') {
  const meeting = await updateRSVPResponse(...)
  return NextResponse.json(serializeMeeting(meeting))
}
~~~

Other actions are update, approval, and minutes.

Why use an action field? Related meeting operations stay in one route.

Trade-off: the route becomes a large dispatcher. Separate endpoints or strongly typed commands may become easier to validate as the feature grows.

---

## 13. Calendar query selection

app/api/calendar/route.ts reads:

~~~ts
const id = searchParams.get('id')
const userId = searchParams.get('userId')
const date = searchParams.get('date')
const startDate = searchParams.get('startDate')
const endDate = searchParams.get('endDate')
~~~

It selects a database operation:

~~~text
id                           → getCalendarEntryById
userId + date                → getCalendarEntriesByUserAndDate
date only                    → getCalendarEntriesByDate
userId + startDate + endDate → getCalendarEntriesByUserAndDateRange
userId only                  → getCalendarEntriesByUser
otherwise                    → HTTP 400
~~~

The route is a translation layer:

~~~text
Browser query parameters
  → business meaning
  → selected database function
~~~

The browser does not send MongoDB syntax. The server chooses the database operation.

---

## 14. Reading request data

### Headers

~~~ts
request.headers.get('x-team-id')
~~~

Headers hold metadata. Uny10 uses them for team and client context.

### Query parameters

~~~text
/api/calendar?date=2026-10-20&userId=user-1
~~~

Read them with:

~~~ts
const { searchParams } = new URL(request.url)
const date = searchParams.get('date')
~~~

or request.nextUrl.searchParams.

Use query parameters for filtering, scopes, and IDs in read/delete requests.

### JSON body

~~~ts
const body = await request.json()
~~~

Use the body for structured create/update data. Validate it before database writes.

### Dynamic path params

For /api/companies/[id], the path value is read from params.

A route must read values from the correct location and return HTTP 400 for missing or malformed client input.

---

## 15. Validation, normalization, and serialization

### Validation

Validation checks whether input is acceptable.

Example in app/api/companies/route.ts:

~~~ts
if (!body.name || !body.type || !body.companyPOC || !body.sponsorshipPOC) {
  return NextResponse.json(
    { error: 'Missing required fields' },
    { status: 400 }
  )
}
~~~

HTTP 400 means the client sent invalid or incomplete input.

### Normalization

Normalization converts different input forms into one predictable form.

Uny10 examples:

- resolveTeamIdFromRequest accepts multiple team field spellings.
- getStoredAuthUser normalizes localStorage data.
- Company helpers convert IDs to strings.
- Calendar routes convert date text to Date values.

### Serialization

Serialization converts server-side values into JSON-safe values.

MongoDB IDs and Date values may require conversion:

~~~ts
function serializeMeeting(meeting: any) {
  return {
    ...meeting,
    _id: meeting._id?.toString?.() ?? meeting._id,
  }
}
~~~

The calendar route converts Date values to ISO strings. This gives the frontend predictable values.

---

## 16. Database connection strategy

Uny10 has two connection helpers:

~~~text
lib/mongodb.ts     → Mongoose connection
lib/db/mongodb.ts  → native MongoClient connection
~~~

Connection caching matters because opening a new MongoDB connection for every request is slow and can exceed connection limits in serverless deployments.

The conceptual behavior is:

~~~text
First request  → create and cache connection
Later request  → reuse cached connection
~~~

Mongoose path:

~~~ts
await connectToDatabase()
const companies = await Company.find(query).lean().exec()
~~~

Native driver path:

~~~ts
const { db } = await connectToDatabase()
const collection = db.collection('companies')
const companies = await collection.find(filter).toArray()
~~~

Both reach MongoDB, but they are different persistence abstractions.

---

## 17. Error handling

Most route handlers follow this shape:

~~~ts
try {
  // parse input, authorize, query database, return success
} catch (error) {
  console.error('...', error)
  return NextResponse.json(
    { error: 'Human-readable failure message' },
    { status: 500 }
  )
}
~~~

Common status codes:

| Status | Meaning |
|---:|---|
| 200 | Successful read or update |
| 201 | Resource created |
| 400 | Missing or invalid input |
| 401 | Missing, invalid, or expired authentication |
| 403 | Authenticated but not allowed or access revoked |
| 404 | Resource or team not found |
| 409 | Business conflict |
| 429 | Too many requests |
| 500 | Unexpected server/database failure |

Typical failure flow:

~~~text
Database/helper throws
  → route catch block
  → server logs detailed error
  → route returns safe JSON error
  → client checks response.ok/status
  → UI shows error or redirects
~~~

Raw database errors should not be sent to the browser because they may reveal implementation details. Uny10 generally logs details and returns a simpler client message, although error formats are not perfectly uniform.

---

## 18. Client-side response handling

### Raw Response pattern

Used with authFetch:

~~~ts
const response = await authFetch('/api/companies')

if (!response.ok) {
  const data = await response.json()
  throw new Error(data.error || 'Request failed')
}

const companies = await response.json()
~~~

authFetch returns the raw Response, so the caller parses it.

### Shared parser pattern

lib/api/meetings.ts contains:

~~~ts
async function parseResponse<T>(response: Response): Promise<T> {
  if (!response.ok) {
    const data = await response.json().catch(() => ({}))
    throw new Error(data?.error || 'Request failed')
  }
  return response.json()
}
~~~

The feature helper then returns parseResponse(response).

After success, a component may set state, add an item to an array, show a toast, close a dialog, trigger a refresh, or redirect. A response does not automatically update React state.

---

## 19. Dynamic company update flow

Suppose the user edits company company-123.

~~~text
Company form
  → apiClient.updateCompany('company-123', updates)
  → PATCH /api/companies/company-123?teamId=team-123
  → middleware.ts
  → app/api/companies/[id]/route.ts PATCH
  → await params gives id
  → request.json gives updates
  → resolveTeamIdFromRequest
  → enforceTeamAccess
  → lib/db/companies.ts updateCompany
  → MongoDB findOneAndUpdate with ID and team filter
  → updated company or 404
  → JSON response
  → apiClient parses it
  → UI refreshes/displays result
~~~

The repository combines the company-ID condition with a team filter:

~~~ts
{ ...query, ...buildTeamFilter(teamId) }
~~~

This matters because knowing another company's ID should not reveal or modify a record outside the user's team.

Authorization and query scoping defend different boundaries:

- enforceTeamAccess: can this user access the team?
- repository filter: does this record belong to that team?

---

## 20. Adding a follow-up

The endpoint is:

~~~text
app/api/companies/[id]/followups/route.ts
~~~

Add flow:

~~~text
Company detail UI
  → POST /api/companies/:id/followups
  → read company id from params
  → read follow-up from JSON body
  → resolve team
  → enforce session and team access
  → validate id, date, and note
  → addFollowUp(companyId, followUp, teamId)
  → MongoDB $push into embedded followUps array
  → return updated company with HTTP 201
~~~

Embedding follow-ups is convenient because the company and its follow-ups are returned together. The trade-off is that the company document can grow. A separate FollowUps collection may scale better if follow-ups become numerous or require independent querying.

---

## 21. Responsibilities by layer

Shared helpers handle repeated infrastructure:

- authFetch: client request preparation and common 401 behavior.
- apiClient: company API methods and JSON/error parsing.
- resolveTeamIdFromRequest: common team resolution.
- getSessionUserId: server-side identity.
- enforceTeamAccess: authentication and team authorization.
- connectToDatabase: connection reuse.

Route handlers handle endpoint-specific behavior:

- Required company fields.
- Meeting scope and action.
- Calendar query combinations.
- Notifications after an approved meeting.
- Choosing the correct database function.

Database helpers/models handle persistence:

- Collection/model selection.
- MongoDB and Mongoose queries.
- ObjectId conversion.
- Team filters.
- MongoDB updates such as $push, $pull, and $set.

The intended separation is:

~~~text
Component/helper → request shape
Route handler    → HTTP contract and orchestration
Shared gate      → session/team access
Repository/model → database operations
~~~

---

## 22. Important design decisions and trade-offs

### Next.js route handlers instead of a separate backend

The frontend and API are in one Next.js application. This reduces deployment and communication complexity and allows shared TypeScript code.

Trade-off: UI, API, server actions, and database code can become closely coupled. A separate backend would create a stricter service boundary but add operational complexity.

### Team context in headers and query/body

The team ID is easy for routes to receive, but it is client-controlled context. The server must validate membership instead of accepting it as proof.

### Database-backed opaque sessions

The server can revoke a session by removing the stored token. The trade-off is a database lookup for session validation.

### Multiple data access styles

lib/db uses native MongoDB for several routes, while lib/actions and models use Mongoose. This may reflect feature history or migration stages, but it can cause differences in validation, serialization, and team filtering.

### Action-based PATCH routes

A single route can handle related meeting/calendar operations through an action field. It reduces files but can become a large dispatcher with weaker input typing.

---

## 23. Security considerations

The browser is not trusted. It can change localStorage, headers, query parameters, body values, and even call APIs without using the UI.

The secure order is:

~~~text
Cookie/session identifies user
  → server loads user
  → server checks requested team
  → database query is team-scoped
  → operation is performed
~~~

Current safeguards include:

- refresh_token cookie session.
- getSessionUserId validation against MongoDB and expiry.
- enforceTeamAccess membership checks.
- Team filters in company repository queries.
- Early middleware checks.
- Manual input validation in many routes.
- SSRF protections in expand-url.

Important limitations:

- x-user-id and x-user-status are client-supplied.
- Some routes use different authentication patterns.
- Shared helpers are not used everywhere.
- User IDs supplied in body/query should be compared with the session user for user-specific actions.
- Validation is often manual rather than schema-based.

---

## 24. Edge cases

### No cached user

authFetch sends no user/team headers. A protected request may receive 400 from middleware or 401 from session validation.

### Stale localStorage

localStorage may say the user is logged in while the cookie is expired or revoked. The server returns 401, and the helper/AuthProvider clears local state and redirects.

### Network failure

There may be no HTTP response at all. fetch rejects, so the caller needs try/catch. AuthProvider sometimes keeps cached state during temporary network failures.

### Invalid JSON

request.json() can throw. Many route catch blocks turn this into 500, although malformed client input would ideally be reported as 400.

### Missing resource

A null database result becomes a 404 response.

### Invalid scope/action

Meetings and calendar routes return 400 when the requested scope, action, or query combination is not supported.

### Database failure

The server logs the failure and usually returns 500. The UI should present a retryable error.

### Duplicate mutation

Double-clicks and retries may create duplicate records unless idempotency keys or unique business constraints are used.

---

## 25. How to debug an API request

1. Find the frontend call for the URL.
2. Identify direct fetch, authFetch, apiClient, feature helper, or server action.
3. Map the URL to app/api/**/route.ts.
4. Match the HTTP method to GET, POST, PATCH, or DELETE.
5. Check where input comes from: headers, query, body, or params.
6. Check middleware.ts.
7. Check team resolution and enforceTeamAccess.
8. Find the database/model function called.
9. Inspect its team filter and query.
10. Inspect response status and body.
11. Return to the client and trace response parsing and React state.

Example:

~~~text
Caller: authFetch('/api/companies')
Route: app/api/companies/route.ts GET
Team: resolveTeamIdFromRequest(request)
Auth: enforceTeamAccess(request, teamId)
DB: getAllCompanies(teamId)
Response: NextResponse.json(companies)
UI: response.json() → setCompanies(...)
~~~

---

## 26. How I should explain this topic to an interviewer

### Simple answer

> “Uny10 uses Next.js route handlers under app/api. A frontend component usually calls authFetch, the class-based apiClient, or a feature helper such as lib/api/meetings.ts. These helpers add JSON headers, include the session cookie, and attach the current team context. Next.js maps the URL and HTTP method to the exported function in route.ts. The route reads query parameters, body data, or dynamic path parameters, resolves the team, and calls enforceTeamAccess. That helper validates the session cookie on the server and checks team membership. After authorization, the route calls a database helper or model, serializes the result, and returns JSON with an appropriate status. The client parses the response and updates React state or shows an error. Middleware runs before the route for early checks, but it is not the only authorization layer.”

### Humanized answer

> “I think of each API call as a pipeline. The UI starts the request, a shared helper adds repeated details such as cookies and team information, and the URL selects a Next.js route handler. The route translates the HTTP request into a business operation. Before touching the database, it checks who the user is and whether they can access that team. Then it calls the database layer, converts the result into JSON, and sends it back. The component receives the result and changes the UI.”

### If asked about authFetch and apiClient

> “authFetch is a generic wrapper that returns the raw Response, so the caller or a feature helper parses it. apiClient is a class with domain methods such as createCompany and centralized JSON/error handling. They overlap, so I would consider consolidating them or defining a clear rule for when each is used.”

### If asked about the security boundary

> “The browser can provide team headers and query parameters, but those are only context. The server must not trust them as proof. Identity comes from the validated session cookie through getSessionUserId, enforceTeamAccess checks membership, and the database query also includes the team filter.”

---

## 27. Likely interviewer questions and answers

### Q1. What is a route handler?

A server function that receives an HTTP request and returns an HTTP response. In Next.js App Router it is an exported GET, POST, PATCH, or DELETE function inside route.ts.

### Q2. How does /api/companies reach its file?

Next.js maps the folder structure under app to the URL. app/api/companies/route.ts represents /api/companies, and the exported method matching the request is called.

### Q3. What are GET, POST, PATCH, and DELETE?

GET reads, POST creates or starts an operation, PATCH partially updates, and DELETE removes data.

### Q4. Why does Uny10 use authFetch?

It centralizes JSON headers, cookies, team headers, team query parameters, and 401 redirect behavior.

### Q5. Does authFetch authenticate the user?

No. It prepares the request and includes the cookie. The server authenticates by validating the cookie with getSessionUserId.

### Q6. What does credentials include do?

It allows cookies to be sent with the request when browser cookie rules permit it. Uny10 needs this for the cookie-backed session.

### Q7. Why send teamId in both a header and query parameter?

The project supports several route conventions, and resolveTeamIdFromRequest reads multiple locations. The server still must validate access and cannot trust the team ID alone.

### Q8. What happens before a route handler?

middleware.ts runs for API paths. It handles public routes, checks some status/team context, and either returns an error or calls NextResponse.next().

### Q9. Is middleware the complete authorization system?

No. It is an early layer. enforceTeamAccess performs authoritative session and team membership checks.

### Q10. What happens when enforceTeamAccess fails?

The route immediately returns its NextResponse. Missing session usually produces 401, revoked access produces 403, and an unknown team can produce 404.

### Q11. Why team-scope the database query after authorization?

Authorization proves the user can access a team. The query filter ensures the selected document belongs to that team. Both protect against different mistakes.

### Q12. What is serialization?

Converting server values into JSON-safe values. Uny10 converts MongoDB IDs to strings and dates to ISO strings.

### Q13. Why use response.ok?

It tells the client whether the HTTP status indicates success. The client can parse success JSON or handle an error body.

### Q14. Why return 201 for creates?

201 communicates that a new resource was created.

### Q15. Does every operation use an API route?

No. Some company UI code imports server-action-style functions from lib/actions/companies.ts. Other code uses authFetch, apiClient, or direct fetch. The actual import determines the path.

### Q16. What is a server action?

A server-side function intended to be invoked from a Next.js application. In Uny10, lib/actions files use use server and directly access Mongoose. They are different from URL-based route handlers.

### Q17. What is the risk of multiple data access styles?

Mongoose and native MongoDB paths can differ in validation, serialization, team filtering, and transaction behavior. Standardizing the repository boundary would improve consistency.

### Q18. How would you improve validation?

Use consistent runtime schemas for request bodies and query parameters. This would give clearer errors and reduce repeated manual checks.

### Q19. How would you improve error handling?

Use a standard response shape with stable error codes, safe messages, and optional field errors. A shared client parser could handle common status codes consistently.

### Q20. What if the user double-submits a form?

The route may create duplicates. Disable the UI button, but also protect the server with idempotency keys or unique business constraints.

### Q21. How would you test a route?

Test success, missing fields, malformed JSON, expired sessions, unauthorized teams, unknown IDs, database failures, duplicate requests, and exact status codes. Test both isolated routes and browser flows.

### Q22. What is the most important security rule?

Never trust identity or permission claims from the browser. Derive identity from the server session, authorize on the backend, and team-scope database operations.

---

## 28. Important things to remember

1. A URL maps to a route.ts file; the exported HTTP method handles it.
2. authFetch is a generic client wrapper for cookies and team context.
3. apiClient is a class-based API abstraction for company and follow-up methods.
4. lib/api/meetings.ts wraps authFetch and adds response parsing.
5. Login, onboarding, and AuthProvider use direct fetch for special control.
6. Some company UI code uses lib/actions/companies.ts instead of the API route.
7. middleware.ts is an early filter, not the complete authorization layer.
8. resolveTeamIdFromRequest determines the team context.
9. getSessionUserId derives identity from the cookie-backed session.
10. enforceTeamAccess combines authentication and team authorization.
11. Route handlers translate HTTP input into business operations.
12. Database helpers hide MongoDB/Mongoose details.
13. Queries should include team filters for tenant isolation.
14. request.json reads a JSON body; searchParams reads URL query values; params reads dynamic path values.
15. NextResponse.json sends JSON with an optional status.
16. response.ok is a status check; it does not parse the body.
17. 201 means created, 400 bad input, 401 unauthenticated, 403 forbidden, 404 not found, and 500 server failure.
18. Serialization makes MongoDB IDs and dates safe for the frontend.
19. The browser can fake headers and IDs; the backend makes the security decision.
20. The interview summary is: client helper → route → middleware → session/team gate → database → JSON → UI.

---

## 29. What I learned

- How Next.js maps app/api/**/route.ts files to HTTP endpoints.
- How a request moves from component to helper, route, authorization gate, database, and React state.
- Why authFetch and apiClient exist and how they differ.
- How middleware differs from route-level authorization.
- How Uny10 resolves team context and validates sessions.
- How HTTP methods, query parameters, bodies, and dynamic path params are used.
- Why validation, serialization, status codes, error handling, and connection caching matter.
- Why actual imports must be traced because the project contains API, server-action, Mongoose, and native MongoDB paths.

## 30. Start-to-finish diagram

~~~text
User interaction or component effect
  → direct fetch / authFetch / apiClient / feature helper
  → HTTP method + URL + headers + cookies + query/body
  → Next.js route matching
  → middleware.ts
  → route.ts reads input
  → team resolution
  → session validation
  → team authorization
  → validation and business logic
  → Mongoose or native MongoDB
  → team-scoped query/update
  → serialized JSON response
  → client parser
  → React state, toast, redirect, or refresh
~~~

## 31. Final interview summary

> “Uny10's API layer is implemented with Next.js App Router route handlers. The frontend calls routes through shared helpers such as authFetch, apiClient, and feature-specific API functions. These helpers add common headers, include the session cookie, and attach team context. The route handler reads request input, resolves the team, validates the server-side session, and calls enforceTeamAccess. If access is allowed, it calls a database helper or model, applies team-scoped queries, serializes MongoDB values, and returns JSON with an appropriate status code. The frontend parses the response and updates its state. One design consideration is that the project currently has multiple request and database-access patterns, so standardizing them would improve consistency and maintainability.”

