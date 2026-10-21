# Topic 4 — MongoDB Schemas, Repositories, Indexes, and Connection Strategy in Uny10

This note explains how Uny10 stores, reads, updates, and protects data in MongoDB.

The simplest picture is:

~~~text
API route or server action
        ↓
database helper/repository
        ↓
Mongoose model OR native MongoDB collection
        ↓
cached MongoDB connection
        ↓
MongoDB document
~~~

Uny10 does not use only one database style. Some parts use Mongoose models, and several feature repositories use the native MongoDB driver. Understanding that split is essential.

---

## 1. MongoDB basics

### What is a database?

A database stores application data so it can survive page refreshes, server restarts, and different users.

Uny10 stores data such as:

- Users
- Teams
- Team memberships
- Companies
- Meetings
- Calendar entries
- Visit plans
- Notifications
- Support requests

### What is MongoDB?

MongoDB is a document database. It stores JSON-like objects called documents.

A simplified company document might look like:

~~~json
{
  "_id": "MongoDB-generated-id",
  "teamId": "team-123",
  "name": "Example Company",
  "type": "offline",
  "amount": 50000,
  "status": "interested",
  "companyPOC": [
    {
      "name": "Asha",
      "phone": "9999999999",
      "email": "asha@example.com"
    }
  ],
  "followUps": []
}
~~~

### MongoDB vocabulary

| Term | Simple meaning | Uny10 example |
|---|---|---|
| Database | Container holding application data | omnispons |
| Collection | Group of similar documents | companies, Users, Meetings |
| Document | One stored object | One company or one meeting |
| Field | A property inside a document | teamId, name, status |
| Query | Instructions for finding documents | find companies for teamId |
| Index | Extra lookup structure for faster queries | index on teamId and name |
| ObjectId | MongoDB's usual generated identifier | document _id |

MongoDB is flexible, but Uny10 still defines expected structures through TypeScript interfaces and Mongoose schemas.

---

## 2. What is a schema?

A schema is a description of what a document should look like.

A schema answers questions such as:

- Which fields exist?
- Which fields are required?
- Which values are allowed?
- What are the default values?
- Which fields are indexed?
- Which fields are nested inside other fields?

Why schemas are needed:

1. They give developers a predictable data shape.
2. They catch invalid values before they reach the database.
3. They document the data model.
4. They provide defaults and type conversion.
5. They make the code easier to maintain.

Important distinction:

- MongoDB itself is flexible and does not require every document to have exactly the same structure.
- A Mongoose schema gives application code validation and structure.
- Native MongoDB repositories use TypeScript interfaces and manual logic, so their runtime enforcement is weaker unless the code validates input separately.

---

## 3. Two database styles in Uny10

### Style 1: Mongoose models

Mongoose is a library that sits on top of MongoDB.

It provides:

- Schema definitions
- Validation
- Models
- Query methods such as find and findOneAndUpdate
- Defaults
- Middleware such as pre-save hooks
- Index declarations

The files are:

~~~text
lib/models/user.ts
lib/models/company.ts
lib/models/meeting.ts
lib/models/calendar.ts
lib/models/visitplan.ts
lib/models/notification.ts
~~~

Example:

~~~ts
const CompanySchema = new mongoose.Schema({
  teamId: {
    type: String,
    default: 'default-team',
    index: true,
  },
  name: {
    type: String,
    required: true,
  },
})
~~~

Then the schema becomes a model:

~~~ts
export const Company =
  mongoose.models.Company || mongoose.model('Company', CompanySchema)
~~~

The model is used like:

~~~ts
const company = new Company(payload)
await company.save()
~~~

### Style 2: native MongoDB repositories

The native MongoDB driver is used directly in several files under:

~~~text
lib/db/
~~~

Examples:

~~~text
lib/db/companies.ts
lib/db/meetings.ts
lib/db/calendar.ts
lib/db/teams.ts
lib/db/support.ts
lib/db/team-members.ts
~~~

A repository gets a collection directly:

~~~ts
const { db } = await connectToDatabase()
const collection = db.collection('companies')
~~~

Then it performs operations such as:

~~~ts
collection.find(filter).toArray()
collection.insertOne(document)
collection.findOneAndUpdate(filter, update, options)
collection.deleteOne(filter)
~~~

### Why this split matters

Uny10 has a mixed persistence architecture:

~~~text
Mongoose path:
server action or model-based code
  → Mongoose connection
  → Mongoose schema/model
  → MongoDB

Native repository path:
API route
  → native MongoClient connection
  → collection
  → MongoDB
~~~

This may be the result of feature history, gradual migration, or using the most convenient tool for each feature.

The trade-off is that two paths can behave differently:

- Validation may differ.
- Serialization may differ.
- Index creation may happen in different places.
- Team filtering may be implemented differently.
- Error behavior may differ.
- Updates may use different return-value conventions.

---

## 4. The main data model

The main entities are:

~~~text
Users
Teams
TeamMembers
companies
meetings
calendar
VisitPlans
Notifications
Support
~~~

There are two kinds of relationships.

### Embedded data

Embedded data is stored inside the parent document.

Uny10 examples:

- Company contacts inside a company.
- Company followUps inside a company.
- Meeting attendees and RSVP responses inside a meeting.
- Calendar eventSchedule inside a calendar entry.
- VisitPlan assignedCompanies inside a visit plan.
- Team members also exist as an embedded members array inside a Team document.

Example:

~~~text
Company
  ├─ companyPOC[]
  ├─ sponsorshipPOC[]
  └─ followUps[]
~~~

Benefit: reading the parent often returns the related data in one query.

Trade-off: the parent document can become large, and updating one embedded item may be more complex.

### Referenced-by-ID data

Referenced data stores only an ID pointing to another entity.

Uny10 examples:

- Meeting companyId points to a company.
- Calendar meetingIds point to meetings.
- Calendar visitIds point to visit plans.
- TeamMember userId points to a user.
- Team coordinator.userId points to a user.

Benefit: data remains separated and can be updated independently.

Trade-off: the application may need multiple queries and must maintain referential consistency itself.

---

## 5. User schema: lib/models/user.ts

The User schema stores identity, team information, roles, status, statistics, and the application session.

Important groups of fields:

### Identity fields

~~~ts
googleId
email
name
username
avatar
avatarUrl
~~~

Email is required, trimmed, lowercased, and unique.

Google ID is unique and sparse. Sparse means documents without a Google ID do not all conflict with one another.

### Team fields

~~~ts
teamIds
teamNames
teamId
activeTeamId
teamName
~~~

There are newer multi-team fields and legacy fields. This suggests the project has evolved from an older team representation to a newer one.

### Role and status

~~~ts
role: 'coordinator' | 'subcoordinator' | 'admin'
status: 'pending' | 'active' | 'rejected'
approvalStatus: 'pending' | 'approved' | 'rejected'
isActive: boolean
~~~

These fields represent different ideas:

- role: what kind of user this is.
- status: broad account access state.
- approvalStatus: approval workflow state.
- isActive: whether the record is currently active.

### Session fields

~~~ts
refreshToken
refreshTokenExpires
~~~

Uny10 stores the opaque session token and expiry in the user document.

### Statistics

~~~ts
totalVisits
companiesListed
totalAmountFinalized
companiesFinalized
~~~

These are denormalized values: information derived from other records but stored on the user for quick display.

### Mongoose indexes on User

~~~ts
UserSchema.index({ googleId: 1 }, { unique: true, sparse: true })
UserSchema.index({ email: 1 }, { unique: true })
UserSchema.index({ teamIds: 1 })
UserSchema.index({ teamId: 1, status: 1 })
UserSchema.index({ activeTeamId: 1 })
UserSchema.index({ teamId: 1, role: 1 })
UserSchema.index({ teamName: 1, designation: 1 })
UserSchema.index({ teamId: 1, designation: 1 })
~~~

The indexes support common lookup patterns: identity lookup, team filtering, status filtering, and role filtering.

---

## 6. Company schema: lib/models/company.ts

The Company model represents a sponsorship/CRM company.

Important fields:

~~~ts
teamId
name
type
companyPOC
sponsorshipPOC
location
referenceURL
proposalLink
amount
status
mouLink
specialDeliverables
followUps
VisitsHistory
remarks
~~~

The model uses nested schemas:

### ContactSchema

A company contact contains:

~~~ts
userId?
name
phone
email
~~~

The userId is optional because a contact may not be a registered Uny10 user.

### FollowUpSchema

A follow-up contains:

~~~ts
id
date
note
source
addedBy
~~~

This is embedded inside the company document as followUps.

### LocationSchema

A location contains:

~~~ts
address
city
coordinates.lat
coordinates.lng
~~~

### Status enum

Company status is limited to:

~~~text
not_started
interested
finalized
rejected
~~~

This prevents arbitrary status values when the Mongoose model is used.

### Team index

~~~ts
teamId: {
  type: String,
  default: 'default-team',
  index: true,
}
~~~

The team index helps isolate and retrieve companies by tenant/team.

---

## 7. Meeting schema: lib/models/meeting.ts

A Meeting document stores the meeting, participants, approval flow, RSVP data, and outcome.

### Core fields

~~~text
title
type
date
time
duration
meetingLink
venue
agenda
createdById
createdByName
createdByRole
~~~

Online meetings use meetingLink. Offline meetings use venue.

### Embedded attendees

AttendeeSchema stores:

~~~text
name
avatar
initials
~~~

### Embedded RSVP responses

RSVPResponseSchema stores:

~~~text
userId
userName
status: yes | no | maybe
respondedAt
~~~

This allows one meeting document to contain participant responses.

### Embedded approval request

ApprovalRequestSchema stores:

~~~text
requestedAt
approvedAt
approvedBy
rejectedAt
rejectionReason
~~~

This supports subcoordinator requests that need coordinator review.

### State fields

~~~text
rsvpStatus: scheduled | pending | cancelled | visited
approvalStatus: approved | pending | rejected
requestType: create | edit | delete
~~~

These fields form a small state machine for meeting workflow.

### References

~~~text
companyId
companyName
~~~

The meeting can be connected to a company using its ID and a copied name.

### Meeting indexes

~~~ts
MeetingSchema.index({ createdById: 1 })
MeetingSchema.index({ date: 1, time: 1 })
MeetingSchema.index({ approvalStatus: 1, requestType: 1 })
MeetingSchema.index({ rsvpStatus: 1 })
MeetingSchema.index({ createdAt: -1 })
~~~

These indexes match likely queries: creator history, chronological lists, pending approvals, status filters, and recent records.

---

## 8. Calendar schema: lib/models/calendar.ts

Calendar stores a user's schedule for a date.

Fields include:

~~~text
date
userId
userName
note
eventSchedule[]
visitIds[]
meetingIds[]
isActive
createdAt
updatedAt
~~~

### EventScheduleSchema

Each scheduled event contains:

~~~text
title
time
details
createdAt
updatedAt
~~~

The schema uses {_id: false} for event schedule items. This means Mongoose does not create a separate MongoDB ObjectId for each small embedded event.

### Soft delete

Calendar entries have:

~~~ts
isActive: {
  type: Boolean,
  default: true,
  index: true,
}
~~~

Instead of deleting a document, the repository sets isActive to false. This is called a soft delete.

Benefit: historical data can remain available and accidental deletion is easier to recover from.

Trade-off: every read must remember to filter isActive: true, otherwise deleted data can appear.

### Calendar compound index

~~~ts
CalendarSchema.index({ userId: 1, date: 1 })
~~~

This supports the common query: “find this user's calendar entry for this date.”

---

## 9. VisitPlan schema: lib/models/visitplan.ts

A VisitPlan contains a user's planned company visits for a date and team.

Fields include:

~~~text
date
userId
userName
teamId
festId
assignedCompanies[]
totalAssignedCompanies
lastUpdatedBy
~~~

### Embedded assigned companies

Each assigned company contains:

~~~text
companyId
companyName
order
status: assigned | visited | skipped | rescheduled
note
~~~

This stores the visit route/order inside one plan document.

### Pre-save hook

The model contains:

~~~ts
VisitPlanSchema.pre('save', function (next) {
  this.totalAssignedCompanies =
    Array.isArray(this.assignedCompanies)
      ? this.assignedCompanies.length
      : 0
  next()
})
~~~

A **pre-save hook** is code Mongoose runs before saving a document.

Why it exists: it keeps totalAssignedCompanies synchronized with the actual array length when save() is used.

Important limitation: this hook does not necessarily run for every update method, such as direct updateOne/findOneAndUpdate operations. A field like this can become stale if updates bypass save.

### Unique compound index

~~~ts
VisitPlanSchema.index(
  { date: 1, userId: 1, teamId: 1 },
  { unique: true }
)
~~~

This means one user should have at most one visit plan per date within a team.

The database enforces this even if two requests arrive at almost the same time.

---

## 10. Notification schema

lib/models/notification.ts stores:

~~~text
userId
title
subtitle
type
timeLabel
read
metadata
createdAt
updatedAt
~~~

The userId field is indexed because notifications are commonly loaded for one user.

The notification route uses the session user ID rather than trusting an arbitrary requested user ID when reading or updating notifications. That is an important security/data-ownership pattern.

---

## 11. Team and TeamMember data

Uny10 represents team membership in two related places.

### Teams collection

lib/db/teams.ts uses a native MongoDB Team interface. A Team contains:

~~~text
teamId
teamName
teamNameNormalized
coordinator
members[]
joinCode
createdAt
updatedAt
~~~

The members array is embedded inside the Team document.

### TeamMembers collection

lib/db/team-members.ts stores separate membership records:

~~~text
userId
teamId
teamName
role
status
approvalStatus
designation
isActive
statistics
createdAt
updatedAt
~~~

This separate collection is useful for querying membership by team and user without loading the entire team document.

### Why both representations exist

The embedded Team.members array is convenient for displaying a team aggregate. The TeamMembers collection is better for membership lookup, approval, status, and per-member statistics.

Trade-off: the same relationship exists in two places. Updates must keep them synchronized. Inconsistency is possible if one is updated and the other fails.

### TeamMember unique index

~~~ts
await collection.createIndex(
  { userId: 1, teamId: 1 },
  { unique: true }
)
~~~

This ensures one membership record per user/team pair.

---

## 12. What is a repository?

A **repository** is a module that hides database details behind application functions.

Instead of a route writing MongoDB syntax directly:

~~~ts
collection.find({
  teamId,
  status: 'active'
})
~~~

the route calls:

~~~ts
getTeamMembers(teamId)
~~~

Why repositories are useful:

1. Routes remain focused on HTTP.
2. Database queries are centralized.
3. Team filters can be reused.
4. Data conversion can happen in one place.
5. Tests can target repository behavior separately.
6. Changing the persistence implementation becomes easier.

Uny10's repository-like files include:

~~~text
lib/db/companies.ts
lib/db/meetings.ts
lib/db/calendar.ts
lib/db/teams.ts
lib/db/team-members.ts
lib/db/support.ts
lib/db/users-auth.ts
~~~

They expose functions such as:

~~~text
getAllCompanies
createCompany
updateCompany
getUpcomingMeetings
createMeeting
getCalendarEntriesByUserAndDate
createTeam
getTeamMember
upsertTeamMember
~~~

---

## 13. Repository example: companies

The native repository is:

~~~text
lib/db/companies.ts
~~~

### Collection access

~~~ts
async function getCompaniesCollection(): Promise<Collection<Company>> {
  const { db } = await connectToDatabase()
  return db.collection(
    process.env.MONGODB_COMPANIES_COLLECTION || 'companies'
  )
}
~~~

This separates collection selection from every individual operation.

### Team filter

~~~ts
function buildTeamFilter(teamId: string) {
  if (teamId === 'default-team') {
    return {
      $or: [
        { teamId },
        { teamId: { $exists: false } }
      ]
    }
  }
  return { teamId }
}
~~~

For normal teams, only that team is selected. For default-team, older records without teamId are included for backward compatibility.

### Read flow

~~~ts
export async function getAllCompanies(teamId = 'default-team') {
  const collection = await getCompaniesCollection()
  const companies = await collection
    .find(buildTeamFilter(teamId))
    .toArray()

  return companies.map((doc) => ({
    ...doc,
    id: doc._id.toString(),
  }))
}
~~~

Step-by-step:

1. Get or reuse the native MongoDB connection.
2. Select the configured companies collection.
3. Build the team filter.
4. Query MongoDB.
5. Convert the cursor into an array.
6. Add an easier string id for the frontend.
7. Return the application-level Company objects.

### Create flow

~~~ts
const newCompany = {
  ...companyData,
  teamId,
  createdAt: new Date().toISOString(),
  updatedAt: new Date().toISOString(),
}

const result = await collection.insertOne(newCompany)
~~~

The repository assigns:

- teamId
- createdAt
- updatedAt

Then it returns the inserted ObjectId both as _id and string id.

### Update flow

The repository uses findOneAndUpdate with:

- company ID
- team filter
- $set for changed fields
- updatedAt
- returnDocument: after

This is important because the response contains the updated document instead of requiring a second read.

### Embedded follow-up operations

Adding a follow-up uses:

~~~ts
$push: { followUps: followUp }
~~~

Deleting one uses:

~~~ts
$pull: { followUps: { id: followUpId } }
~~~

Updating one uses arrayFilters to target the matching embedded item.

---

## 14. Repository example: meetings

The native repository is:

~~~text
lib/db/meetings.ts
~~~

### IDs can have two forms

The helper builds an ID filter that supports:

- Custom id field
- MongoDB _id when the input is a valid ObjectId

Conceptually:

~~~ts
const filters = [{ id }]
if (ObjectId.isValid(id)) {
  filters.push({ _id: new ObjectId(id) })
}
~~~

This is a compatibility strategy. It lets older/custom IDs and MongoDB IDs work.

### Team-scoped reads

The repository combines filters:

~~~ts
.find({
  approvalStatus: 'pending',
  ...buildTeamFilter(teamId)
})
~~~

The team filter is repeatedly applied to prevent cross-team reads.

### RSVP update

The RSVP operation:

1. Removes the existing response for the user with $pull.
2. Adds the new response with $push.
3. Updates updatedAt.
4. Returns the updated meeting.

This is a two-step update. If the second operation fails after the first succeeds, the previous RSVP may already be removed. A transaction could make this more atomic if necessary.

### Meeting status queries

getUpcomingMeetings reads scheduled meetings, parses dates in application code, filters out past dates, and sorts the result.

This is simple and readable, but for a large dataset it may be more efficient to query the date range directly in MongoDB instead of loading all scheduled meetings first.

---

## 15. Repository example: calendar

The native repository is:

~~~text
lib/db/calendar.ts
~~~

It stores Date values and also derives a dateKey such as 2026-10-20.

### Why date and dateKey both exist

date supports date-range queries using MongoDB Date comparisons.

dateKey supports exact day matching and compatibility with older records.

The repository normalizes entries so the frontend receives a consistent date and dateKey.

### Soft-delete read rule

Most calendar reads include:

~~~ts
isActive: true
~~~

Deletion does not remove the document. It updates:

~~~ts
$set: {
  isActive: false,
  updatedAt: new Date()
}
~~~

Every query must remember this rule.

---

## 16. What is an index?

An index is a database lookup structure that helps MongoDB find documents faster.

Without an index, MongoDB may scan every document in a collection. This is called a collection scan.

With an index, MongoDB can use an ordered lookup structure for a field or field combination.

Example:

~~~ts
MeetingSchema.index({ date: 1, time: 1 })
~~~

This helps queries that filter or sort by date and time.

### Index direction

- 1 means ascending order.
- -1 means descending order.

Example:

~~~ts
{ createdAt: -1 }
~~~

supports newest-first ordering.

### Compound index

A compound index contains multiple fields:

~~~ts
{ date: 1, userId: 1, teamId: 1 }
~~~

This is useful when queries commonly use those fields together.

### Unique index

A unique index prevents duplicate values.

Example:

~~~ts
{ userId: 1, teamId: 1 }, { unique: true }
~~~

This prevents duplicate membership records for the same user and team.

### Sparse unique index

User googleId uses:

~~~ts
{ unique: true, sparse: true }
~~~

Sparse means only documents that contain the field participate in the unique constraint. Users without a Google ID do not all conflict with each other.

---

## 17. Indexes actually used in Uny10

### User indexes

~~~text
googleId unique sparse
email unique
teamIds
teamId + status
activeTeamId
teamId + role
teamName + designation
teamId + designation
~~~

They support identity, team, role, and status lookups.

### Company indexes

Native ensureIndexes in lib/db/companies.ts creates:

~~~ts
{ name: 1 }
{ type: 1, status: 1 }
{ teamId: 1, name: 1 }
{ updatedAt: -1 }
{ 'followUps.id': 1 }, { sparse: true }
~~~

The important multi-tenant index is teamId + name.

### Meeting indexes

~~~text
createdById
date + time
approvalStatus + requestType
rsvpStatus
createdAt descending
~~~

### Calendar indexes

~~~text
date
userId
isActive
userId + date
~~~

### Visit plan indexes

~~~text
date
userId
teamId
festId
date + userId + teamId unique
festId + date
~~~

The unique date/user/team index enforces one plan per person per team per date.

### Team indexes

lib/db/teams.ts creates:

~~~text
teamId unique
teamNameNormalized unique
joinCode unique
coordinator.userId
~~~

These enforce unique team IDs, names, and join codes.

### TeamMembers indexes

lib/db/team-members.ts creates:

~~~text
userId + teamId unique
teamId
userId
~~~

This supports both “members of this team” and “teams for this user” queries.

### Notification index

userId is indexed because notifications are retrieved for the current user.

---

## 18. How indexes are created

There are two patterns.

### Mongoose-declared indexes

Indexes are declared inside schemas:

~~~ts
UserSchema.index(...)
MeetingSchema.index(...)
VisitPlanSchema.index(...)
~~~

Mongoose can create/manage these indexes according to its configuration and model behavior.

### Native createIndex calls

Native repositories explicitly create indexes:

~~~ts
await collection.createIndex({ teamId: 1, name: 1 })
~~~

Uny10 calls these functions in application flows:

- companies route calls ensureIndexes on GET.
- teams route and createTeam call ensureTeamIndexes.
- team member operations call ensureTeamMemberIndexes.
- users-auth operations call ensureUserIndexes.

The functions catch errors and log warnings so the application can continue:

~~~ts
console.warn('Failed to ensure indexes, proceeding without them')
~~~

### Trade-off

Creating indexes at runtime is convenient for setup, but production systems often create and manage indexes through deployment/migration steps. Runtime creation can add startup latency or hide an operational problem if index creation fails.

---

## 19. Connection strategy 1: Mongoose connection

The file is:

~~~text
lib/mongodb.ts
~~~

It reads:

~~~ts
const MONGODB_URI = process.env.MONGODB_URI
~~~

If it is missing, the application throws an error.

The database name is configurable:

~~~ts
dbName: process.env.MONGODB_DB_NAME || 'omnispons'
~~~

### Global cache

The code stores connection state on a global object:

~~~ts
let cached = global as any

if (!cached.mongoose) {
  cached.mongoose = { conn: null, promise: null }
}
~~~

It checks for an existing connection:

~~~ts
if (cached.mongoose.conn) {
  return cached.mongoose.conn
}
~~~

If a connection is already being created, it reuses the promise:

~~~ts
if (!cached.mongoose.promise) {
  cached.mongoose.promise = mongoose.connect(...)
}
~~~

Then all callers await the same promise.

### Why cache the promise?

If multiple requests arrive at the same time while no connection exists, they should not all start separate connections. Sharing the in-flight promise avoids duplicate connection creation.

### bufferCommands: false

The options include:

~~~ts
bufferCommands: false
~~~

This tells Mongoose not to silently queue operations while the database connection is unavailable. Failures become visible sooner instead of waiting in a hidden buffer.

### DNS fallback

The file can configure DNS servers from MONGODB_DNS_SERVERS, with fallback values 8.8.8.8 and 1.1.1.1. This is a deployment/networking workaround for MongoDB host resolution problems.

Potential trade-off: forcing public DNS values can be environment-specific and should be reviewed carefully in controlled networks.

---

## 20. Connection strategy 2: native MongoClient

The file is:

~~~text
lib/db/mongodb.ts
~~~

It stores:

~~~ts
let cachedClient: MongoClient | null = null
let cachedDb: Db | null = null
~~~

The function first checks:

~~~ts
if (cachedClient && cachedDb) {
  return { client: cachedClient, db: cachedDb }
}
~~~

If there is no cached connection, it:

1. Reads MONGODB_URI.
2. Creates a MongoClient.
3. Configures maxPoolSize: 10 and minPoolSize: 1.
4. Connects.
5. Selects MONGODB_DB_NAME or omnispons.
6. Caches the client and database.

The max pool size limits simultaneous database connections from this client. This matters in serverless or multi-request environments.

It also exposes closeDatabase for explicit shutdown, mostly useful for scripts/tests or controlled processes.

---

## 21. Why connection reuse matters in Next.js/serverless deployment

A serverless function may be created many times. If every request does this:

~~~text
request → new MongoClient → connect → query → close
~~~

the application may become slow and create too many MongoDB connections.

Uny10 instead aims for:

~~~text
first invocation → connect and cache
later invocation → reuse cached connection
~~~

Important limitation: a global cache is reused only within the same runtime instance. It is not a distributed connection shared across every serverless instance. MongoDB pool limits and deployment scale still need to be considered.

---

## 22. Complete flow: creating a company through the API route

This flow uses the native repository path.

~~~text
User submits company form
  → client sends POST /api/companies
  → middleware checks early conditions
  → app/api/companies/route.ts POST
  → request.json()
  → resolveTeamIdFromRequest
  → enforceTeamAccess
  → ensure authentication/team access
  → createCompany(body, teamId)
  → get native MongoClient from lib/db/mongodb.ts
  → select companies collection
  → resolve contact user IDs
  → add teamId and timestamps
  → insertOne()
  → return inserted ObjectId and string id
  → NextResponse.json(company, { status: 201 })
  → client parses response
  → React state updates
~~~

The repository is responsible for persistence details. The route is responsible for HTTP and authorization orchestration.

---

## 23. Complete flow: creating a company through server-action-style code

The current CompaniesContent component imports from:

~~~text
lib/actions/companies.ts
~~~

That function:

1. Calls the Mongoose connection helper.
2. Optionally derives map coordinates.
3. Sets teamId in the payload.
4. Creates a Mongoose Company document.
5. Calls save().
6. Recalculates user/company statistics.
7. Converts the saved document into a plain frontend object.

~~~text
CompaniesContent form
  → lib/actions/companies.ts createCompany
  → lib/mongodb.ts Mongoose connection
  → Company model
  → MongoDB
  → toPlainCompany
  → component setCompanies
~~~

This does not use app/api/companies/route.ts in the same way as apiClient.createCompany.

The two paths have similar names but different implementations, so an interview answer should mention the actual import path.

---

## 24. Complete flow: a team-scoped company read

~~~text
GET /api/companies?teamId=team-123
  → middleware checks team context
  → route resolves team-123
  → getSessionUserId reads refresh_token
  → User lookup verifies token and expiry
  → getTeamById checks team
  → getTeamMember checks membership if needed
  → getAllCompanies(team-123)
  → MongoDB query includes teamId: team-123
  → companies returned
~~~

There are two separate protections:

1. Authorization: the user must be allowed to access team-123.
2. Query isolation: the database query must return only records belonging to team-123.

Both are needed.

---

## 25. Embedded versus referenced data in Uny10

### Company contacts and follow-ups

Embedded:

~~~text
Company
  ├─ companyPOC[]
  ├─ sponsorshipPOC[]
  └─ followUps[]
~~~

Good for: reading the company and related details together.

Risk: document growth and more complex array updates.

### Meeting RSVP and approval data

Embedded inside Meeting:

~~~text
Meeting
  ├─ attendees[]
  ├─ rsvpResponses[]
  └─ approvalRequest
~~~

Good for: showing one meeting with its current workflow state.

Risk: two concurrent RSVP updates need careful handling. The current repository removes then pushes, which is simple but not a single atomic replacement.

### Calendar references

Calendar stores meetingIds and visitIds rather than full meeting/visit objects.

Good for: keeping calendar entries small and allowing the original entity to change independently.

Risk: referenced documents can be deleted or missing, so the application must tolerate stale IDs.

### Team memberships

The project uses both embedded Team.members and separate TeamMembers records.

Good for: convenient team display plus efficient membership queries.

Risk: synchronization between the two representations.

---

## 26. Denormalization in Uny10

**Denormalization** means storing a value in more than one place to make reads easier or faster.

Examples:

- teamName appears in Team, TeamMembers, and sometimes User.
- companyName appears in assigned visit companies.
- userName appears in meetings, calendar, and notifications.
- user statistics are stored on User and TeamMembers.
- date and dateKey both exist for calendar data.

Why use it:

- Fewer joins or follow-up queries.
- Faster or simpler UI reads.
- Historical display can remain stable.

Trade-off:

- A name change must update copied values.
- Data can become stale.
- More write logic is needed.

Uny10's updateTeamName function demonstrates synchronization: it updates the Team document and then updates related Users and TeamMembers records.

---

## 27. ObjectId and string IDs

MongoDB usually creates an ObjectId in the _id field.

The application often exposes IDs as strings to the frontend:

~~~ts
id: result.insertedId.toString()
~~~

When a request sends the string back, repositories check whether it is a valid ObjectId:

~~~ts
if (ObjectId.isValid(id)) {
  query._id = new ObjectId(id)
} else {
  query.id = id
}
~~~

Why this conversion exists:

- Browser JSON naturally uses strings.
- MongoDB queries may require ObjectId values.
- Older documents may use a custom id field.
- The API can support both formats.

Trade-off: supporting two ID conventions increases compatibility but also query complexity and potential ambiguity.

---

## 28. Query operators used in the repositories

MongoDB update operators appear throughout Uny10.

| Operator | Meaning | Uny10 use |
|---|---|---|
| $set | Replace/set fields | Update company, meeting status, timestamps |
| $push | Append to an array | Add follow-up, RSVP, event |
| $pull | Remove matching array item | Delete follow-up, remove RSVP |
| $addToSet | Add only if not already present | Visit history, linked IDs |
| $inc | Increase a number | Visit/stat counters |
| $or | Match one of multiple conditions | Default-team compatibility, ID lookup |
| $exists | Check whether a field exists | Include older documents without teamId |
| $gte/$lte | Range comparison | Calendar date ranges |
| $ne | Not equal | Team name uniqueness check |

Example:

~~~ts
await collection.updateOne(
  { teamId, 'members.userId': userId },
  {
    $addToSet: {
      ['members.$.Visits.' + dateKey]: companyName,
    },
  }
)
~~~

The query identifies one team/member and the update changes a nested field.

---

## 29. Reliability and consistency concerns

### Runtime index creation

Convenient, but index failures may only produce warnings. A production deployment should verify indexes explicitly.

### Separate Team and TeamMembers records

A team update can succeed while the membership update fails. A transaction or reconciliation process could improve consistency.

### Two-step RSVP update

Removing an old RSVP and inserting a new one are separate operations. A transaction or atomic array update could avoid intermediate states.

### Denormalized names and statistics

Copied values can become stale. Scheduled reconciliation or event-driven updates could help.

### Soft deletes

Every read must include isActive: true. Missing that filter can expose logically deleted data.

### Date strings versus Date values

Meetings store date as a string, while calendar and visit plans use Date. This makes date logic less uniform. A consistent UTC/date strategy would reduce parsing bugs.

### Manual validation in native repositories

TypeScript interfaces help during development, but they do not validate runtime JSON. Route schemas should validate before repository calls.

---

## 30. How to explain this topic to an interviewer

### Simple interview answer

> “Uny10 uses MongoDB as its main database. It has two data-access styles: Mongoose schemas and native MongoDB repositories. The Mongoose models in lib/models describe fields, nested objects, enums, defaults, and indexes. The repositories in lib/db hide database operations behind functions like getAllCompanies, createMeeting, and getTeamMember. API routes call those functions instead of writing MongoDB queries directly. Most records include teamId, and repositories include team filters so one team's data is isolated from another team's data. Indexes support common lookups such as email, team, status, dates, and membership. The project also caches Mongoose and native MongoDB connections so every request does not create a new connection. Some data is embedded, like company follow-ups and meeting RSVP responses, while some relationships are stored by ID, like meeting-to-company and calendar-to-meeting links.”

### Humanized version

> “I think of the database layer as three levels. The schema describes what a document should look like. The repository explains how the application asks for or changes that document. The connection helper makes sure the application can reach MongoDB efficiently. For example, when a user creates a meeting, the API route checks access, calls the meetings repository, the repository reuses a MongoDB connection and inserts a team-scoped document, and the route returns the created meeting. Indexes are like shortcuts that help MongoDB find common records faster.”

### If asked why both Mongoose and native MongoDB exist

> “The project currently uses both. Mongoose provides schemas, defaults, validation, and model hooks, while the native driver gives direct collection-level control. This can be practical during feature evolution, but it creates consistency risks around validation, serialization, indexes, and team filters. I would consider standardizing new features on one approach or documenting a clear boundary.”

---

## 31. Likely interviewer questions and answers

### Q1. What is a MongoDB document?

A JSON-like object stored inside a collection. In Uny10, one document can represent a user, company, meeting, calendar entry, or support request.

### Q2. What is a collection?

A group of related MongoDB documents. Examples are Users, Teams, TeamMembers, companies, meetings, calendar, VisitPlans, Notifications, and Support.

### Q3. What is a schema?

A description of the expected fields, types, defaults, allowed values, and indexes for a document. Uny10 uses Mongoose schemas for several entities.

### Q4. Does MongoDB itself enforce the Mongoose schema?

No. Mongoose enforces its schema when writes go through the Mongoose model. Native MongoDB writes need their own runtime validation.

### Q5. What is a repository?

A module that hides database operations behind application functions. It keeps API routes focused on HTTP and business orchestration.

### Q6. Why use repositories?

They centralize database queries, team filters, ID conversion, and update logic. They also make it easier to change or test persistence code.

### Q7. What is an index?

An index is a lookup structure that helps MongoDB find and sort documents faster without scanning the entire collection.

### Q8. What is a compound index?

An index containing multiple fields, such as date, userId, and teamId. It is useful when queries commonly use those fields together.

### Q9. What is a unique index?

An index that prevents duplicate values. Uny10 uses one on userId plus teamId for TeamMembers and on date plus userId plus teamId for VisitPlans.

### Q10. Why use a sparse unique index for googleId?

Some users may not have a googleId. Sparse uniqueness applies only to documents that contain the field, so missing values do not all conflict.

### Q11. Why is teamId included in repository filters?

Uny10 is multi-team. Team filters prevent a user authorized for one team from reading or modifying another team's records.

### Q12. Is a teamId from the request trustworthy?

No. It is client-supplied context. The server must validate the session user's access to that team, then use the team ID in database filters.

### Q13. What is embedded data?

Data stored inside its parent document, such as company follow-ups or meeting RSVP responses. It is convenient for reading related information together.

### Q14. What is referenced data?

Data stored separately and connected with an ID, such as meeting.companyId or calendar.meetingIds. It reduces parent document growth but may require additional queries.

### Q15. Why does Uny10 use both embedded Team.members and TeamMembers?

The embedded array is convenient for team-level display, while the TeamMembers collection supports efficient membership and approval queries. The trade-off is synchronization risk.

### Q16. What is a soft delete?

Instead of removing a document, the application marks it inactive. Calendar uses isActive: false. Reads must consistently filter isActive: true.

### Q17. Why convert ObjectId to a string?

Browsers and JSON use strings naturally. The server converts the string back to ObjectId when querying MongoDB.

### Q18. Why cache a MongoDB connection?

Opening a connection for every request is slow and can exhaust database connection limits. Uny10 reuses cached Mongoose or native client connections.

### Q19. Why cache the connection promise too?

Concurrent first requests can share the same in-flight connection promise instead of creating several simultaneous connections.

### Q20. What does maxPoolSize mean?

It limits the number of connections in the native MongoDB driver's connection pool. Uny10 sets maxPoolSize to 10 in lib/db/mongodb.ts.

### Q21. What is the difference between Mongoose and the native MongoDB driver?

Mongoose provides schemas, models, validation, defaults, and hooks. The native driver works directly with collections and gives lower-level control but requires more manual application logic.

### Q22. What is a Mongoose pre-save hook?

A function that runs before Mongoose save(). VisitPlan uses one to calculate totalAssignedCompanies from assignedCompanies.length.

### Q23. What limitation does the VisitPlan pre-save hook have?

It may not run for every direct update method such as findOneAndUpdate. If updates bypass save(), the derived count can become stale.

### Q24. What happens if index creation fails?

Several ensure-index functions catch the error, log a warning, and allow the application to continue. This keeps the app available but can cause slower queries or missed uniqueness guarantees.

### Q25. How would you optimize getUpcomingMeetings?

Currently the repository may load scheduled meetings and filter dates in JavaScript. For a large collection, I would query MongoDB directly using a date range and an index, reducing data transferred and work done in the application.

### Q26. What consistency problem exists with RSVP updates?

The repository removes the user's existing RSVP and then adds a new one in two operations. A failure between them could leave no response. A transaction or atomic update could improve consistency.

### Q27. What happens if a company document grows too large because of followUps?

MongoDB documents have a size limit. If follow-ups become numerous, a separate FollowUps collection would be safer and easier to query independently.

### Q28. What is denormalization?

Storing copied information in multiple places for easier or faster reads. Uny10 copies names, team names, and statistics in several documents. The trade-off is possible stale data.

### Q29. How would you improve the data layer?

I would choose a clear persistence boundary, standardize validation, make team scoping mandatory in repository APIs, use consistent dates and IDs, verify indexes during deployment, add transactions for multi-document updates, and add tests for cross-team access.

### Q30. How would you explain the database request flow?

> “The route validates the request and user/team access, then calls a repository. The repository obtains a cached connection, selects a model or collection, builds a team-scoped query, performs the MongoDB operation, converts database-specific values into application values, and returns the result to the route.”

---

## 32. Important things to remember

1. MongoDB stores documents inside collections.
2. A Mongoose schema describes the expected document shape.
3. A repository hides database query details from route handlers.
4. Uny10 uses both Mongoose models and native MongoDB repositories.
5. Mongoose provides schemas, defaults, validation, and hooks.
6. Native repositories provide direct collection access and require more manual validation.
7. Users, companies, meetings, calendar, visit plans, and notifications have Mongoose models.
8. Companies, meetings, calendar, teams, team members, support, and users-auth have repository modules under lib/db.
9. teamId is central to multi-team data isolation.
10. Authorization and database team filters are separate protections.
11. Embedded data is convenient but can make documents large.
12. Referenced IDs keep entities separate but require more coordination.
13. Indexes speed up common lookups and can enforce uniqueness.
14. Compound indexes match common multi-field queries.
15. Unique indexes protect rules such as one membership per user/team.
16. Sparse unique indexes are useful when a unique field is optional.
17. ObjectIds are converted to strings for frontend JSON.
18. Mongoose and native MongoDB connections are cached separately.
19. Connection caching is important for Next.js/serverless runtimes.
20. Runtime index creation is convenient but should be verified operationally.
21. Soft deletes require every read to include the active filter.
22. Denormalized data improves reads but creates synchronization work.
23. Dates are not fully uniform: meetings use strings while calendar/visit plans use Date.
24. The current code has useful backward-compatibility behavior for default-team and older IDs.
25. The most important interview summary is: schema → repository → cached connection → team-scoped query → MongoDB → normalized response.

---

## 33. Final start-to-finish summary

~~~text
User action
  → API route or server action
  → request/session/team validation
  → repository or Mongoose model
  → cached Mongoose/native MongoDB connection
  → collection/document query
  → index-assisted lookup where available
  → embedded update or referenced-record update
  → ObjectId/date normalization
  → JSON response or plain server result
  → frontend state update
~~~

A simple interview sentence is:

> “In Uny10, schemas describe the data shape, repositories hide database operations, indexes make common queries and uniqueness rules efficient, and cached connection helpers prevent opening a new MongoDB connection for every request.”

