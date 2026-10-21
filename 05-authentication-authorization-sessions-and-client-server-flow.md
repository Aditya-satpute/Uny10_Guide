# Authentication, Authorization, Sessions, Cookies, and Client–Server Communication in Uny10

This note studies the authentication and authorization part of the actual Uny10 code. It starts from basic terminology, then traces the complete execution flows: Google sign-in, session creation, app startup, protected API requests, team onboarding, approval, waiting-room polling, team switching, and logout.

The central question is:

> How does Uny10 know who the browser belongs to, which team the user is acting for, and whether that user may perform a specific operation?

The short answer is:

~~~text
Google proves an external identity
  → Uny10 creates/finds a User
  → Uny10 creates a random database-backed session token
  → browser stores it as an HttpOnly cookie
  → every protected request sends the cookie automatically
  → server looks up the token and derives the real user ID
  → server checks active membership in the requested team
  → route performs a team-filtered operation
~~~

## 1. Authentication versus authorization

### Authentication

**Meaning:** Authentication answers: “Who are you?”

In Uny10, Google sign-in is used as the initial identity provider. The application receives Google profile/credential information, finds or creates a local Users record, and establishes an Uny10 session.

### Authorization

**Meaning:** Authorization answers: “What are you allowed to do?”

After the server knows the user ID, Uny10 checks team membership, approval state, active state, and sometimes role. For a protected team-scoped API request, the relevant code is:

- lib/validate-session.ts — identifies the user from the session cookie.
- lib/team-scope.ts — checks access to a requested team.
- app/api/*/route.ts — may apply role-specific business rules.
- lib/db/* — should query using the authorized team ID.

A user can be authenticated but not authorized:

~~~text
Google identity valid
  + Uny10 session valid
  + user is pending/rejected/not a member of team X
  = authenticated, but not authorized for team X
~~~

This distinction is essential in interviews. A login check alone does not provide tenant isolation.

## 2. Client–server communication basics

### What is an HTTP request?

An HTTP request is a message from a browser to a server. It usually contains a method such as GET or POST, a URL, headers, and an optional body, commonly JSON.

Example from app/login/page.tsx:

~~~tsx
fetch('/api/auth/google-callback', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  credentials: 'include',
  body: JSON.stringify({
    googleId: decoded.sub,
    email: decoded.email,
    name: decoded.name,
    avatar: decoded.picture,
  }),
})
~~~

- POST says the browser is sending data to cause a server-side operation.
- The URL selects the Next.js route handler.
- Content-Type tells the server the body is JSON.
- credentials: include tells the browser to include cookies.
- JSON.stringify converts a JavaScript object into a string for transport.

### What is an HTTP response?

The server returns a status code, headers, and a body, often JSON.

For example:

~~~json
{
  "status": "authenticated",
  "user": {
    "id": "...",
    "teamId": "team_...",
    "role": "coordinator",
    "status": "active"
  }
}
~~~

The browser reads it using await response.json().

### Same-origin communication

Uny10’s UI and API use one application origin, for example:

~~~text
https://unyten.example.com/
https://unyten.example.com/api/auth/session
~~~

This helps cookie-based sessions because the browser naturally sends the cookie to the same origin. It also avoids much of the CORS configuration required by separate frontend and backend domains.

### Headers versus body versus cookies

Uny10 uses all three:

- Body: login payload, onboarding action, join code, meeting data.
- Headers: x-team-id, x-user-id, and x-user-status are attached by authFetch.
- Cookie: refresh_token is the server-side session credential.

The security rule is:

~~~text
Client headers and body are claims from the client.
The HttpOnly cookie plus server database lookup is the trusted identity source.
~~~

The code comments in lib/validate-session.ts explicitly say not to trust client-supplied x-user-id.

## 3. Tokens: several different things are involved

The word “token” is overloaded. Uny10 involves more than one kind.

### Google credential / Google ID-token-like JWT

When the user clicks Google sign-in, GoogleLogin from @react-oauth/google returns a credential string. It has the familiar three-part JWT shape:

~~~text
header.payload.signature
~~~

A JWT is a JSON Web Token: a compact string containing claims such as subject, email, and name. Its payload is base64url-encoded. Base64url encoding is not encryption.

In app/login/page.tsx, Uny10 calls decodeJwtPayload(credential). It takes the middle section, converts URL-safe characters, restores padding, calls atob, and parses JSON. It then sends decoded.sub, decoded.email, decoded.name, and decoded.picture to the application API.

**Important security distinction:** decoding a JWT is not the same as verifying its signature. The current code extracts claims in the browser and the server callback accepts the posted Google ID, email, name, and avatar. A production-hardening improvement would send the original credential to the server and verify its signature, issuer, audience, expiry, and nonce using Google’s official verification mechanism.

### Uny10 refresh_token cookie

The callback route creates:

~~~ts
const refreshToken = crypto.randomUUID()
~~~

This is an opaque random identifier. It does not contain readable user claims and is not a JWT.

The project calls it refresh_token, but conceptually it is a database-backed session token:

~~~text
cookie value = random UUID
Users.refreshToken = same UUID
Users.refreshTokenExpires = future date
~~~

It is not the same as an OAuth refresh token issued by Google. The name is application-specific and can be confusing in an interview.

### Access token

An access token is normally a short-lived credential presented to an API to authorize access. Uny10 does not use a separate Uny10 access-token/JWT layer. It uses the database-backed cookie session directly.

### CSRF token

A CSRF token is a separate unpredictable value used to prove that a state-changing request came from the legitimate application rather than an attacker’s cross-site form.

The current code does not show a separate CSRF token. It relies mainly on same-origin behavior, SameSite=Lax cookies, and server-side validation. That is a security consideration, not a complete CSRF-token implementation.

## 4. Cookies and their security attributes

The Google callback sets:

~~~ts
cookieStore.set("refresh_token", refreshToken, {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax",
  path: "/",
  maxAge: 30 * 24 * 60 * 60,
})
~~~

### HttpOnly

JavaScript cannot read the cookie through document.cookie.

This helps protect the token from direct theft by an XSS script. In Uny10, server code reads it through cookies() or request.cookies.

HttpOnly does not stop the browser from sending the cookie. It only blocks JavaScript from reading it.

### Secure

The browser sends the cookie only over HTTPS. Uny10 enables it when NODE_ENV is production.

### SameSite=Lax

SameSite controls when a cookie is sent in cross-site situations. Lax allows some top-level navigations but blocks many cross-site subrequests. It reduces CSRF risk but does not replace authorization.

### Path=/

The cookie is valid for all application paths, including /api/auth/session, /api/companies, and /api/meetings.

### Max-Age

The browser keeps the cookie for 30 days. The database also stores a 30-day expiry, so both browser lifetime and database validity matter.

## 5. Sessions: how Uny10 remembers a login

A session associates repeated requests with one logged-in browser.

Uny10’s design:

~~~text
Browser cookie:
  refresh_token = random UUID

MongoDB Users record:
  refreshToken = same UUID
  refreshTokenExpires = future Date
~~~

### Why a database-backed opaque session?

Advantages:

- server can revoke it immediately by clearing the database token;
- token contains no user claims to decode;
- membership/status changes are reflected on future validation;
- logout can invalidate cookie and database state.

Trade-offs:

- protected requests need a database lookup;
- the session store must be available;
- a distributed cache may be needed at high scale;
- the token must remain secret.

This differs from a stateless JWT session. A JWT can be verified without a session-row lookup, but immediate revocation is harder.

## 6. User and membership data

The main schema is lib/models/user.ts, stored as Users. Relevant fields include:

- googleId, email, name, avatar;
- teamIds, teamNames;
- teamId and activeTeamId;
- role: coordinator, subcoordinator, or admin;
- status: pending, active, or rejected;
- approvalStatus: pending, approved, or rejected;
- isActive;
- refreshToken and refreshTokenExpires.

A separate TeamMembers collection is used for per-team membership. lib/db/team-members.ts stores:

- userId and teamId;
- role;
- status;
- approvalStatus;
- isActive;
- rejection fields;
- team-specific statistics.

This separation supports a user belonging to multiple teams with different approvals and roles.

## 7. Flow 1 — Google sign-in from click to session cookie

### Flow diagram

~~~text
User clicks Google button
  → app/login/page.tsx GoogleLogin
  → Google returns credential
  → decodeJwtPayload in browser
  → POST /api/auth/google-callback
  → route validates body
  → find/create/update Users record
  → generate UUID session token
  → store token + expiry in Users
  → Set-Cookie: refresh_token
  → JSON status + user
  → browser stores UI user or pending context
  → redirect to dashboard/onboarding
~~~

### Step 1: login page renders provider

app/login/page.tsx is a Client Component because it uses state, effects, and browser events.

It reads NEXT_PUBLIC_GOOGLE_CLIENT_ID. The NEXT_PUBLIC_ prefix exposes a value to browser code. A Google client ID is public configuration; a Google client secret must never be exposed this way.

If the client ID exists, the page renders GoogleOAuthProvider and GoogleLogin. The provider configures the Google client library and the button starts the Google interaction.

### Step 2: Google returns credential

After successful Google authentication, the library calls handleSuccess.

The browser obtains:

~~~text
googleId = decoded.sub
email    = decoded.email
name     = decoded.name
avatar   = decoded.picture
~~~

### Step 3: browser sends application request

The login page sends:

~~~text
POST /api/auth/google-callback
Content-Type: application/json
credentials: include
body: { googleId, email, name, avatar }
~~~

When the server sends Set-Cookie, the browser can store it for this same-origin request.

### Step 4: route handler receives request

app/api/auth/google-callback/route.ts:

1. awaits request.json();
2. extracts Google ID, email, name, avatar;
3. returns 400 when required values are missing.

A 400 means malformed input. A 401 means missing/invalid authentication.

### Step 5: user lookup/reconciliation

The route calls findUserByGoogleId.

If not found, it calls findUserByEmail. This avoids duplicate local accounts when the email already exists.

If email exists, updateGoogleProfile attaches the Google ID/avatar. If neither exists, createGoogleUser inserts a user with:

~~~text
teamId: default-team
activeTeamId: default-team
teamIds: []
role: subcoordinator
status: pending
~~~

default-team is a placeholder, not a real tenant.

### Step 6: local session creation

The route generates a UUID and expiry, then calls updateUserRefreshToken. This creates Uny10’s own session after Google identity has been accepted.

### Step 7: active team reconciliation

reconcileUserActiveTeam:

1. loads User;
2. loads the user’s TeamMembers rows;
3. filters approved active memberships;
4. prefers current activeTeamId if eligible;
5. otherwise selects the first eligible membership;
6. synchronizes global User fields;
7. returns hasEligibleTeam.

### Step 8: cookie response

The route sets refresh_token as HttpOnly, Secure in production, SameSite=Lax, Path=/, Max-Age 30 days.

It returns one of:

- onboarding_required;
- authenticated;
- waiting_for_approval;
- rejected.

### Step 9: browser chooses route

For authenticated, the browser stores result.user in localStorage and navigates to /.

For onboarding/pending/rejected, it stores pending_user in sessionStorage and navigates to /onboarding.

The local objects help the UI. The cookie remains the security credential.

### Security observation

The current browser sends decoded profile claims. Stronger design:

~~~text
Browser sends original Google credential
  → server verifies signature/audience/issuer/expiry
  → server trusts verified subject/email
  → server creates local session
~~~

## 8. Flow 2 — reload and session verification

### Flow diagram

~~~text
Browser loads page
  → app/layout.tsx
  → AuthProvider starts user:null, isLoading:true
  → AuthRouteGuard shows loading
  → getStoredAuthUser reads localStorage
  → GET /api/auth/session with cookie
  → session route queries Users by token + expiry
  → team membership reconciled
  → JSON user response
  → AuthProvider updates Context/localStorage
  → guard renders or redirects
~~~

### Step 1: initial state

AuthProvider in lib/auth-context.tsx begins with user null and isLoading true. Restoration happens in useEffect because localStorage and window do not exist on the server.

### Step 2: cached restoration

getStoredAuthUser in lib/auth-fetch.ts:

1. confirms browser;
2. reads localStorage.auth_user;
3. parses JSON;
4. normalizes ID, email, name, team IDs, role, and status;
5. removes invalid data.

If valid, AuthProvider calls setUser(storedUser) and setIsLoading(false). This makes startup feel fast, but it does not prove the session remains valid.

### Step 3: server verification

AuthProvider sends:

~~~ts
fetch('/api/auth/session', { credentials: 'include' })
~~~

The browser automatically includes HttpOnly refresh_token.

### Step 4: session route validates cookie

app/api/auth/session/route.ts:

1. connects to MongoDB;
2. reads refresh_token;
3. returns 401 if missing;
4. queries User using refreshToken and refreshTokenExpires greater than now.

If no match exists, it returns 401 Invalid or expired session.

### Step 5: team reconciliation

The route calls reconcileUserActiveTeam and hasActiveTeamAccess.

If a globally active user has inconsistent membership fields, the route can repair TeamMembers, recheck access, rotate the token, set a new cookie, and return the repaired user.

If access is still invalid, it returns 403 with code TEAM_ACCESS_REVOKED.

### Step 6: client update

On success, AuthProvider sets Context and refreshes localStorage. Components using useAuth re-render.

On 401/404 it clears localStorage and redirects to login on protected pages. On team revocation it stores pending context and routes to onboarding.

## 9. Flow 3 — protected API request

### Flow diagram

~~~text
Component action
  → authFetch("/api/companies")
  → reads cached team context
  → adds headers/query + cookie
  → middleware.ts
  → route handler
  → enforceTeamAccess
  → getSessionUserId
  → query Users by cookie token
  → hasActiveTeamAccess
  → query Team/TeamMembers
  → repository filters teamId
  → JSON response
  → component state update
~~~

### Step 1: authFetch

authFetch reads cached user and adds:

~~~text
x-team-id
x-user-id
x-user-status
?teamId=...
credentials: include
~~~

These headers help route context and middleware. They are not trusted proof of identity.

### Step 2: middleware

middleware.ts matches /api/:path*.

It skips the configured public Google callback and onboarding routes. For other APIs it:

- reads team header/query;
- rejects a client-supplied x-user-status of rejected;
- requires team context except for /api/auth/*.

Missing team context returns 400.

Middleware is a lightweight early gate. It does not query MongoDB and does not prove x-user-id.

### Step 3: route-level gate

For example, app/api/companies/route.ts:

~~~ts
const teamId = resolveTeamIdFromRequest(request)
const accessResponse = await enforceTeamAccess(request, teamId)
if (accessResponse) return accessResponse
~~~

resolveTeamIdFromRequest prefers team header, then body for writes, then query parameter, then default-team.

### Step 4: trusted identity

lib/validate-session.ts ignores x-user-id. getSessionUserId:

1. reads request.cookies.get('refresh_token');
2. returns null if missing;
3. connects to database;
4. queries User by token and future expiry;
5. selects only _id;
6. returns database-derived user ID.

Null produces a 401 Authentication required response.

### Step 5: team authorization

lib/team-scope.ts hasActiveTeamAccess:

1. rejects empty/default team or user IDs;
2. loads Team;
3. returns 404 when team does not exist;
4. allows the team coordinator;
5. otherwise loads TeamMembers;
6. requires active membership plus approved or active status.

enforceTeamAccess returns 403 for unauthorized access and null for success.

### Step 6: team-filtered business query

After authorization, the route calls a repository such as getAllCompanies(teamId) from lib/db/companies.ts. The repository must query records for that same team ID.

Authorization and filtering are both required:

~~~text
user authorized for team X
  + database query filters teamId = X
  = team X data only
~~~

### Step 7: response/client behavior

The route returns JSON. On 401, authFetch clears localStorage and redirects to login for non-public pages.

## 10. Flow 4 — create or join team

### Diagram

~~~text
Onboarding UI
  → POST /api/auth/onboarding
  → public-route middleware exception
  → rate limit
  → validate action/user
  → create team OR find by join code
  → update Users
  → upsert TeamMembers
  → notify coordinator if joining
  → JSON response
  → local/session storage update
  → dashboard or waiting room
~~~

### Why onboarding is public

A new Google user may not yet have a real team. The route must be reachable before normal team-scoped access exists.

Public routing does not mean safe for arbitrary mutation. The current route accepts userId in the request body and looks up that user. A stronger design would bind the operation to the session cookie.

### Create-team path

app/onboarding/page.tsx sends userId, teamName, and action create_team.

Server steps:

1. reads forwarded IP and calls checkRateLimit;
2. validates userId/action;
3. validates teamName;
4. loads user;
5. counts real team IDs excluding default-team;
6. enforces MAX_TEAMS from lib/constants.ts, currently 5;
7. createTeam normalizes name, checks uniqueness, generates team ID/join code, and inserts team;
8. onboardAsCoordinator updates activeTeamId and first-team global fields;
9. upsertTeamMember creates approved active coordinator membership;
10. returns team, join code, and user.

Client stores user, removes pending_user, routes to /, and shows the code.

### Join-team path

The client sends userId, joinCode, and action join_team.

Server:

1. rate-limits;
2. uppercases join code;
3. calls getTeamByJoinCode;
4. loads user;
5. checks team limit;
6. handles existing active/pending/rejected cases;
7. onboardAsSubcoordinator associates the user;
8. upsertTeamMember creates pending inactive membership;
9. notifyTeam informs coordinators;
10. returns pending user.

Client routes to waiting room.

### Why rate limiting matters

Join codes are guessable secrets. Without rate limiting, attackers could try many codes. The current limiter allows 10 attempts per minute per forwarded IP in process memory. A distributed Redis-backed limiter would be stronger across serverless instances.

## 11. Flow 5 — approval and waiting room

### Diagram

~~~text
Join accepted
  → user status pending
  → /waiting-room
  → wait 2.5 seconds
  → GET /api/auth/session
  → status checked
  → retry about every 12 seconds while visible
  → active → dashboard
  → rejected → onboarding
~~~

app/waiting-room/page.tsx reads local user, redirects active users to dashboard, redirects rejected users to onboarding, and polls session.

The route uses the HttpOnly cookie as the secure session source; the userId query parameter is not the identity proof.

### Pending approval GET

app/api/auth/pending-approvals/route.ts receives teamId and coordinatorId. It loads the team, checks coordinator ownership/role via TeamMembers, then gets pending TeamMembers and joins them with Users.

### Approval PATCH

The body contains userId, action, teamId, and coordinatorId.

The route validates coordinator role and target membership. For approval it:

- updates the user timestamp;
- upserts TeamMembers as active/approved/isActive true;
- adds a member to the Teams document;
- calls reconcileUserActiveTeam.

For rejection it:

- updates the user timestamp;
- upserts TeamMembers as rejected/inactive;
- records rejectedBy and rejectedAt;
- calls reconcileUserActiveTeam.

The actual per-team state lives in TeamMembers. approveUser and rejectUser alone only update the Users timestamp.

### Poll result

Waiting room receives session response. Active stores the user and redirects to dashboard; rejected stores pending context and routes to onboarding. Polling is simple, but not real-time.

## 12. Flow 6 — team switching

### Diagram

~~~text
AppSidebar team menu
  → load memberships through authFetch
  → select approved team
  → POST /api/auth/switch-team
  → route loads User + TeamMember
  → User.activeTeamId updated
  → response contains team context
  → localStorage updated
  → auth_user_updated event
  → AuthProvider updates Context
  → components re-render
~~~

components/dashboard/app-sidebar.tsx loads memberships from /api/team-members, filters rejected memberships, and marks activeTeamId.

On selection:

1. pending teams are blocked with a toast;
2. authFetch posts target teamId;
3. the route reads teamId and user ID from x-user-id or body;
4. it loads User and TeamMember;
5. it requires approved membership;
6. it updates User.activeTeamId;
7. it returns the team-scoped user;
8. the sidebar stores it and dispatches auth_user_updated;
9. AuthProvider listener calls setUser;
10. useAuth consumers re-render;
11. router.refresh refreshes the route.

### Important security weakness

This route does not visibly call getSessionUserId. It relies on x-user-id/body user ID, which a caller can forge. A hardened route should derive the user from refresh_token and only accept target teamId from the body.

## 13. Flow 7 — logout and revocation

### Diagram

~~~text
User clicks Sign Out
  → AuthProvider.logout
  → remove localStorage.auth_user
  → POST /api/auth/logout with cookie
  → clear User.refreshToken and expiry
  → expire browser cookie
  → navigate to /login
~~~

AuthProvider.logout removes localStorage, posts to logout with credentials include, and redirects.

app/api/auth/logout/route.ts:

1. connects to MongoDB;
2. reads refresh_token;
3. clears matching User.refreshToken and refreshTokenExpires;
4. sets an expired refresh_token cookie using maxAge 0;
5. returns success.

Logout has two invalidation steps:

~~~text
database revocation → old token no longer validates
cookie expiry → browser stops sending it
~~~

The client does not await the logout request before redirecting, so a stronger implementation could await it or show retry behavior.

## 14. Role, status, and membership

Do not confuse these:

### Role

Coordinator, subcoordinator, or admin.

### User status

Pending, active, or rejected.

### Approval status

Pending, approved, or rejected.

### isActive

A boolean active flag.

### TeamMember state

The most accurate per-team authorization record is:

~~~text
userId + teamId
  → role
  → status
  → approvalStatus
  → isActive
~~~

One user can have team A approved/active, team B pending, and team C rejected. That is why activeTeamId and TeamMembers lookups matter.

## 15. Middleware versus authoritative authorization

Layered security in Uny10:

~~~text
middleware
  = early request-shape/status filter

getSessionUserId
  = trusted identity from cookie/database

hasActiveTeamAccess
  = tenant membership check

route business logic
  = role/ownership/action rule

database team filter
  = result isolation
~~~

The source comment in middleware says pending and rejected users are denied, but the actual code rejects only a client-supplied rejected status and permits pending requests through. Do not repeat the comment as if it were the full runtime behavior.

## 16. Security properties and limitations

### XSS

XSS means injected JavaScript executes in the application origin. HttpOnly prevents direct cookie reads, but XSS could still act through the user’s browser while the session is active.

### CSRF

CSRF tricks a browser into sending a state-changing request with cookies. Uny10 uses SameSite=Lax and same-origin APIs as partial defenses. The visible code does not show a separate CSRF token. High-risk mutations should add one and validate origin where appropriate.

### Session fixation

Session fixation causes a victim to use an attacker-known session ID. Uny10 generates a fresh UUID at login and has rotation in a session-repair path. Rotation after sensitive privilege changes may also be appropriate.

### Token leakage

The session token should not appear in URLs, logs, or client-readable storage. Uny10 keeps it in an HttpOnly cookie.

### Brute force

Onboarding join-code attempts are rate-limited, but the limiter is in-memory and process-local. A shared limiter is needed for reliable multi-instance enforcement.

### Client identity trust

The strongest code comment in lib/validate-session.ts says never trust x-user-id. Some routes still accept user IDs from body/query/header. That inconsistency is a major improvement item.

## 17. Recommended hardened design

For every protected request:

~~~text
HttpOnly session cookie
  → server validates cookie
  → server derives sessionUserId
  → body supplies only requested resource/team
  → server verifies sessionUserId has active approved membership
  → server verifies route-specific role
  → database query uses authorized teamId
~~~

For Google:

~~~text
Original Google credential
  → server verifies signature, issuer, audience, expiry, subject
  → find/create local User
  → create opaque local session
  → set HttpOnly cookie
~~~

Recommended implementation improvements:

- reusable requireSession and requireTeamAccess functions;
- no body/header identity for protected operations;
- server-side verification of the original Google credential;
- schema validation for every JSON body;
- explicit CSRF protection for sensitive mutations;
- cross-team integration tests;
- distributed rate limiting;
- audit logs for approvals/team changes;
- consistent use of one secure auth path in Server Actions and route handlers.

## 18. What I learned

- Authentication asks who; authorization asks what the user may do.
- Google provides the initial identity signal; Uny10 creates its own local session.
- Google credential is JWT-shaped, but Uny10 refresh_token is an opaque UUID stored in MongoDB.
- HttpOnly blocks JavaScript reads; Secure requires HTTPS; SameSite limits cross-site sending; Max-Age controls browser lifetime.
- Database expiry determines whether the session is still valid.
- localStorage.auth_user is a UI cache, not proof of identity.
- authFetch adds context headers and includes cookies, but headers are forgeable.
- middleware.ts is an early guard; getSessionUserId and enforceTeamAccess are the important server boundary.
- TeamMembers stores per-team authorization.
- Onboarding creates or requests team access; waiting room polls until approval.
- Logout must clear both database session and cookie.
- Decode is not verify: parsing Google claims does not prove their signature.

## 19. End-to-end authentication flow

~~~text
1. User clicks GoogleLogin
2. Google returns credential to handleSuccess
3. Browser decodes claims and POSTs /api/auth/google-callback
4. Route validates fields
5. findUserByGoogleId/findUserByEmail finds local User
6. createGoogleUser creates default-team pending user if needed
7. crypto.randomUUID creates local session token
8. updateUserRefreshToken stores token and expiry
9. reconcileUserActiveTeam resolves approved membership
10. Set-Cookie sends HttpOnly refresh_token
11. JSON status tells browser authenticated/onboarding/pending/rejected
12. Browser stores display metadata
13. AuthProvider calls GET /api/auth/session
14. Server queries Users by cookie token and expiry
15. Server checks active team membership
16. AuthProvider updates Context
17. AuthRouteGuard renders or redirects
18. Feature calls authFetch/API
19. Cookie travels automatically
20. Middleware performs early checks
21. Route derives trusted user ID
22. Team helper verifies membership
23. Repository filters by teamId
24. JSON returns
25. Component updates state and UI
~~~

## 20. How I should explain this to an interviewer

“Uny10 separates authentication from authorization. Google sign-in is the initial identity provider. app/login/page.tsx receives a Google credential and posts profile claims to /api/auth/google-callback. The route finds or creates a Users record, generates a random UUID, stores it with a 30-day expiry, and sets it as an HttpOnly SameSite cookie named refresh_token.

That cookie is Uny10’s database-backed session token. It is opaque, not a JWT. On protected requests, the browser sends it automatically. lib/validate-session.ts looks up the token and expiry in MongoDB and derives the real user ID; it does not trust x-user-id. lib/team-scope.ts then checks whether that trusted user is the coordinator or an approved active member of the requested team. Only then does the route execute a team-filtered query.

The frontend stores auth_user in localStorage for fast rendering, but that cache is not trusted. AuthProvider restores it and verifies /api/auth/session; AuthRouteGuard handles client redirects. Onboarding creates a coordinator membership or pending TeamMembers record, the waiting room polls until approval, and logout clears both database token and cookie.

The main strengths are revocable server-side sessions and per-team membership checks. The main improvements are verifying the original Google credential server-side, deriving user IDs from the cookie in every auth route, adding CSRF protection, centralizing authorization, and using distributed rate limiting.”

## 21. Likely interviewer questions and answers

### Q1. Authentication versus authorization?

Authentication identifies the user. Authorization decides whether that user may access a resource or perform an operation. Cookie validation authenticates; TeamMembers/team checks authorize.

### Q2. Is refresh_token a JWT?

No. It is a UUID from crypto.randomUUID stored in Users with an expiry. It is an opaque database-backed session token.

### Q3. Why use HttpOnly?

The browser sends it automatically, but JavaScript cannot read it. This reduces direct token theft through XSS. It does not eliminate XSS or CSRF.

### Q4. Why not trust localStorage?

Users can edit localStorage. It is only a cache for UI and request context. The server must use the cookie and database lookup.

### Q5. What does credentials include do?

It tells fetch to include cookies in the request. Uny10 needs it for session, logout, and protected API requests.

### Q6. What happens when a session expires?

The token query fails because the token is missing or expiry is not greater than now. The server returns 401; client auth code clears cached data and redirects to login.

### Q7. How is team isolation enforced?

The server derives a trusted user ID from the cookie, checks coordinator or active approved TeamMembers membership, and repositories filter by the same teamId.

### Q8. Why separate TeamMembers from Users?

A user can belong to multiple teams with different roles and approval states. Users stores identity/current context; TeamMembers stores per-team authorization.

### Q9. What does middleware do?

It runs before routes and performs cheap checks such as team context and a rejected client status. It is not the authoritative identity check.

### Q10. Is x-user-status secure?

No. It is client-controlled. It can be a hint, but security must use cookie-derived identity and database membership.

### Q11. Is decoding a Google JWT enough?

No. Decoding only reads claims. The server should verify signature, issuer, audience, expiry, and subject using the original credential.

### Q12. How does a new user become coordinator?

Google user without a real team enters onboarding. createTeam, onboardAsCoordinator, and an approved active TeamMembers record are executed, then the user enters the dashboard.

### Q13. How does joining work?

A join code is checked, a pending inactive membership is created, coordinators are notified, and the user waits until approval.

### Q14. How does waiting room detect approval?

It polls /api/auth/session after a short delay and periodically while visible. Active status redirects to the dashboard.

### Q15. What does logout do?

Client cache is removed, logout API clears the matching database token and expiry, the cookie is expired, and the browser goes to login.

### Q16. What is the biggest weakness?

Authorization is inconsistent. Some routes accept body/header user IDs and the Google callback trusts browser-decoded claims. Every protected operation should derive user ID from the cookie and share one authorization wrapper.

## 22. Important things to remember

- Authentication = who; authorization = what.
- Google identity and Uny10 session are separate stages.
- refresh_token is a UUID session token, not a JWT or Google refresh token.
- HttpOnly, Secure, SameSite, Path, and Max-Age have different purposes.
- localStorage is not secure identity.
- Never trust x-user-id, x-user-status, or body userId without cookie binding.
- middleware is not the complete auth system.
- getSessionUserId is the identity source.
- enforceTeamAccess verifies tenant access.
- TeamMembers is the per-team authorization record.
- teamId belongs in authorization and database filters.
- Decode is not verify.
- Logout requires database revocation and cookie expiration.
- Explain both the intended design and actual code gaps in interviews.

