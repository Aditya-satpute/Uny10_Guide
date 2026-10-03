# Topic 2 — Frontend Architecture, Next.js Rendering, and Real UI Execution Flows

This note teaches how the browser-side part of Uny10 actually runs. The goal is to connect framework concepts to real code and trace execution, not memorize file lists.

## 1. Mental model

A web application has two places where code can run:

- **Server:** the machine running Next.js. It can use server environment variables, talk to MongoDB, and produce HTML or data.
- **Browser/client:** the user’s tab. It can respond to clicks, hold temporary state, access window, localStorage, sessionStorage, and browser APIs, and update the screen.

Uny10 deliberately uses both:

~~~text
Browser event
  → React component handler
  → state update or server call
  → Server Action OR /api route
  → database/external service
  → returned value
  → React state update
  → re-rendered UI
~~~

## 2. Terms explained before tracing code

### Next.js

**Meaning:** Next.js is a framework built on React. React provides components and UI updates; Next.js adds routing, server rendering, server code, layouts, asset handling, and deployment conventions.

**Why Uny10 needs it:** Uny10 has many product pages, shared navigation, interactive browser features, and backend endpoints. Next.js lets the project keep these in one TypeScript repository.

**Where:** app/, app/api/, next.config.mjs, package.json, and imports from next/navigation, next/dynamic, next/image, and next/script.

### App Router

**Meaning:** Next.js App Router maps folders and special files under app/ to URLs. A page.tsx is a page; [type] and [id] are dynamic URL segments; layout.tsx wraps routes.

**Why:** It gives Uny10 a predictable route tree and lets shared providers/navigation be mounted once.

Examples:

~~~text
app/page.tsx                       → /
app/companies/page.tsx             → /companies
app/companies/[type]/[id]/page.tsx → /companies/:type/:id
app/map/page.tsx                   → /map
app/api/companies/route.ts         → /api/companies
~~~

### Component

**Meaning:** A component is a reusable function that returns UI. It receives inputs called props and can contain state and event handlers.

**Why:** Screens are easier to build when composed from smaller parts. app/companies/page.tsx supplies the shell, while components/companies/companies-content.tsx owns the feature interaction.

### Server Component and Client Component

A **Server Component** executes as part of server rendering and is not shipped as interactive browser JavaScript. It cannot use browser event handlers or client hooks such as useState.

A **Client Component** starts with the directive 'use client'. Its code can run in the browser and use state, effects, click handlers, browser storage, and client navigation.

**Why the split exists:** static composition is cheaper on the server; interactivity must run in the browser. The split also keeps server-only capabilities away from browser code.

**Uny10 examples:**

- app/companies/page.tsx has no 'use client', so it is a server-side page shell.
- components/companies/companies-content.tsx starts with 'use client' because it uses state, effects, URL search parameters, file input events, and browser downloads.
- app/page.tsx starts with 'use client' because it reads useAuth() and useIsMobile().
- app/layout.tsx is a server layout but renders client components such as AuthProvider and AuthRouteGuard.

### Rendering and hydration

**Rendering** means turning component code and data into a UI representation.

**Hydration** is when React in the browser attaches event handlers and client state to HTML that was initially produced for the page. After hydration, a visible button becomes a live React button.

**Why this matters:** Uny10 reads browser-only values such as window.innerWidth and localStorage inside effects or client-only code. Reading them during server rendering would fail because the server has no browser window.

### Hook

A **Hook** is a React function beginning with use that lets a function component use React features.

- useState stores a changing value and causes a re-render when its setter runs.
- useEffect runs browser/network/external work after rendering and can return cleanup code.
- useMemo derives a value and can avoid recalculation when dependencies did not change.
- useRef stores a mutable value or DOM reference without itself causing a re-render.
- useContext reads shared Context state.
- useRouter, usePathname, and useSearchParams are Next.js client-navigation hooks.

### State, props, and re-render

**State** is data owned by a component, such as companies, loading, or activeTab.

**Props** are values passed from parent to child, such as refreshSignal passed from CompaniesContent to AllCompanies.

When a setter such as setCompanies(next) runs, React schedules a re-render. The component function runs again with new state and React updates the necessary DOM.

### Context

**Meaning:** Context makes a value available to descendants without passing it through every intermediate component.

**Why:** Many unrelated dashboard components need the current user and logout function. AuthProvider stores it once; useAuth() reads it anywhere below the provider.

**Where:** lib/auth-context.tsx creates AuthContext, AuthProvider supplies it, and useAuth() consumes it. app/layout.tsx places AuthProvider above AuthRouteGuard and every page.

### Client-side navigation

**Meaning:** a route changes using browser history and Next.js without a traditional full-document reload.

**Why:** Shared layout and client state can remain mounted, making navigation feel like an application.

**Where:** AppSidebar uses Link href="/companies"; company rows use router.push; guards use router.replace; some auth paths intentionally use window.location.href for a full reload.

### Server Action

**Meaning:** a function that Next.js executes on the server even when called from Client Component code. A file beginning with 'use server' marks its exported functions as server-side callable actions.

**Why:** The UI can call a function-shaped server boundary without manually writing a public JSON endpoint for that particular operation.

**Where:** lib/actions/companies.ts begins with 'use server'. components/companies/companies-content.tsx imports getAllCompanies and createCompany directly from it.

Important source distinction: the current Companies UI does not call fetch('/api/companies') for its main load/create path. It invokes Server Actions. The explicit app/api/companies/route.ts exists and is used by other consumers, such as Map.

### API route / Route Handler

**Meaning:** a server function exported from app/api/.../route.ts that handles HTTP methods such as GET and POST.

**Why:** It gives the browser and other clients a normal HTTP/JSON interface.

**Examples:** app/api/auth/session/route.ts, app/api/meetings/route.ts, app/api/visit-plans/route.ts, and app/api/companies/route.ts.

### Cookie, session, and localStorage

A **cookie** is a small browser-managed value automatically sent with matching HTTP requests. Uny10’s refresh_token cookie is HttpOnly, so JavaScript cannot read it.

A **session** is server-maintained information identifying a logged-in browser. Uny10 stores a random token in the cookie and also stores that token and its expiry on the User record.

**localStorage** is browser storage readable by JavaScript and retained across reloads. Uny10 stores a normalized auth_user object there for fast UI restoration and request context.

Both exist because localStorage improves startup UX, while the HttpOnly cookie is the security credential. LocalStorage can be edited by the user and cannot be the authoritative identity source.

## 3. Root composition: how every page is wrapped

The root file is app/layout.tsx:

~~~tsx
<html lang="en">
  <body>
    <AuthProvider>
      <PwaRegistration />
      <AuthRouteGuard>{children}</AuthRouteGuard>
    </AuthProvider>
    {/* production scripts */}
  </body>
</html>
~~~

Read it from outside inward:

1. Next.js uses RootLayout for the entire application.
2. AuthProvider creates shared client authentication state.
3. PwaRegistration registers public/sw.js in the browser.
4. AuthRouteGuard decides whether the current route may be shown.
5. children is whichever page the user requested.

The layout also configures metadata, the manifest, logos, fonts, production Google Analytics, Microsoft Clarity, Vercel Analytics, and development-only Agentation. next/script with strategy="afterInteractive" delays analytics until after the page is interactive instead of blocking the first output.

## 4. Flow 1 — Initial application load and authentication bootstrap

### Flow diagram

~~~text
Browser requests page
  → RootLayout
  → AuthProvider initial state
  → AuthRouteGuard loading screen
  → useEffect reads localStorage/sessionStorage
  → GET /api/auth/session with cookie
  → React auth state and local cache update
  → guard redirects or renders page
~~~

### Step 1: Next.js selects the page and layout

Suppose the user requests /companies.

1. The App Router finds app/companies/page.tsx.
2. It includes the parent app/layout.tsx.
3. CompaniesPage returns SidebarProvider, AppSidebar, TopNavbar, and CompaniesContent.

CompaniesPage has no 'use client', so it is a server-side composition function. CompaniesContent is a client boundary because its interaction needs browser code.

### Step 2: AuthProvider starts unknown/loading

lib/auth-context.tsx initializes:

~~~tsx
const [user, setUser] = useState<AuthUser | null>(null)
const [isLoading, setIsLoading] = useState(true)
~~~

At first:

- user is null because nothing has been restored;
- isLoading is true so the app cannot make an early redirect decision.

The useEffect near line 47 runs after client mount. That is important because localStorage and window do not exist during server rendering.

### Step 3: AuthRouteGuard shows a temporary loading screen

components/auth/auth-route-guard.tsx reads user and isLoading with useAuth(). It derives routeType using useMemo:

~~~tsx
if (pathname.startsWith('/login')) return 'auth'
if (pathname.startsWith('/onboarding')) return 'onboarding'
if (pathname.startsWith('/waiting-room')) return 'waiting'
return 'app'
~~~

While ready is false or isLoading is true, it returns a spinner rather than children. This prevents a protected dashboard from flashing before the auth decision.

### Step 4: cached user restoration

The AuthProvider effect calls getStoredAuthUser() from lib/auth-fetch.ts.

That helper:

1. Confirms it is running in a browser.
2. Reads localStorage key auth_user.
3. Parses JSON.
4. Normalizes id, email, name, team IDs, role, and status.
5. Removes invalid data.

If valid data exists, the provider calls setUser(storedUser) and setIsLoading(false). The interface can now render quickly, even though server verification is still running.

This object is display/request context, for example:

~~~ts
{
  id: "...",
  name: "Asha",
  teamId: "team_abc123",
  activeTeamId: "team_abc123",
  role: "coordinator",
  status: "active"
}
~~~

### Step 5: server session verification

The provider then sends:

~~~ts
fetch('/api/auth/session', { credentials: 'include' })
~~~

credentials: 'include' tells the browser to include cookies. app/api/auth/session/route.ts reads refresh_token on the server, checks expiry against MongoDB, reconciles team state, and returns a server-derived user.

On success:

1. setUser(data.user) replaces cached state.
2. setStoredAuthUser(data.user) refreshes localStorage.
3. components using useAuth() re-render.

On 401 or 404, non-public pages clear local auth and navigate to /login. On revoked team access, the provider moves the previous user into sessionStorage.pending_user and redirects to /onboarding.

On network failure, the provider keeps cached data when possible. This improves resilience but can briefly show stale state.

### Step 6: guard decides whether to show the page

After isLoading is false:

- app route + no user → /login, or /onboarding if pending context exists;
- pending user → /waiting-room;
- rejected user → /login;
- active user → render the requested page.

This is client-side navigation protection. It is not a replacement for server authorization because a user can bypass browser code and call an endpoint directly.

## 5. Flow 2 — Sidebar navigation to Companies

### Flow diagram

~~~text
Click Companies
  → AppSidebar Link
  → Next client navigation
  → app/companies/page.tsx page shell
  → CompaniesContent client boundary
  → AllCompanies child
  → effects load data
~~~

### Step 1: Link creates navigation

components/dashboard/app-sidebar.tsx defines mainNavItems, including:

~~~ts
{ title: "Companies", icon: Building2, href: "/companies" }
~~~

The rendered item is a Next Link. On mobile the click also calls setOpenMobile(false). Next.js changes the route without a full document reload.

usePathname marks the active item. useRouter is used for actions such as router.push to onboarding and team switching.

### Step 2: page shell composes shared UI

app/companies/page.tsx returns:

~~~tsx
<SidebarProvider>
  <AppSidebar />
  <SidebarInset>
    <TopNavbar />
    <main><CompaniesContent /></main>
  </SidebarInset>
</SidebarProvider>
~~~

SidebarProvider is a UI Context provider from components/ui/sidebar.tsx. It lets sidebar primitives share open/collapsed state.

### Step 3: CompaniesContent creates local state

Important state in components/companies/companies-content.tsx:

~~~tsx
const [companies, setCompanies] = useState<CompanyData[]>([])
const [loading, setLoading] = useState(true)
const [actionLoading, setActionLoading] = useState(false)
const [companiesRefreshSignal, setCompaniesRefreshSignal] = useState(0)
const [form, setForm] = useState<AddCompanyForm>(initialForm)
~~~

The first render has no companies, is loading, has no active save, and has a blank form.

A controlled input is an input whose displayed value comes from React state. For example, value={form.name} and an onChange setter mean React owns the field.

### Step 4: URL query chooses a tab

useSearchParams reads the query string. initialTab is derived with useMemo:

~~~tsx
const tab = searchParams.get("tab")
if (tab === "add" || tab === "upload" || tab === "download") return tab
return "all"
~~~

The value is UI/navigation state, not database state. A later effect synchronizes activeTab if the URL changes.

### Step 5: load effect invokes a Server Action

On mount:

~~~tsx
useEffect(() => {
  const loadCompanies = async () => {
    const userRaw = localStorage.getItem("auth_user")
    const teamId = userRaw ? JSON.parse(userRaw).teamId : undefined
    const data = await getAllCompanies(teamId)
    setCompanies(data)
    setLoading(false)
  }
  loadCompanies()
}, [])
~~~

Actual execution:

1. The browser mounts CompaniesContent.
2. React runs the effect after the first render.
3. The component reads cached teamId from localStorage.
4. getAllCompanies is imported from lib/actions/companies.ts, whose first line is 'use server'.
5. Next.js sends the action invocation to server-side code. There is no named app/api/companies route in this call.
6. The server action calls connectToDatabase().
7. It builds a Mongoose query. For a real team it uses { teamId }; for absent/default it uses a compatibility $or.
8. Company.find(query).lean().exec() reads plain objects.
9. toPlainCompany converts IDs, dates, contacts, follow-ups, and coordinates into serializable CompanyData.
10. The result crosses back to the client action call.
11. setCompanies(data) stores it.
12. setLoading(false) removes loading.
13. React re-renders.

all-companies.tsx separately imports getAllCompanies. Its effect runs on mount and whenever refreshSignal changes:

~~~tsx
useEffect(() => {
  const teamId = getAuthTeamId()
  const data = await getAllCompanies(teamId)
  setCompanies(data)
}, [refreshSignal])
~~~

So the page has overlapping list-loading responsibilities: parent state supports export/form behavior, while AllCompanies owns its own table state.

### Important source observation

app/api/companies/route.ts also exposes GET and POST and calls enforceTeamAccess before lib/db/companies.ts. But the current Companies components use the Server Action functions in lib/actions/companies.ts, not apiClient.getAllCompanies() or authFetch('/api/companies').

Two actual data paths exist:

~~~text
Companies page → Server Action → Mongoose model

Map/other API consumers → authFetch → /api/companies → team gate → repository
~~~

## 6. Flow 3 — Creating a company from the actual UI

### Flow diagram

~~~text
Input onChange
  → setForm
  → React re-render
  → click Create Company
  → handleCreateCompany
  → local validation and payload creation
  → Server Action createCompany
  → Mongoose connection and Company.save
  → user/team stats sync
  → serialized result
  → setCompanies and refresh signal
  → AllCompanies reloads
  → toast/UI update
~~~

### Step 1: typing changes state

The company name input follows this pattern:

~~~tsx
<Input
  value={form.name}
  onChange={(event) =>
    setForm((previous) => ({
      ...previous,
      name: event.target.value
    }))
  }
/>
~~~

1. value makes the input display form.name.
2. The browser emits an input event.
3. event.target.value is the new string.
4. The functional setter receives the latest form object.
5. The spread preserves other fields.
6. React stores the new form and re-renders the input.

### Step 2: click handler validates and constructs data

The Create Company button calls handleCreateCompany.

It first checks type, non-empty name, and non-empty amount. If invalid, it shows a Sonner toast and returns before any server call.

If valid, it:

- sets actionLoading true;
- trims contacts and removes incomplete POC rows;
- resolves the selected sponsorship contact from sponsorshipPocOptions;
- converts the amount string to a number;
- includes an offline location or online reference URL;
- initializes specialDeliverables and followUps;
- reads teamId from localStorage;
- calls createCompany(payload, teamId).

### Step 3: Server Action saves

lib/actions/companies.ts executes server-side:

1. connectToDatabase reuses or creates a Mongoose connection.
2. withDerivedCoordinates checks existing coordinates.
3. It may expand approved map URLs and extract latitude/longitude patterns.
4. payload.teamId = teamId associates the record with a team.
5. new Company(payload) applies the Mongoose Company schema.
6. company.save() inserts the document.
7. syncAllUserStatsFromCompanies recalculates active user/team-member statistics.
8. toPlainCompany(savedCompany.toObject()) returns a serializable result.

### Step 4: client updates after success

The component receives created:

~~~tsx
setCompanies((previous) => [...previous, created])
setCompaniesRefreshSignal((previous) => previous + 1)
setForm(initialForm)
toast.success('Company added successfully')
~~~

The first line immediately adds the result to parent state. The second changes a prop, causing AllCompanies to reload. The form resets and the user gets feedback.

This gives a quick response plus a fresh read, but it also creates duplicate work.

### Step 5: failure cleanup

Errors are caught, logged, and shown as “Could not add company.” The finally block always calls setActionLoading(false), preventing a failed request from leaving the button disabled forever.

## 7. Flow 4 — Filtering and detail navigation

### Diagram

~~~text
User types search/filter
  → setSearchQuery or setStatusFilter
  → AllCompanies re-renders
  → filteredCompanies runs in memory
  → table rows change
~~~

AllCompanies keeps searchQuery, statusFilter, and typeFilter in local state. It filters the already-loaded array by company name, POC names, status, and type. No database request occurs for every keystroke.

This is **client-side filtering**. It is instant for a modest dataset, but a large tenant would need pagination, debounced requests, server-side filtering, and database indexes.

When a row is clicked, handleRowClick ignores clicks from controls such as buttons, links, inputs, and menu items. Otherwise it calls router.push with /companies, the company type, id, and encoded name. The dynamic detail page reads route parameters with useParams and optional query values with useSearchParams.

## 8. Flow 5 — Dashboard responsiveness and custom Hook

app/page.tsx is a Client Component because it calls useAuth() and useIsMobile().

hooks/use-mobile.ts:

~~~tsx
const [isMobile, setIsMobile] = React.useState<boolean | undefined>(undefined)

React.useEffect(() => {
  const mql = window.matchMedia('(max-width: 767px)')
  const onChange = () => {
    setIsMobile(window.innerWidth < 768)
  }
  mql.addEventListener('change', onChange)
  setIsMobile(window.innerWidth < 768)
  return () => mql.removeEventListener('change', onChange)
}, [])

return !!isMobile
~~~

Execution:

1. First render: isMobile is undefined, so !!undefined is false.
2. After browser mount, the effect reads window.innerWidth.
3. setIsMobile stores true or false and causes a re-render.
4. A media-query change updates it again.
5. Cleanup removes the listener on unmount.

The dashboard renders a mobile or desktop component arrangement. CSS classes also provide responsive layout, so JavaScript and CSS share responsibility.

The initial false can briefly choose the desktop branch before the effect updates. A stricter design could return undefined and show a neutral skeleton until the breakpoint is known.

## 9. Flow 6 — Map and browser-only Leaflet

A **browser-only library** expects window, document, or the DOM while loading or initializing.

A **dynamic import** loads a module later instead of putting it in the initial bundle.

**SSR** means Server-Side Rendering: producing initial HTML on the server.

Leaflet is browser-oriented, so MapContent uses:

~~~tsx
const MapView = dynamic(
  () => import("./map-view").then((mod) => mod.MapView),
  {
    ssr: false,
    loading: () => <LoadingMap />
  }
)
~~~

### Diagram

~~~text
/map page shell
  → MapContent client component
  → useEffect calls authFetch('/api/companies')
  → companies state
  → dynamic MapView with ssr:false
  → MapView useEffect creates Leaflet map in a DOM ref
  → later effects synchronize markers
~~~

### Step 1: fetch companies

MapContent starts with companies=[] and isLoadingCompanies=true. Its effect calls authFetch('/api/companies').

authFetch reads the normalized cached user, adds x-team-id, x-user-id, and x-user-status, appends teamId, includes cookies, and returns the HTTP response.

The route then performs middleware and enforceTeamAccess checks. On success, MapContent stores the companies. useMemo derives coordinate-filtered, searched, and nearby lists.

### Step 2: load map module

ssr:false prevents Next.js from rendering MapView on the server. The fallback displays “Loading map...” while the browser loads it.

### Step 3: initialize Leaflet after mount

MapView uses:

~~~tsx
const mapRef = useRef<L.Map | null>(null)
const mapContainerRef = useRef<HTMLDivElement>(null)
~~~

A ref is used because Leaflet is imperative: code commands the map object directly.

Its first effect waits until mapContainerRef.current exists, calls L.map, creates an OpenStreetMap tile layer, saves the map in mapRef, and sets mapReady(true). Cleanup calls map.remove() and clears the ref.

### Step 4: synchronize map objects

When companies changes, a marker effect removes stale markers, creates markers for companies with coordinates, and changes icons based on status/selection.

Other effects respond to selectedCompany, company bounds, userLocation, and radius. The integration pattern is:

~~~text
React state represents desired values
  → effect observes dependency changes
  → imperative Leaflet API is updated
  → cleanup removes external resources
~~~

## 10. Flow 7 — Shared auth state and team switching

### Diagram

~~~text
User opens team menu
  → AppSidebar reads useAuth()
  → effect loads memberships through authFetch
  → user selects approved team
  → POST /api/auth/switch-team
  → local cache updates
  → auth_user_updated event
  → AuthProvider setUser
  → all useAuth consumers re-render
  → router.refresh()
~~~

AuthProvider supplies user, isLoading, teamId, userStatus, userRole, logout, and refreshUser.

AppSidebar reads it to show the user and current team. Its effect requests memberships from /api/team-members with authFetch, filters rejected memberships, and constructs a team list.

When a different approved team is selected:

1. handleSwitchTeam prevents selecting the current team.
2. It blocks pending memberships with a toast.
3. It calls authFetch with POST /api/auth/switch-team and body { teamId }.
4. The server returns updated user/team context.
5. setStoredAuthUser writes it to localStorage.
6. A CustomEvent named auth_user_updated is dispatched.
7. AuthProvider’s window listener receives it and calls setUser(detail).
8. Every useAuth consumer sees the new context and re-renders.
9. router.refresh refreshes the current route and mobile navigation closes.

The custom event is a small browser event bus. It is useful because sidebar and provider are not connected by a direct setter, but it can become harder to track than a dedicated global store as the app grows.

## 11. State-management patterns

### Local component state

Used for form fields, filters, selected company, loading flags, modal visibility, and active tabs. This is the dominant feature pattern.

### React Context

Used for cross-cutting authentication state. It avoids passing the user through every dashboard component.

### Browser storage

- localStorage.auth_user: normalized cached user.
- sessionStorage.pending_user: short-lived onboarding context.

### Zustand

lib/company-store.ts uses Zustand create(). Zustand is a small external store library. It lets components subscribe to store data without manually creating a Context provider.

The current store contains an in-memory initialCompanies dataset and actions such as addCompany, updateCompany, deleteCompany, and updateStatus. The main database-backed Companies screen uses Server Actions instead, so this store is an alternate/legacy local state path.

### Derived state

useMemo derives values such as sponsorship contact options, filtered companies, map coordinate lists, and tab state. It does not make data persistent or secure.

## 12. Design decisions and trade-offs

### Why Companies page is server shell plus client child

The page only composes layout. The child needs hooks, form events, file input, localStorage, generated downloads, toasts, and URL state. Keeping the shell server-side reduces the client boundary while preserving interaction.

### Why dashboard page is client-marked

It directly uses auth context and browser-dependent mobile detection and chooses different component trees. The cost is a larger client boundary for that route.

### Why map uses dynamic import and ssr:false

Leaflet needs the browser DOM. Deferring it avoids server failure and reduces initial JavaScript. The cost is a loading state and later map appearance.

### Why effects have cleanup

Effects create work outside React: listeners, timers, subscriptions, and map objects. Cleanup prevents leaks and duplicate behavior. use-mobile removes its media listener; MapView removes Leaflet; AuthProvider removes its custom event listener.

### Why client-side filtering is not infinitely scalable

It is fast and simple for a loaded list, but it requires loading all records and matching in the browser. Large datasets need server-side filtering and pagination.

### Why both Server Actions and API routes exist

Server Actions are convenient for direct server-function calls. API routes are explicit HTTP contracts useful for maps, independent clients, and testing. Both increase flexibility but also create duplicate data paths and possible authorization differences.

## 13. Security and correctness observations

1. **The auth cache is not trusted.** localStorage can be edited. The secure cookie and server checks should be authoritative.
2. **Company Server Actions need stronger authorization.** lib/actions/companies.ts accepts teamId from the client and does not call getSessionUserId/enforceTeamAccess itself. app/api/companies/route.ts does call the common team gate, but the Companies page does not use that route for its main path.
3. **Some action operations are not team-filtered.** updateCompany and deleteCompany operate by ID; getCompanyById only adds a team filter for a non-default team. This is a potential cross-team gap if an untrusted caller can invoke the action.
4. **Duplicate loading exists.** Parent and AllCompanies both call getAllCompanies; create updates local state and then triggers a fresh load.
5. **Browser-only reads must stay in client effects/handlers.** useIsMobile follows that rule; new code should not read window or localStorage during server rendering.
6. **Leaflet tooltip HTML needs escaping.** MapView interpolates company names and locations into HTML strings; untrusted values should be escaped or sanitized.
7. **Error handling is inconsistent.** Some features use toasts, some strings, and auth helpers redirect on 401. A shared policy would be easier to reason about.

## 14. Improvements worth proposing

- Choose one primary company data path and put shared validation/auth around it.
- Make Server Actions derive the user from the HttpOnly cookie and verify team membership.
- Let one component own the company list, passing data to children or using a query/cache library.
- Add Zod or equivalent server-side validation for create/update/import payloads.
- Add pagination, debounced search, and server-side filtering for large teams.
- Return an explicit unknown mobile state and show a neutral skeleton if layout shift is unacceptable.
- Escape user-controlled strings interpolated into Leaflet HTML.
- Standardize loading/error/empty UI and add error boundaries.
- Use server session bootstrap or server rendering for initial auth, with localStorage only as optional UX cache.

## 15. What I learned

- React supplies components and state updates; Next.js supplies routing, server/client boundaries, rendering, and server execution.
- app/ folders define pages and API endpoints; components/ holds reusable feature UI.
- 'use client' means browser interactivity; 'use server' marks callable server-side functions.
- useState stores changing UI data; useEffect runs after render; useMemo derives values; useRef stores DOM or imperative objects.
- AuthProvider is shared auth state; AuthRouteGuard is the client navigation gate.
- Uny10 uses both Server Actions and explicit API routes.
- The current Companies page uses Server Actions, while Map uses authFetch('/api/companies').
- Leaflet is dynamically imported with SSR disabled because it requires the DOM.
- localStorage speeds UI restoration but is not secure identity.
- State changes re-render components; effects and custom events connect React to browser APIs and other modules.

## 16. How this frontend works from start to finish

~~~text
1. Browser requests a route
2. Next.js finds app/page and wraps it in app/layout
3. Client providers initialize
4. AuthProvider restores cache and verifies /api/auth/session
5. AuthRouteGuard chooses loading, redirect, or page
6. Page composes shared shell and feature client component
7. Feature initializes state and effects
8. Effects read URL/storage or call Server Action/API route
9. Server queries MongoDB or external service
10. Result returns to the client boundary
11. setState changes component state
12. React re-renders affected UI
13. Effect cleanup runs when dependencies/components change
~~~

Concrete Companies create flow:

~~~text
Input onChange
  → setForm
  → React re-render
  → click Create Company
  → handleCreateCompany
  → local validation and payload
  → Server Action createCompany
  → connectToDatabase
  → coordinate enrichment
  → Mongoose Company.save
  → statistics synchronization
  → plain serializable result
  → setCompanies + refresh signal
  → AllCompanies reload
  → table and toast update
~~~

## 17. How I should explain this topic to an interviewer

“Uny10 uses the Next.js App Router with a deliberate server/client split. The root app/layout.tsx is the shared server layout, but it mounts client providers: AuthProvider, AuthRouteGuard, and PWA registration. Simple page files such as app/companies/page.tsx compose the shell, while interactive components such as CompaniesContent use client hooks for forms, URL tabs, file input, and browser downloads.

On mount, the Companies component reads cached team context and calls the use-server getAllCompanies action. That action runs Mongoose queries on the server and returns serializable data. The project also exposes /api/companies; Map uses that explicit API through authFetch, so the codebase has both Server Action and HTTP API patterns.

Leaflet is dynamically loaded with ssr:false because it requires the browser DOM. The main frontend trade-offs are a unified codebase and responsive interactive UI versus duplicated data paths, local-cache trust boundaries, and uneven authorization in the company Server Actions.”

For a concrete create explanation: “The input is controlled by form state. Clicking Create Company invokes handleCreateCompany, which validates and constructs CompanyData, calls the server createCompany action, connects to MongoDB, enriches coordinates, saves a Mongoose Company, synchronizes stats, returns a plain object, and then the browser updates local state, resets the form, refreshes the table, and shows a toast.”

## 18. Likely interviewer follow-up questions and answers

### Q1. Difference between Server and Client Components?

A Server Component is server-side composition that is not shipped as interactive browser code. A Client Component, marked 'use client', can use state, effects, event handlers, and browser APIs. app/companies/page.tsx is the shell; CompaniesContent is the interactive child.

### Q2. What does hydration do?

Hydration attaches React behavior to the initial HTML so buttons, inputs, and effects work in the browser. Uny10 avoids common mismatches by reading localStorage/window/geolocation inside client effects rather than during initial server rendering.

### Q3. What is a Hook?

A Hook is a React function that gives a function component state, effects, context, memoization, or refs. Uny10 uses useState for company data, useEffect for loading, and useAuth for shared auth.

### Q4. Does Companies call GET /api/companies?

Not in its current main load/create path. It imports getAllCompanies and createCompany from the 'use server' file lib/actions/companies.ts. The explicit API route exists and other features such as Map call it through authFetch.

### Q5. Is a Server Action automatically secure?

No. It runs on the server but still needs validation and authorization. In this code, company actions receive teamId from client localStorage and do not call the common access gate themselves. They should derive the user from the secure session and verify membership.

### Q6. Why use useEffect for data loading?

The component needs to perform a side effect after it exists in the browser. useEffect is suitable for network calls and browser storage, and it supports cleanup for listeners or subscriptions.

### Q7. Why use useMemo?

It caches a derived calculation until dependencies change. Uny10 uses it for filters, contact options, map lists, and tab values. It is not a security feature or persistent state.

### Q8. Why does Map use dynamic with ssr:false?

Leaflet expects the DOM and browser globals. Server rendering has no DOM. Dynamic loading plus ssr:false defers it to the browser and provides a loading fallback.

### Q9. Why Context for auth but local state for filters?

Auth is cross-cutting and needed by many descendants, so Context avoids prop drilling. Search filters and form values belong only to the company screen, so local state keeps ownership close to the UI.

### Q10. What happens when a team is switched?

AppSidebar posts to /api/auth/switch-team through authFetch, stores the returned user, dispatches auth_user_updated, and calls router.refresh. AuthProvider listens and updates its Context, so all useAuth consumers see the new team.

### Q11. What would you improve first?

I would unify the company data path and authorization boundary, remove duplicate list fetching, add typed server validation, and add cross-team tests for Server Actions. I would keep browser-only libraries behind client boundaries.

## 19. Concepts and terminologies to remember

- **Next.js:** React framework adding routing, server execution, rendering, and deployment conventions.
- **App Router:** folder-based routing under app/.
- **Server Component:** server-rendered component without browser interactivity.
- **Client Component:** 'use client' component that can use state/effects/browser APIs.
- **Rendering:** producing UI from component code and data.
- **Hydration:** attaching client React behavior to initial HTML.
- **Hook:** React function for state, effects, context, memoization, or refs.
- **State:** changing data owned by a component/store.
- **Props:** inputs passed from parent to child.
- **Context:** shared descendant-readable state.
- **Effect:** post-render work such as fetching, listeners, timers, or external libraries.
- **Memoization:** reusing a calculated value when dependencies are unchanged.
- **Ref:** mutable value or DOM reference that does not itself trigger rendering.
- **Server Action:** Next.js server function callable from client code.
- **Route Handler/API route:** explicit HTTP method function under app/api/**/route.ts.
- **Client-side navigation:** route change without a full document reload.
- **Cookie:** browser-managed request value; refresh_token is HttpOnly.
- **Session:** server-side logged-in browser identity represented by a database-backed token.
- **Controlled input:** form value owned by React state.
- **Dynamic import/code splitting:** loading a module later to reduce initial work.
- **SSR:** Server-Side Rendering.
- **Imperative integration:** directly controlling a non-React object such as Leaflet inside an effect.
- **Source of truth:** authoritative state; for auth, server session rather than localStorage.

