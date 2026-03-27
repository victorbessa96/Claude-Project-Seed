# Module: Authentication & Authorization

Patterns for building auth systems: verifying who users are, controlling what they can do, and managing identity across sessions. This module covers the full auth architecture, not just the security primitives in [06-security](../06-security.md).

**Prerequisite knowledge:** [06-security](../06-security.md) (password hashing properties, session cookie flags, CSRF), [03-architecture](../03-architecture.md) (layered architecture, middleware, cross-layer concerns), [04-data-modeling](../04-data-modeling.md) (schema design, relationships).

---

## Auth Architecture Decision

The first decision is how to identify authenticated users across requests.

| Approach | How it works | Stateful? | Revocation | Best for |
|----------|-------------|-----------|------------|----------|
| **Session-based** | Server stores session data; client sends a cookie with session ID | Yes (server stores sessions) | Immediate (delete session) | Traditional web apps, server-rendered pages |
| **Token-based (JWT)** | Server issues a signed token; client sends it with each request | No (token is self-contained) | Hard (token valid until expiry) | APIs, mobile apps, microservices |
| **OAuth / SSO** | Delegate auth to a provider; receive a token back | Depends on implementation | Via provider | Apps that want "Sign in with Google/GitHub" |
| **API keys** | Static key per client, sent with each request | No | Immediate (revoke key) | Service-to-service, developer APIs |

**How to choose:**
- If you have a traditional web frontend served by your backend: **session-based** is simplest
- If your frontend and backend are separate deployments (SPA + API): **token-based** or **session-based with cookies** both work; tokens add complexity but enable multi-service architectures
- If you don't want to manage passwords: **OAuth/SSO**
- If you need all three: start with one, add others as needed. Don't build for three auth methods on day one.

---

## Session-Based Auth

The server creates a session after login, stores it (in memory, database, or cache), and sends a session ID to the client as a cookie.

### Session Lifecycle

1. **Login:** Client sends credentials. Server validates, creates a session record (user ID, creation time, expiry), and returns a Set-Cookie header with the session ID.
2. **Subsequent requests:** Browser automatically attaches the cookie. Server reads the session ID, looks up the session, and identifies the user.
3. **Logout:** Server deletes the session record. Client's cookie becomes invalid.
4. **Expiry:** Sessions expire after a defined period. Expired sessions are rejected; user must re-authenticate.

### Session Storage

| Storage | Pros | Cons |
|---------|------|------|
| **In-memory** | Fast, simple | Lost on restart; doesn't scale to multiple servers |
| **Database** | Persistent, queryable | Slower; adds DB load per request |
| **External cache** | Fast, shared across servers | Extra infrastructure dependency |

**In practice:**
- For single-server apps: in-memory or database is fine
- For multi-server: database or shared cache
- Store minimal data in the session (user ID, role). Don't store large objects.
- Generate session IDs with a cryptographically secure random generator. Never use sequential IDs or predictable values.

### Expiration Strategies

- **Absolute expiration:** Session dies after N hours regardless of activity (e.g., 8 hours). Simple. Forces periodic re-authentication.
- **Sliding expiration:** Session extends by N minutes on each request. Sessions die only from inactivity. Users stay logged in during active use.
- **Hybrid:** Sliding expiration up to an absolute maximum (e.g., extends 30 minutes per request, but dies after 24 hours regardless).

**Why hybrid is usually best:** Sliding alone means a session can theoretically live forever. Absolute alone means active users get logged out mid-task. Hybrid balances security with usability.

---

## Token-Based Auth (JWT)

The server issues a signed token after login. The client stores it and sends it with each request. The server validates the signature without looking up a session.

### Token Structure

A JWT contains three parts: header (algorithm), payload (claims), and signature.

**Standard claims to include:**
- `sub` (subject): user ID
- `iat` (issued at): when the token was created
- `exp` (expiration): when the token expires
- `role` or custom claims: user's role or permissions

**What NOT to put in tokens:**
- Sensitive data (passwords, secrets): tokens can be decoded by anyone
- Large data (user profile, preferences): tokens are sent with every request; size matters
- Frequently changing data (notification count, online status): token data is frozen at issuance

### Token Storage on the Client

| Location | XSS vulnerable? | CSRF vulnerable? | Sent automatically? |
|----------|-----------------|-------------------|---------------------|
| **httpOnly cookie** | No (JS can't access) | Yes (cookies auto-attach) | Yes |
| **localStorage** | Yes (JS can read) | No (must be added manually) | No |
| **In-memory (variable)** | No (lost on page close) | No | No |

**Recommendation:** httpOnly cookie with SameSite=Strict and CSRF protection. This gets the best combination of XSS and CSRF protection. localStorage is acceptable for SPAs where CSRF isn't a concern, but XSS becomes the threat to mitigate.

### Refresh Token Rotation

Short-lived access tokens (15-30 minutes) paired with a long-lived refresh token (days to weeks).

**Flow:**
1. Login returns both an access token and a refresh token
2. Access token is used for API requests
3. When the access token expires, the client sends the refresh token to get a new pair
4. The old refresh token is invalidated (one-time use)
5. If the refresh token is also expired, the user must log in again

**Why rotation matters:** If a refresh token is stolen, it can only be used once. The legitimate user's next refresh attempt fails (the token was already used), signaling a compromise.

### Revocation Strategies

JWTs are hard to revoke because they're self-contained. Options:

- **Short-lived tokens:** Set expiry to 15-30 minutes. Worst case, a stolen token works for that window.
- **Token blocklist:** Maintain a list of revoked token IDs. Check on each request. Adds statefulness but enables immediate revocation.
- **Refresh token revocation:** Revoke the refresh token. The access token still works until expiry, but can't be renewed.

---

## OAuth / SSO Integration

Delegating authentication to an external provider (Google, GitHub, Microsoft, etc.).

### Authorization Code Flow

The most secure OAuth flow for server-side applications:

1. **Redirect to provider:** User clicks "Sign in with GitHub." Your app redirects to the provider's authorization URL with your client ID, redirect URI, and a random `state` parameter (CSRF protection).
2. **User authenticates:** The user logs in at the provider and grants permissions.
3. **Provider redirects back:** Provider redirects to your callback URL with an authorization `code` and the `state` parameter.
4. **Exchange code for token:** Your server sends the code + client secret to the provider's token endpoint. This happens server-to-server (the client secret never reaches the browser).
5. **Get user info:** Use the access token to fetch the user's profile from the provider.
6. **Create or link account:** Find or create a user record in your database.

### Account Linking

When a user signs in with a provider for the first time:
- Check if an account with that email already exists
- If yes: link the OAuth identity to the existing account (after confirmation)
- If no: create a new account from the provider's profile data
- Store the provider name, provider user ID, and access/refresh tokens in a separate `user_identities` table

### Token Management for Providers

- Store provider access and refresh tokens encrypted in the database
- Refresh provider tokens before they expire (if your app makes API calls on behalf of the user)
- Handle token revocation (user revokes access at the provider)

---

## Password Lifecycle

For apps that manage their own passwords (not OAuth-only).

### Registration Flow

1. Validate email format and uniqueness
2. Enforce password requirements (minimum length, not in common password lists)
3. Hash the password (see [06-security](../06-security.md) for hashing properties)
4. Store the hash. Never store the plain password, even temporarily.
5. Optionally: send a verification email before activating the account

### Login Flow

1. Look up the user by email/username
2. Compare the submitted password against the stored hash using constant-time comparison
3. On success: create a session or issue a token
4. On failure: return a generic error ("Invalid email or password"). Never reveal whether the email exists.
5. Log failed attempts for the account. After N failures (5-10), lock the account temporarily or require a CAPTCHA.

### "Forgot Password" Flow

1. User submits their email
2. Always respond with "If an account exists, we sent a reset link." Never confirm whether the email is registered.
3. Generate a single-use, time-limited token (30-60 minutes). Store its hash in the database.
4. Send the token in a link via email
5. When the user clicks the link: validate the token, let them set a new password, invalidate the token, and invalidate all existing sessions.

### Account Lockout

- After N consecutive failed login attempts, temporarily lock the account (15-30 minutes)
- Notify the user via email when their account is locked
- Don't reveal the lockout to the attacker (same generic error message)
- Consider escalating: 5 failures = 15 min lock, 10 failures = 60 min lock, 20 failures = account disabled until email verification

---

## Role & Permission Models

### RBAC (Role-Based Access Control)

Assign users to roles; roles have permissions.

```
User → Role → Permissions

Example:
  admin  → [create, read, update, delete, manage_users, manage_settings]
  editor → [create, read, update]
  viewer → [read]
```

**Why RBAC:** Simple to implement, easy to understand, covers most applications. A user's capabilities are determined by their role, not by individual permission assignments.

**In practice:**
- Store roles in a dedicated table or as an enum on the user record
- For simple apps (2-3 roles): a `role` column on the user table is sufficient
- For complex apps (many roles, custom permissions): use a roles table + role_permissions junction table
- Check permissions in middleware, not scattered through business logic

### ABAC (Attribute-Based Access Control)

Permissions based on attributes of the user, resource, and context.

```
Rule: user.department == resource.department AND user.clearance >= resource.classification
```

**When ABAC over RBAC:**
- When permissions depend on the relationship between the user and the specific resource (e.g., "users can edit their own posts but not others'")
- When rules are too numerous or dynamic for static roles
- When you need row-level access control

**In practice:** Most apps start with RBAC and add attribute-based rules for specific cases (e.g., "only the author can edit this article"). Pure ABAC is complex; use it only when RBAC genuinely can't express your rules.

### Permission Checking Pattern

```
# Middleware level: is the user authenticated?
require_auth(request)

# Route level: does the user have the right role?
require_role(request.user, "editor")

# Business logic level: does the user own this resource?
if article.author_id != request.user.id:
    raise Forbidden("You can only edit your own articles")
```

**Why layered checking:** Authentication (who are you?) is universal. Authorization (what can you do?) depends on the endpoint. Resource ownership (is this yours?) depends on the data. Each check belongs at a different layer.

---

## Multi-Tenancy Isolation

When multiple organizations or groups share the same application.

### Isolation Strategies

| Strategy | Isolation level | Complexity | Best for |
|----------|----------------|------------|----------|
| **Row-level** | Shared tables, `tenant_id` column | Low | SaaS with many small tenants |
| **Schema-level** | Separate schema per tenant, shared database | Medium | Medium tenants needing stronger isolation |
| **Database-level** | Separate database per tenant | High | Large tenants, compliance requirements |

### Row-Level Isolation (Most Common)

- Every table that contains tenant data has a `tenant_id` column
- Every query includes `WHERE tenant_id = :current_tenant`
- The current tenant is derived from the authenticated user's session
- Enforce at the data access layer so business logic can't accidentally skip the filter

**The cardinal rule:** A tenant must never see another tenant's data. This is the most common and most dangerous multi-tenancy bug. Enforce it at the lowest layer possible (database views, row-level security policies, or a data access wrapper that automatically adds the tenant filter).

**In practice:**
- Add `tenant_id` to every relevant table from the start. Retrofitting is painful.
- Create a database helper that automatically adds the tenant filter to every query
- Test with multiple tenants in your test suite. Include a test that explicitly verifies one tenant can't see another's data.

---

## Auth Middleware Design

### Where Auth Checks Live

```
Request → Auth Middleware → Route Handler → Business Logic → Data Access
          ↑                 ↑                ↑
          "Is there a       "Does this role  "Does this user
           valid session?"   have access?"    own this resource?"
```

- **Auth middleware:** Runs before every route. Extracts the session/token, validates it, attaches the user to the request context. Returns 401 if invalid.
- **Route-level guards:** Decorators or checks that verify the user's role. Returns 403 if unauthorized.
- **Business logic checks:** Ownership and attribute-based checks. Returns 403 with a specific message.

### Public vs Protected Endpoints

- Mark public endpoints explicitly (login, registration, health check, public API)
- Default to protected. Every new endpoint requires authentication unless explicitly marked public.
- For mixed endpoints (e.g., article list is public, but shows extra data for authenticated users): run auth middleware in "optional" mode (don't reject unauthenticated requests, but attach user data if available)

---

## "Remember Me" & Device Trust

### Long-Lived Sessions

"Remember me" extends the session beyond the default expiry (e.g., 30 days instead of 8 hours).

**In practice:**
- Use a separate, long-lived token stored in a persistent cookie (not the session cookie)
- When the session expires but the "remember me" token is valid: create a new session automatically
- The "remember me" token should be single-use (rotated on each use) to limit damage from theft
- Store these tokens hashed in the database (like passwords). If the database leaks, stolen tokens are useless.

### Re-Authentication for Sensitive Actions

Even with a valid session, some actions should require the password again:
- Changing the password
- Changing the email address
- Deleting the account
- Viewing or changing billing information
- Managing API keys

**Why:** If a session is hijacked, the attacker can browse but can't perform destructive or irreversible actions without the password.

---

## Testing Auth

### Testing Protected Endpoints

- Create a test helper that generates authenticated requests (pre-set session cookie or token)
- Test each endpoint with: no auth (expect 401), wrong role (expect 403), correct role (expect 200)
- Test token expiry: generate an expired token, verify it's rejected

### Testing Roles and Permissions

- Create test users with each role
- Verify that each role can only access its permitted endpoints
- Test edge cases: what happens when a role is changed while a session is active?

### Mocking Auth in Integration Tests

- For tests that aren't testing auth itself: bypass auth middleware entirely or inject a pre-authenticated user
- This keeps auth concerns from cluttering every test
- But always have dedicated auth tests that exercise the full flow

---

## Related Documents

- [06-security](../06-security.md) -- Password hashing properties, session cookie flags, CSRF protection
- [03-architecture](../03-architecture.md) -- Middleware patterns, cross-layer concerns
- [04-data-modeling](../04-data-modeling.md) -- Schema design for users, roles, sessions
- [05-ui-ux](../05-ui-ux.md) -- Login forms, first-run experience
