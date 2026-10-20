# Authentication and Authorization — Interview Guide for Uny10

This note is designed for explaining Uny10's authentication and authorization process in a simple, natural, interview-friendly way.

It describes the current Uny10 implementation, while also pointing out important security improvements that could be made.

---

## 1. The simple difference

### Authentication

Authentication answers:

> “Who are you?”

In Uny10, authentication happens when a user signs in with Google. The application receives the user's Google information, finds or creates the user in the database, creates a session token, and stores that token in a cookie.

### Authorization

Authorization answers:

> “What are you allowed to do?”

After Uny10 knows who the user is, it checks whether that user can access a team or perform an operation. It uses information such as:

- User identity
- Team membership
- Role, such as coordinator or subcoordinator
- Membership status
- Approval status
- Active team

Authentication happens first. Authorization happens after authentication.

Simple example:

```text
Authentication: This is Rahul.
Authorization: Rahul is an approved coordinator in Team A, so he can approve members in Team A.
```

---

## 2. How to explain the complete process simply

### Interview-friendly short answer

> “In Uny10, the user signs in through Google on the frontend. After Google login succeeds, the login page sends the user's Google-related information to the backend API at `/api/auth/google-callback`. The backend finds or creates the user, generates a random session token, stores that token and its expiry in MongoDB, and sends the token back as an HttpOnly cookie. On later requests, the browser automatically sends that cookie. The server reads the cookie, looks up the matching user and checks whether the session has expired. That is authentication. For authorization, the server checks the user's membership in the requested team, their role, whether they are active, and whether their membership has been approved. Only after these checks does the server allow the requested operation.”

This is usually enough for the first explanation. Do not begin with every file, token type, or security detail unless the interviewer asks for more depth.

### More humanized version

> “I think of it as two questions. First, authentication confirms the person's identity. In our application, Google helps us identify the user, and we create our own application session after login. Second, authorization decides what that identified user can do. Since Uny10 is team-based, we check whether the user belongs to the selected team and whether their membership and role allow the requested action. The browser keeps the session cookie, but the server remains responsible for making the final security decision.”

---

## 3. The actual Uny10 flow in simple words

```text
User clicks Google Login
        ↓
app/login/page.tsx receives the Google result
        ↓
Frontend sends POST /api/auth/google-callback
        ↓
Backend finds or creates the user
        ↓
Backend creates a random session token
        ↓
Token is stored in MongoDB and sent as an HttpOnly cookie
        ↓
Browser sends the cookie on later requests
        ↓
/api/auth/session identifies the user
        ↓
Protected routes check team membership and permissions
        ↓
The server allows or rejects the operation
```

### Main files involved

| Responsibility | Uny10 code |
|---|---|
| Login UI and Google callback | `app/login/page.tsx` |
| Create or find user and create session | `app/api/auth/google-callback/route.ts` |
| Read and validate current session | `app/api/auth/session/route.ts` |
| Read the user ID from the session cookie | `lib/validate-session.ts` |
| Check team access | `lib/team-scope.ts` |
| Client-side authentication state | `lib/auth-context.tsx` |
| Add auth/team headers to API requests | `lib/auth-fetch.ts` |
| Redirect users based on authentication state | `components/auth/auth-route-guard.tsx` |
| Logout and revoke session | `app/api/auth/logout/route.ts` |
| Team membership and approval data | `lib/models/team-members.ts`, `lib/db/team-members.ts` |

---

## 4. Interview script: authentication

Use this script if the interviewer asks: “How does authentication work in your project?”

> “The user starts on the login page, which is implemented in `app/login/page.tsx`. We use Google Login to identify the user. When Google login succeeds, the frontend receives the Google result and sends a POST request to `/api/auth/google-callback`.
>
> The API route in `app/api/auth/google-callback/route.ts` validates the submitted data, searches for the user, and creates a user if one does not exist. It then creates a random application session token using `crypto.randomUUID()`. The token and its expiration time are stored in the user record in MongoDB. The same token is returned to the browser through an HttpOnly `refresh_token` cookie.
>
> For later requests, the browser automatically sends that cookie. The server reads the cookie and searches for a user whose stored token matches and whose expiration time is still in the future. If a matching user is found, the request is authenticated. If not, the server returns an unauthenticated response.”

### Why this answer is good

It explains:

1. Where login begins.
2. Which API receives the request.
3. How the server identifies the user.
4. How the session is created.
5. How later requests are recognized.

It avoids unnecessary jargon while still showing that you understand the complete flow.

---

## 5. Interview script: authorization

Use this script if the interviewer asks: “How do you authorize users?”

> “Authentication only tells us who the user is, so Uny10 performs a second step for authorization. Before a protected operation runs, the server identifies the user from the session cookie and checks access to the requested team. The team access logic in `lib/team-scope.ts` checks whether the user is a coordinator or an approved and active member in `TeamMembers`. The user's role, membership status, and approval status determine what they can do. For example, a pending member should not receive the same access as an approved member, and a user from one team should not be able to access another team's data. The backend performs the final authorization check instead of relying only on frontend visibility.”

### Shorter version

> “Authentication identifies the user. Authorization checks the user's relationship with the requested team. In Uny10, that means checking team membership, role, active status, and approval status before allowing the operation.”

---

## 6. Interview script: authentication versus authorization

> “Authentication is identity verification, while authorization is permission checking. In Uny10, Google login helps identify the user and the application session keeps the user logged in. Authorization then checks whether that user is an approved member of the selected team and whether their role permits the requested action. So authentication answers ‘who is this?’ and authorization answers ‘what can this user do?’”

---

## 7. Interview script: cookies, sessions, and tokens

> “A token is a value that represents a login session. A session is the server-side record that connects that token to a user. A cookie is the browser mechanism used to send the token back to the server automatically. In Uny10, the random token is stored in the database along with its expiry, and the browser stores the same value in an HttpOnly cookie. On each protected request, the server uses the cookie value to find the corresponding user and verify that the session has not expired.”

### Important clarification

Uny10's `refresh_token` is an opaque database-backed session token. It is not a JWT and it is not the Google OAuth refresh token.

```text
Cookie  = browser storage and transport mechanism
Token   = random session identifier
Session = server-side relationship between token and user
```

---

## 8. Interview script: client-server communication

> “The frontend and backend communicate over HTTP. The frontend creates a request using `fetch`, including the method, headers, and JSON body. Next.js maps the URL `/api/auth/google-callback` to the server file `app/api/auth/google-callback/route.ts`. The route reads the request body, performs database and session work, and returns a JSON response. The browser also receives the session cookie through the response. For future requests, `credentials: 'include'` allows cookies to be sent, and the backend uses the cookie to authenticate the request.”

### Simple request flow

```text
Frontend fetch()
    ↓
HTTP request: POST /api/auth/google-callback
    ↓
Next.js route.ts
    ↓
Database and session logic
    ↓
HTTP response: JSON + Set-Cookie
    ↓
Frontend updates state and UI
```

---

## 9. Likely interviewer questions and answers

### Q1. What is authentication?

**Answer:**

Authentication verifies the identity of a user. In Uny10, Google Login provides the initial identity information. The backend then creates an application session so the user can remain logged in across requests.

### Q2. What is authorization?

**Answer:**

Authorization decides whether an already identified user is allowed to access a resource or perform an action. In Uny10, authorization is mainly based on team membership, role, active status, approval status, and the selected team.

### Q3. What is the difference between authentication and authorization?

**Answer:**

Authentication asks “Who are you?” Authorization asks “What are you allowed to do?” A user may be successfully authenticated but still be unauthorized to access a particular team or operation.

### Q4. Where does the login request go in Uny10?

**Answer:**

The client code in `app/login/page.tsx` sends a POST request to `/api/auth/google-callback`. Next.js maps that URL to `app/api/auth/google-callback/route.ts`.

### Q5. Is `app/login/page.tsx` client-side code?

**Answer:**

Yes. It contains `'use client'`, so it runs in the browser. It displays the login interface, handles the Google callback, and sends data to the backend API.

### Q6. Is `app/api/auth/google-callback/route.ts` server-side code?

**Answer:**

Yes. It is a Next.js API route and runs on the server. It processes the request, accesses MongoDB, creates or updates the user, creates a session token, sets the cookie, and returns the response.

### Q7. What happens after Google Login succeeds?

**Answer:**

The login component receives the Google result, extracts the relevant user information, and sends it to `/api/auth/google-callback`. The server finds or creates the user, creates a random token, stores it with an expiry in MongoDB, sets the `refresh_token` cookie, and returns a status such as authenticated, onboarding required, waiting for approval, or rejected.

### Q8. Does Uny10 use JWT for its application session?

**Answer:**

No. The application session uses an opaque random token created with `crypto.randomUUID()`. The token itself does not contain user information. The server looks it up in MongoDB and obtains the user from that database record.

The Google credential has a JWT-like structure, but that is separate from Uny10's own session token.

### Q9. What is an opaque token?

**Answer:**

An opaque token is a random identifier whose meaning cannot be understood by simply decoding it. The server must look it up in a database or session store. Uny10's application session token is opaque.

### Q10. Why store the session token in an HttpOnly cookie?

**Answer:**

An HttpOnly cookie cannot be read by normal JavaScript in the browser. This reduces the impact of many XSS attacks because injected JavaScript cannot directly read the session token. The browser can still send the cookie automatically to the matching server.

### Q11. What does `credentials: 'include'` do?

**Answer:**

It tells the browser to include cookies with the request and accept cookies from the response when the request rules allow it. Uny10 uses this when making authentication-related API requests.

### Q12. What happens when the user reloads the page?

**Answer:**

The client-side authentication context restores local user information and calls `/api/auth/session`. The session route reads the `refresh_token` cookie, finds a matching non-expired user in MongoDB, reconciles team information, and returns the current user. The frontend then updates the authentication context and renders the appropriate page.

### Q13. How does the server know that a session is valid?

**Answer:**

`lib/validate-session.ts` reads the `refresh_token` cookie. The server searches for a user whose stored token matches the cookie and whose `refreshTokenExpires` value is greater than the current time. If no such user exists, the session is invalid.

### Q14. What happens when a session expires?

**Answer:**

The database query no longer finds the user because the expiry condition fails. The API returns an unauthenticated response, usually HTTP 401. The frontend can then clear its local user state and redirect the user to login.

### Q15. How does Uny10 authorize access to a team?

**Answer:**

The backend first identifies the user from the session. It then uses team access logic in `lib/team-scope.ts` to check the requested team. Coordinators may receive direct access based on their role, while other users generally need an active and approved `TeamMembers` record. The request is rejected when the user does not have access.

### Q16. Why is authorization checked on the backend if the frontend hides buttons?

**Answer:**

Frontend hiding is only a user-interface feature. A malicious user can call an API directly without using the visible button. Therefore, the backend must perform the real authorization check for every sensitive operation. The frontend can improve user experience, but it cannot be the security boundary.

### Q17. What is middleware in this context?

**Answer:**

Middleware is code that runs before a request reaches the final route handler. In Uny10, `middleware.ts` performs early request checks such as public API handling, team-scope presence, and some client-supplied status checks. The more authoritative session and team checks are performed by helpers such as `getSessionUserId` and `enforceTeamAccess`.

### Q18. Is the middleware alone enough to secure Uny10?

**Answer:**

No. Middleware is an early filter, but each sensitive route should independently validate the session and authorization. The server should not trust only frontend headers such as `x-user-id` or `x-user-status`. The authoritative identity should come from the server-validated session cookie.

### Q19. What is the role of `AuthProvider`?

**Answer:**

`AuthProvider` in `lib/auth-context.tsx` stores the current user and loading state for the React application. It restores cached information, verifies the session with `/api/auth/session`, exposes logout and refresh functions, and informs components when the authentication state changes. It improves frontend coordination, but it does not replace backend authorization.

### Q20. What happens during logout?

**Answer:**

The client removes its cached user information and calls `/api/auth/logout`. The server finds the session using the cookie, clears the stored token and expiry in MongoDB, and clears the browser cookie using an expired cookie. Future requests using the old token should fail.

### Q21. Why does Uny10 store a token in the database instead of putting user information directly in a JWT?

**Answer:**

A database-backed opaque session makes server-side revocation straightforward. Logout or forced invalidation can remove the token from the database immediately. The trade-off is that the server needs a database lookup for session validation. A JWT can reduce lookups, but revocation and permission changes are more complicated unless another mechanism is added.

### Q22. What is the difference between a cookie and a session?

**Answer:**

A cookie is stored by the browser and sent with requests. A session is the server-side login state associated with the user. In Uny10, the cookie contains the session token, while MongoDB stores the token-to-user relationship and expiration time.

### Q23. What is the difference between a session token and a Google token?

**Answer:**

The Google credential comes from Google and represents the Google login result. Uny10 then creates its own application session token. The application uses its own session token for later requests instead of treating the Google credential as the application's long-term session.

### Q24. What security issue exists in the current Google callback design?

**Answer:**

The frontend decodes the Google credential and sends fields such as `googleId`, `email`, `name`, and `avatar` to the backend. Decoding is not the same as cryptographic verification. A stronger implementation would send the original credential to the server and verify its signature, issuer, audience, expiry, and relevant claims before trusting the identity.

### Q25. What security issue can occur if the server trusts `x-user-id`?

**Answer:**

The client controls request headers, so a malicious client could change `x-user-id`. The server should derive the user identity from the validated session cookie and use client headers only as optional context, never as proof of identity.

### Q26. How would you protect against XSS?

**Answer:**

I would avoid rendering untrusted HTML, escape user-controlled content, use framework protections, validate input, and apply a Content Security Policy where appropriate. The HttpOnly cookie also prevents injected JavaScript from directly reading the session token, although it does not prevent every form of XSS damage.

### Q27. How would you protect against CSRF?

**Answer:**

Because cookies are automatically sent by the browser, state-changing requests need CSRF protection when the deployment scenario requires it. Possible protections include SameSite cookies, checking the Origin or Referer header, and using a CSRF token for sensitive mutations. Uny10 uses `SameSite=Lax`, which helps, but a complete production design should evaluate every cross-site request scenario.

### Q28. What is session fixation and how can it be avoided?

**Answer:**

Session fixation happens when an attacker gets a victim to use a session identifier known to the attacker. The usual protection is to create a new random session token after successful authentication and never accept attacker-selected session identifiers. Uny10 creates a random token during the Google callback, which is the correct general direction.

### Q29. What would you improve in Uny10's authentication design?

**Answer:**

I would make the server verify the original Google credential, derive the user entirely from the server-side session, remove reliance on client-provided identity headers, apply authorization consistently to every protected route, add CSRF protections for sensitive cookie-authenticated mutations, rotate sessions when appropriate, add rate limiting to login and onboarding operations, and use secure cookie settings in every production environment.

### Q30. How would you explain the security boundary?

**Answer:**

The browser is not trusted. It can display the UI and send requests, but the backend is the security boundary. The backend must validate the session, load the user, check team membership and permissions, validate input, and decide whether the operation is allowed.

---

## 10. A complete answer for a deeper interview discussion

If the interviewer asks for the complete end-to-end design, use this answer:

> “The user begins authentication on the client in `app/login/page.tsx`. After Google Login succeeds, the frontend calls the server route `/api/auth/google-callback`. The server-side route finds or creates the user and creates an opaque random application session token. It stores the token and its expiry in MongoDB and sends the token in an HttpOnly cookie.
>
> When the browser later requests a protected resource, it automatically sends the cookie. The server reads it through the request cookies and validates it by finding the matching non-expired user. The current session is also loaded by `/api/auth/session` when the application starts or reloads.
>
> After authentication, authorization checks whether the user is allowed to access the requested team. Uny10 uses team membership, role, active state, approval state, and active-team information. The final decision is made on the backend, because frontend checks can be bypassed. On logout, the server removes the stored token and clears the cookie, so the old session is no longer valid.
>
> One improvement I would make is to verify the original Google credential on the server rather than trusting decoded fields sent by the browser. I would also ensure every protected route derives identity from the session cookie rather than from client-provided headers.”

---

## 11. Common mistakes to avoid in an interview

Do not say:

- “The frontend authenticates the user.” The frontend starts the process, but the server creates and validates the application session.
- “The cookie is the session.” The cookie carries the token; the server-side session state is stored in the database.
- “The `refresh_token` is a JWT.” In Uny10 it is an opaque random database-backed session token.
- “Hiding a button authorizes a user.” Real authorization must happen on the backend.
- “Decoding a JWT verifies it.” Decoding only reads its contents; signature and claim verification are separate.
- “Middleware handles all authorization.” In Uny10, middleware performs early checks, while route-level session and team access helpers are important for authoritative protection.
- “The client can be trusted because it is our React application.” A user can send custom HTTP requests without using the UI.

---

## 12. Important things to remember

1. Authentication means identity; authorization means permission.
2. Google login is the initial identity provider.
3. Uny10 creates its own opaque application session token.
4. The token is stored in MongoDB and transported in an HttpOnly cookie.
5. `/api/auth/session` restores and verifies the current login.
6. `lib/validate-session.ts` is used to derive the user from the cookie.
7. `lib/team-scope.ts` checks access to the requested team.
8. Team role, active status, and approval status affect authorization.
9. Frontend state improves the UI but is not a security boundary.
10. Client-provided user IDs and status headers must not be treated as proof of identity.
11. Decoding a Google credential is not the same as verifying it.
12. Logout must invalidate the server-side session, not only remove local browser data.
13. The best first explanation is: “Who is the user?” followed by “What can that user do?”

---

## 13. One-minute final answer

> “Uny10 separates authentication from authorization. Authentication starts with Google Login. The client sends the login result to the backend, where the user is found or created and an opaque random session token is stored in MongoDB. The token is sent to the browser in an HttpOnly cookie. Later requests include that cookie, and the backend validates it by finding the matching non-expired user. Authorization then checks the user's access to the requested team using membership, role, active status, and approval status. The frontend can hide or show UI based on this state, but the backend makes the final security decision. During logout, the backend revokes the stored token and clears the cookie.”

