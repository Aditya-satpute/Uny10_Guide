# Topic 6 — Companies and Follow-ups Feature Flow in Uny10

This note traces the real Companies feature through user actions.

The most important project-specific fact is that Uny10 has two company data paths:

1. The current visible company screens mainly call server-action-style functions from lib/actions/companies.ts.
2. Separate HTTP API routes exist under app/api/companies and use native MongoDB repositories from lib/db/companies.ts.

The function names look similar, but these are different implementations.

---

## 1. What the feature does

The Companies feature lets a user:

- View all companies.
- Search and filter companies.
- Create online or offline companies.
- Import companies from CSV.
- Open a company detail page.
- Edit company fields and status.
- Assign a sponsorship POC.
- Add, edit, and delete follow-ups.
- See company contacts, documents, location, and visit history.

A company is the parent CRM record. A follow-up is an activity stored inside that company.

---

## 2. Important files

~~~text
app/companies/page.tsx
components/companies/companies-content.tsx
components/companies/all-companies.tsx
components/companies/online-companies.tsx
components/companies/offline-companies.tsx
app/companies/[type]/[id]/page.tsx

lib/actions/companies.ts
lib/api-client.ts
lib/db/companies.ts
lib/models/company.ts

app/api/companies/route.ts
app/api/companies/[id]/route.ts
app/api/companies/[id]/followups/route.ts
~~~

Supporting code:

~~~text
lib/mongodb.ts
lib/db/mongodb.ts
lib/actions/users.ts
lib/team-scope.ts
lib/company-details.ts
~~~

---

## 3. The data shape

A CompanyData object contains:

~~~text
id
teamId
name
type
companyPOC[]
sponsorshipPOC[]
location
referenceURL
proposalLink
amount
status
mouLink
specialDeliverables[]
followUps[]
VisitsHistory
remarks
createdAt
updatedAt
~~~

Company type is:

~~~text
online | offline
~~~

Company status is:

~~~text
not_started | interested | finalized | rejected
~~~

A POC means Point of Contact. Each POC can contain:

~~~text
userId
name
phone
email
~~~

The database follow-up shape is:

~~~text
id
date
note
source
addedBy
~~~

The detail-page UI uses a different shape:

~~~text
id
date
poc
scheduledFollowUp
remarks
~~~

The function buildCompanyPayload converts the UI shape back to the database shape:

~~~ts
{
  id: fu.id,
  date: fu.scheduledFollowUp || fu.date,
  note: fu.remarks,
  source: 'manual',
  addedBy: fu.poc?.name || 'System',
}
~~~

This mapping is essential. The screen field remarks becomes the database field note.

---

## 4. Opening the Companies page

The page app/companies/page.tsx renders CompaniesContent.

~~~text
app/companies/page.tsx
  → CompaniesContent
      → AllCompanies
      → OnlineCompanies
      → OfflineCompanies
~~~

CompaniesContent is a client component. It controls tabs, form state, loading, CSV importing, and refresh signals.

### User action: open /companies

~~~text
User opens /companies
  → CompaniesPage renders CompaniesContent
  → component reads current team from localStorage
  → getAllCompanies(teamId)
  → Mongoose Company query
  → result normalized
  → React state updated
  → company list rendered
~~~

---

## 5. Loading companies in the current UI

The current company components import from:

~~~ts
import {
  createCompany,
  getAllCompanies,
  CompanyData
} from '@/lib/actions/companies'
~~~

This is important: the visible current company UI does not necessarily call GET /api/companies. It calls server-action-style functions.

### Step-by-step

1. The component reads auth_user from localStorage.
2. It extracts teamId.
3. It calls getAllCompanies(teamId).
4. lib/actions/companies.ts calls the Mongoose connection helper.
5. It builds a team query.
6. It queries the Company model.
7. It uses lean() to receive plain objects.
8. It maps results through toPlainCompany.
9. The component calls setCompanies.
10. The list renders the records.

The server action uses a query like:

~~~ts
const query = !teamId || teamId === 'default-team'
  ? {
      $or: [
        { teamId: 'default-team' },
        { teamId: { $exists: false } }
      ]
    }
  : { teamId }

const companies = await Company.find(query).lean().exec()
~~~

The default-team behavior includes older documents that do not have a teamId.

toPlainCompany normalizes:

- _id into string id.
- Missing arrays into empty arrays.
- Coordinates into numbers.
- Dates into display strings.
- Follow-ups into plain objects.

---

## 6. Search and filtering

AllCompanies keeps searchQuery, statusFilter, and typeFilter in React state.

Filtering happens in the browser:

~~~text
company name contains search
OR company POC name contains search
OR sponsorship POC name contains search

AND status matches
AND type matches
~~~

Why this is useful:

- No request is needed for every keystroke.
- Filtering feels immediate.
- The component already has the list.

Trade-off:

- Large datasets should use server-side search and pagination.
- The browser receives all accessible companies.
- Large arrays can make client filtering slow.

---

## 7. Opening a company detail page

AllCompanies handles a row click:

~~~ts
router.push(
  '/companies/' + company.type + '/' + company.id +
  '?name=' + encodeURIComponent(company.name)
)
~~~

The target is:

~~~text
app/companies/[type]/[id]/page.tsx
~~~

### Flow

~~~text
User clicks a company
  → AllCompanies.handleRowClick
  → router.push('/companies/{type}/{id}')
  → dynamic CompanyDetailPage renders
  → useParams reads type and id
  → getCompanyById(id)
  → database company is loaded
  → database shape is mapped to CompanyDetail UI shape
  → setCompany(...)
  → detail page displays company and follow-ups
~~~

### Static/fallback initial state

The detail page initially calls:

~~~ts
getCompanyDetail(type, id) ?? createFallbackCompany(type, id, companyName)
~~~

getCompanyDetail comes from lib/company-details.ts and contains static example records.

Then a useEffect loads the real database record. If found, it replaces the initial state.

This means the first render can contain a static or fallback object before the database result arrives.

### Loading real data

The effect:

1. Reads auth_user.
2. Gets the team ID.
3. Loads users with getUsers(teamId).
4. Creates maps from user names to user records and avatars.
5. Calls getCompanyById(id).
6. Converts the database company into CompanyDetail.
7. Converts VisitsHistory into a display array.
8. Updates company and visitsHistory state.

---

## 8. Important security observation about detail loading

The detail page calls:

~~~ts
const dbCompany = await getCompanyById(id)
~~~

It does not pass teamId.

The server-action implementation of getCompanyById only applies a team filter when teamId is supplied and is not default-team.

The separate API route does more:

~~~ts
const teamId = resolveTeamIdFromRequest(request)
const accessResponse = await enforceTeamAccess(request, teamId)
if (accessResponse) return accessResponse

const company = await getCompanyById(id, teamId)
~~~

The API route is therefore a stronger protected pattern.

In a multi-team application, the server action should derive the authenticated user and team on the server and always enforce team access before returning a company.

---

## 9. User action: create an online or offline company

### Online flow

~~~text
User clicks Add Company
  → dialog opens
  → form values stored in newCompany state
  → user submits
  → handleAddCompany builds CompanyData
  → lib/actions/companies.ts createCompany
  → fields and coordinates normalized
  → Mongoose Company.save()
  → user statistics recalculated
  → created company returned
  → setCompanies appends it
  → dialog closes
~~~

### Offline flow

The same flow is used, but the object includes:

~~~text
type: offline
location.address
location.city
~~~

### Form state

The components keep values such as:

~~~text
name
companyPOC
sponsorshipPOC
referenceURL or location
amount
status
specialDeliverables
remarks
~~~

Typing changes React state only. Nothing is stored until the user submits.

### Server-side createCompany

The function in lib/actions/companies.ts:

1. Connects through lib/mongodb.ts.
2. Calls withDerivedCoordinates.
3. Adds teamId when available.
4. Creates a Mongoose Company instance.
5. Calls save().
6. Calls syncAllUserStatsFromCompanies().
7. Converts the saved document with toPlainCompany.
8. Returns the result.

### Why withDerivedCoordinates exists

It tries to use:

- Existing coordinates.
- Map/reference URLs.
- Expanded shortened map URLs.
- Latitude/longitude patterns in text.

For offline records it also normalizes the address into location.

This lets the map feature use company locations without requiring manual coordinate entry.

### Client update

After success:

~~~ts
setCompanies(previous => [...previous, createdCompany])
setIsAddDialogOpen(false)
~~~

The component uses the returned database record, so the new ID and normalized fields are immediately available.

---

## 10. User action: create from the main CompaniesContent form

CompaniesContent has another create flow.

The handler:

1. Checks required fields in the browser.
2. Trims POC values.
3. Removes incomplete company POC entries.
4. Builds a CompanyData object.
5. Calls createCompany.
6. Appends the returned company.
7. Increments companiesRefreshSignal.
8. Resets the form.
9. Shows a success toast.

The browser checks are for user experience. They are not a security boundary. The server must still validate input.

---

## 11. User action: CSV import

### Flow diagram

~~~text
Select CSV
  → File.text()
  → parseCsv()
  → normalizeCsvRow()
  → valid row becomes CompanyData
  → createCompany(row)
  → MongoDB insert
  → repeat for each row
  → local state appends imported rows
  → toast reports imported/skipped rows
~~~

### Step-by-step

1. The user selects a file.
2. The component stores it in pendingCsvFile.
3. The user confirms the upload.
4. The browser reads the file text.
5. parseCsv handles rows, headers, quoted commas, and values.
6. normalizeCsvRow supports header aliases.
7. It converts amount to a number.
8. It converts type to online/offline.
9. It converts status to an allowed status.
10. It converts specialDeliverables into an array.
11. It creates a location object for offline rows.
12. Rows without name or type are skipped.
13. Each valid row calls createCompany.
14. Successful results are appended to local state.

The loop awaits each row sequentially. This is simple but can be slow for large files. Bulk insertion or controlled concurrency could improve it.

Possible partial-import issue: if early rows succeed and a later row fails, the earlier rows remain saved.

---

## 12. User action: change company status

OnlineCompanies and OfflineCompanies use updateCompanyDetails.

~~~ts
const updated = await updateCompanyDetails(companyId, {
  status: newStatus
})
~~~

### Flow

~~~text
User changes status
  → handleStatusChange
  → updateCompanyDetails
  → Mongoose connection
  → find existing company
  → merge update and location
  → derive coordinates if needed
  → Company.findByIdAndUpdate
  → synchronize user statistics
  → return updated company
  → replace matching item in React state
  → status badge changes
~~~

The function loads the existing company first, merges values, removes _id/id from the update payload, and saves with new: true.

Why load first? It preserves existing fields and avoids accidentally replacing the company with an incomplete partial object.

---

## 13. User action: edit or assign a sponsorship POC

AllCompanies loads users and keeps only:

~~~text
isActive === true
approvalStatus === 'approved'
~~~

When a row edit is saved, it sends:

~~~ts
{
  type: rowEditType,
  status: rowEditStatus,
  sponsorshipPOC: [{
    userId,
    name,
    phone,
    email
  }]
}
~~~

The detail page has a similar assignment modal.

### Detail assignment flow

~~~text
User opens POC modal
  → approved active subcoordinators are shown
  → user selects member
  → local assignedTo changes immediately
  → handleSaveChanges saves company
  → success keeps state
  → failure restores previous assignedTo
~~~

This is optimistic UI with rollback.

---

## 14. User action: delete a company

### Current visible list flow

~~~text
User chooses Delete
  → delete confirmation/action
  → deleteCompany(companyId)
  → Mongoose Company.findByIdAndDelete
  → synchronize user statistics
  → filter deleted ID from local state
  → row disappears
~~~

The current server-action delete function accepts only the company ID and uses findByIdAndDelete.

### API route alternative

The API route supports:

~~~text
DELETE /api/companies/:id
  → app/api/companies/[id]/route.ts
  → enforceTeamAccess
  → lib/db/companies.ts deleteCompany(id, teamId)
  → native deleteOne with ID and team filter
  → success response
~~~

The API route is more visibly team-scoped.

### Important security concern

In a multi-team application, delete/update functions should derive the authenticated team server-side and include it in the database filter. A plain findByIdAndDelete(companyId) can be unsafe if the caller can reach it without a trusted authorization boundary.

---

## 15. Follow-up UI shape and modal

The detail page stores:

~~~ts
const [followUpDraft, setFollowUpDraft] = useState({
  pocId: '',
  scheduledFollowUp: '',
  remarks: ''
})
~~~

The Follow Ups section renders each item with:

- Recorded date.
- Next scheduled date.
- Assigned POC.
- Remarks.
- Edit button.
- Delete button.

Clicking Add Follow Up calls openFollowUpModal().

Clicking Edit calls openFollowUpModal(followUp.id), which copies the existing item into the draft state.

---

## 16. User action: add a follow-up in the current detail screen

### Actual current flow

~~~text
Click Add Follow Up
  → openFollowUpModal()
  → user selects POC and date
  → user enters remarks
  → Save calls saveFollowUpFromModal()
  → browser validates POC and date
  → new follow-up is added to local state
  → buildCompanyPayload converts fields
  → updateCompanyDetails saves full company
  → success toast or rollback
~~~

### Step 1: browser validation

The code requires:

~~~ts
const selectedMember = subCoordinatorMembers.find(
  member => member.id === followUpDraft.pocId
)

if (!selectedMember) {
  toast.error('Please select a sub-coordinator')
  return
}

if (!followUpDraft.scheduledFollowUp) {
  toast.error('Please select the next follow-up date')
  return
}
~~~

### Step 2: create local item

For a new item:

~~~ts
{
  id: 'f-' + Date.now(),
  date: todayIso,
  poc: selectedMember,
  scheduledFollowUp: followUpDraft.scheduledFollowUp,
  remarks: followUpDraft.remarks
}
~~~

The browser generates the ID.

### Step 3: optimistic local update

~~~ts
setCompany(nextCompany)
~~~

The follow-up appears immediately.

### Step 4: map to database format

handleSaveChanges calls buildCompanyPayload. The UI object becomes:

~~~ts
{
  id,
  date: scheduledFollowUp || date,
  note: remarks,
  source: 'manual',
  addedBy: poc.name
}
~~~

### Step 5: save complete company

The detail page calls:

~~~ts
await updateCompanyDetails(id, dataToSave)
~~~

This is the server-action-style Mongoose function. The current detail page does not normally call the dedicated follow-up POST endpoint.

### Step 6: success or rollback

On success:

- Toast says follow-up added.
- Modal closes.
- Draft is cleared.

On failure:

~~~ts
setCompany(previousState)
toast.error('Failed to save follow-up')
~~~

The optimistic change is rolled back.

---

## 17. User action: edit a follow-up

### Flow

~~~text
Click edit icon
  → openFollowUpModal(followUp.id)
  → existing item copied to draft
  → user changes date/POC/remarks
  → saveFollowUpFromModal()
  → matching local array item is replaced
  → complete company payload is rebuilt
  → updateCompanyDetails saves company
  → success keeps update or failure restores old array
~~~

The code finds the item:

~~~ts
const followUp = company.followUps.find(
  item => item.id === followUpId
)
~~~

Then it initializes the draft:

~~~ts
{
  pocId: followUp.poc?.id || '',
  scheduledFollowUp: followUp.scheduledFollowUp || followUp.date || '',
  remarks: followUp.remarks || ''
}
~~~

The UI changes only one item locally, but persistence sends the complete followUps array.

---

## 18. User action: delete a follow-up

### Current detail-page flow

~~~text
Click trash icon
  → save previous followUps
  → filter selected item out locally
  → handleSaveChanges(false, nextCompany)
  → save complete company payload
  → success toast
  → failure restores previous array
~~~

The local deletion is:

~~~ts
followUps: company.followUps.filter(
  item => item.id !== followUp.id
)
~~~

### Dedicated API alternative

The API route supports:

~~~text
DELETE /api/companies/:id/followups?followUpId=followup-1
~~~

It validates the session/team and the repository uses MongoDB $pull to remove only that embedded follow-up.

---

## 19. The dedicated follow-up API path

The file is:

~~~text
app/api/companies/[id]/followups/route.ts
~~~

It contains POST, PATCH, and DELETE.

### POST add

1. Read company id from params.
2. Parse JSON.
3. Resolve team ID.
4. Call enforceTeamAccess.
5. Require body.id, body.date, and body.note.
6. Call lib/db/companies.ts addFollowUp.
7. Repository finds the company with ID and team filter.
8. MongoDB uses $push.
9. Return updated company with HTTP 201.

### PATCH update

Body:

~~~json
{
  "followUpId": "followup-1",
  "updates": {
    "date": "2026-10-25",
    "note": "Updated note"
  }
}
~~~

The repository matches followUps.id and uses arrayFilters to target the embedded item.

### DELETE remove

1. Read company id from params.
2. Read followUpId from query parameters.
3. Resolve team and enforce access.
4. Use $pull to remove the matching item.
5. Return success or 404.

---

## 20. Native follow-up repository operations

lib/db/companies.ts implements the targeted API operations.

### Add with $push

~~~ts
$push: { followUps: followUp }
~~~

This appends an item to the embedded array.

### Update with arrayFilters

~~~ts
$set: {
  'followUps.$[elem]': {
    ...updateData
  }
},
arrayFilters: [
  { 'elem.id': followUpId }
]
~~~

arrayFilters tells MongoDB which array element should be changed.

### Delete with $pull

~~~ts
$pull: {
  followUps: { id: followUpId }
}
~~~

This removes matching embedded items.

All operations also include buildTeamFilter(teamId).

---

## 21. Whole-company save versus targeted follow-up update

### Current UI: whole-company save

~~~text
updateCompanyDetails(companyId, completeCompanyPayload)
~~~

Benefits:

- Simple detail-page implementation.
- One save function handles all company fields.
- The returned company represents complete state.
- Easy optimistic rollback.

Risks:

- Sends more data than needed.
- A stale screen may overwrite another user's newer follow-up.
- Authorization must be carefully enforced inside the server action.
- A partial UI mapping can accidentally replace missing fields.

### Dedicated API: targeted operation

~~~text
addFollowUp(companyId, followUp, teamId)
updateFollowUp(companyId, followUpId, updates, teamId)
deleteFollowUp(companyId, followUpId, teamId)
~~~

Benefits:

- Smaller update.
- Better concurrent editing behavior.
- Explicit team-scoped operation.
- Easier to audit.

Risks:

- More client/API code.
- The frontend must merge returned data correctly.
- All three operations need consistent validation.

---

## 22. Other features that consume follow-ups

### Calendar

calendar-content.tsx reads company.followUps, filters valid dates, and turns each follow-up into a calendar event.

~~~text
Company.followUps
  → valid dates
  → calendar event objects
  → combined with meetings, visits, and calendar records
  → calendar UI
~~~

### Profile notifications

profile-content.tsx reads company follow-ups and creates notification-like items from:

- Company name.
- Follow-up date.
- Follow-up note.
- Company ID.
- Follow-up ID.

Therefore, changing the follow-up field mapping affects the detail page, calendar, profile notifications, CSV export, and API data.

### Visit history

The company repository also supports addCompanyVisitHistory. It uses $addToSet to avoid duplicate visitor names for the same company/date.

The detail page converts the object/map shape into display records:

~~~text
VisitsHistory object
  → [{ date, visitors[] }]
  → sorted visit history UI
~~~

---

## 23. Error handling and optimistic rollback

### List pages

List components use try/catch. On failure they set an error message such as:

~~~text
Failed to load companies
Failed to add company
Failed to update status
Failed to delete company
~~~

### Detail page

The detail page often uses this pattern:

1. Save previous state.
2. Update local state immediately.
3. Await the server save.
4. Keep the new state on success.
5. Restore previous state on failure.
6. Show a toast.

This makes the UI feel fast.

Possible problem: two saves close together can produce race conditions. A later failed request could restore older state over a newer successful change. A request queue or version check would improve this.

---

## 24. Security and correctness issues

### Current API route protection

The dedicated API path uses:

~~~text
resolveTeamIdFromRequest
  → enforceTeamAccess
  → team-scoped repository query
~~~

This is the preferred protected pattern.

### Current server-action concern

The visible company screens use functions such as getCompanyById, updateCompanyDetails, and deleteCompany. Some accept only an ID and do not visibly derive the session/team inside the function.

For a multi-team application, the server action should:

1. Read the server-side session.
2. Determine the authenticated user.
3. Determine the authorized active team.
4. Check membership.
5. Include teamId in the database filter.

### Browser validation is not authorization

A browser can bypass required-field checks, change POC IDs, or call a different API. Server-side validation and authorization are required.

### Stable IDs are safer than names

The detail page maps users partly by name. Duplicate names or renamed users can cause wrong avatar/POC mapping. Stable user IDs should be the primary relationship.

---

## 25. Important edge cases

- Empty or incomplete company fields.
- Incomplete POC rows.
- Offline company with missing location.
- Invalid or shortened reference URL.
- Duplicate company names.
- Duplicate client-generated follow-up IDs.
- Concurrent detail-page edits.
- Old companies without teamId.
- Missing followUps array.
- Partial CSV imports.
- Failed coordinate lookup.
- A POC user no longer approved or active.
- Follow-up dates that cannot be parsed by the calendar.
- Stale company state overwriting a newer database update.

---

## 26. How to explain this feature to an interviewer

### Simple answer

> “The Companies feature is a team-scoped CRM module. The list page loads company records, filters them by type/status/search, and lets users create or update companies. In the current UI, the components mainly call server-action-style functions from lib/actions/companies.ts. Those functions use Mongoose, normalize company data, save the Company document, recalculate related user statistics, and return plain objects for React state. A company contains embedded contacts and follow-ups. On the detail page, follow-ups are edited optimistically in local state and then mapped back to the database shape. The project also has dedicated protected API routes that use native MongoDB repositories and targeted $push, arrayFilters, and $pull updates for follow-ups. I would improve consistency by standardizing the UI on the server-authorized targeted API path or enforcing session/team checks inside every server action.”

### Humanized answer

> “I think of a company as the parent record and follow-ups as a small activity timeline inside it. The list screen loads and filters companies. The detail screen converts the database record into a UI-friendly object. When I add a follow-up, the UI updates immediately, then converts it back into the database format and saves it. If saving fails, the old state is restored. The codebase also contains smaller API operations for changing only one follow-up, which would be better for concurrent edits.”

---

## 27. Likely interviewer questions and answers

### Q1. What does the Companies feature manage?

Team-scoped sponsorship/CRM companies, contacts, statuses, locations, POCs, documents, and follow-up activities.

### Q2. Where does the page start?

app/companies/page.tsx renders CompaniesContent, which controls the company UI.

### Q3. How are companies loaded in the current UI?

The visible company components import getAllCompanies or getCompaniesByType from lib/actions/companies.ts. These functions use Mongoose and return normalized plain objects.

### Q4. Does the current UI always call GET /api/companies?

No. The current components often call server-action-style functions. The API route exists, but the real path depends on the import.

### Q5. How is search implemented?

The component filters the already-loaded local array by company name and POC names, with status/type filters.

### Q6. How is a company created?

A form builds CompanyData, calls createCompany, the server normalizes coordinates and fields, saves a Mongoose Company, synchronizes statistics, and returns the saved object. The component appends it to state.

### Q7. How are online and offline companies different?

Online companies use referenceURL. Offline companies use location.address and location.city. Both use the same Company model and status values.

### Q8. How does CSV import work?

The browser parses and normalizes the file, skips invalid rows, calls createCompany for each valid row, and appends successful results.

### Q9. How are follow-ups stored?

As an embedded followUps array inside the company document.

### Q10. Why embed follow-ups?

The company and its activity timeline can be read together. The trade-off is document growth and more complex array updates.

### Q11. How does the current detail screen add a follow-up?

It validates the selected approved subcoordinator/date, updates local state optimistically, maps the UI object to id/date/note/source/addedBy, and saves the complete company through updateCompanyDetails.

### Q12. Does the current detail screen use the dedicated follow-up POST endpoint?

Not normally. The dedicated endpoint exists, but the current screen saves the complete company through the server-action-style function.

### Q13. How does the API update one follow-up?

It receives company ID from params and followUpId/updates from the request, checks team access, and uses arrayFilters with $set.

### Q14. How does the API delete one follow-up?

It receives company ID and followUpId, checks access, and uses $pull.

### Q15. What is optimistic UI?

The UI changes before the server confirms success. If the server fails, the previous state is restored.

### Q16. What risk exists with whole-company saves?

A stale screen can overwrite another user's newer changes. Targeted follow-up mutations reduce this risk.

### Q17. How is tenant isolation supposed to work?

The server validates session/team access and repositories include teamId in their query filters. Client-provided team IDs are not sufficient by themselves.

### Q18. What security concern exists in the current server-action path?

Some functions receive only a company ID and do not visibly enforce the current session/team inside the function. They should use server-side identity and team filters.

### Q19. Why are company statistics recalculated?

Company assignment and finalized status affect user/team metrics, so syncAllUserStatsFromCompanies recalculates derived values.

### Q20. How would you improve the feature?

Standardize on one protected path, use runtime schemas, enforce authorization inside server actions, use targeted follow-up mutations, generate IDs on the server, add uniqueness rules, use pagination, and protect concurrent updates.

---

## 28. Important things to remember

1. A company is the parent CRM record.
2. Follow-ups are embedded inside the company.
3. The visible UI mainly uses lib/actions/companies.ts.
4. API routes also exist under app/api/companies.
5. lib/db/companies.ts is the native repository path.
6. lib/models/company.ts is the Mongoose schema path.
7. Search/filtering is mostly client-side.
8. Company creation normalizes fields and coordinates.
9. Online uses referenceURL; offline uses location.
10. Detail page initially has static/fallback data, then loads the database record.
11. Database and UI company shapes are not identical.
12. buildCompanyPayload performs the reverse mapping.
13. Current follow-up add/edit/delete saves the complete company.
14. Dedicated API routes support targeted follow-up operations.
15. $push adds, arrayFilters/$set updates, and $pull deletes embedded follow-ups.
16. API routes use explicit session/team access checks.
17. Server-action update/delete paths need consistent authorization review.
18. Optimistic UI requires rollback.
19. Follow-ups also feed calendar and profile notification views.
20. Stable IDs are safer than matching users only by names.
21. Whole-company saves are simple but can overwrite concurrent changes.
22. Team filters are essential for tenant isolation.
23. Browser validation is not security.
24. The interview flow is: user action → component state → server action/API → authorization → Company model/repository → MongoDB → normalized response → UI.

---

## 29. Final flow summary

~~~text
Open Companies
  → CompaniesContent/list component
  → getAllCompanies or getCompaniesByType
  → Mongoose Company.find
  → toPlainCompany
  → React state
  → filter and render

Create company
  → form state
  → CompanyData
  → createCompany
  → coordinate/field normalization
  → Company.save()
  → statistics synchronization
  → returned company
  → UI state

Open detail
  → router.push with type/id
  → dynamic detail page
  → getCompanyById
  → database shape mapped to UI shape
  → company and follow-ups rendered

Add/edit/delete follow-up in current UI
  → modal/button
  → optimistic state
  → buildCompanyPayload
  → updateCompanyDetails saves full company
  → success or rollback

Dedicated API alternative
  → POST/PATCH/DELETE follow-up route
  → enforceTeamAccess
  → native repository
  → $push / arrayFilters / $pull
  → team-scoped updated company
~~~

The simplest interview sentence is:

> “Uny10 stores follow-ups as embedded activities inside a team-scoped Company document. The current UI uses server-action-style Mongoose functions and optimistic state, while dedicated API routes provide targeted native-MongoDB operations. The main improvement would be making authorization and data access consistent across both paths.”

