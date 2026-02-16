# A story about an application that kept getting blocked — and exactly why each time was different

---

## The Building with Many Rooms

Imagine a large company building. Inside are dozens of rooms, and each room holds something different. Some rooms are open to anyone with a staff badge. Some rooms require you to not only show your badge but also prove you are carrying a physical security key. And some rooms will not even let you in unless you scanned your fingerprint within the last ten minutes.

Now imagine a delivery courier who carries packages on behalf of staff members. The courier does not open doors themselves — they carry a signed pass issued by the front desk that lists exactly which rooms they are allowed to enter, on whose behalf, and what the staff member did to prove their identity before the pass was issued.

This is the world our story takes place in. Except instead of a building, it is a set of software systems. Instead of a courier, it is an application. And instead of a signed pass, it is a digital token — a small, signed document that travels with every request.

And today, the courier is going to get rejected. Three times. Each time for a completely different reason.

---

## Meet the People and Systems

### Alice — The Developer

Alice is a software engineer at a company. Every morning she opens a tool called the **Task Manager App** to organise her work — viewing tasks, assigning them to colleagues, and checking project history.

This morning, Alice opened the app at **9:00 AM** and logged in with her email and password. That is all she did. One factor. One password.

---

### The Task Manager App — The Courier

The Task Manager App is not a person. It is an application that acts on Alice's behalf. When she logs in, the company's login system issues it a **JWT access token** — a small, digitally signed document — that the app carries whenever it knocks on other systems' doors.

This token carries two types of information:
1. **What Alice is allowed to do** — her permissions, called *scopes*
2. **How Alice proved she was Alice** — the strength and method of her login

Here is what Alice's first token looks like when you decode it. Every field name here is defined in a published specification:

```json
{
  "iss": "https://wso2is.company.example.com:9443/oauth2/token",
  "sub": "alice-001",
  "aud": "project-repo-api",
  "scope": "openid profile tasks:read project:audit:read",
  "acr": "urn:company:acr:basic",
  "amr": ["pwd"],
  "auth_time": 1739444400,
  "iat": 1739444400,
  "exp": 1739448000,
  "jti": "abc-123"
}
```

What each claim means and where it comes from:

| Claim | Value | Plain meaning | Defined in |
|-------|-------|--------------|------------|
| `iss` | `https://wso2is.company.example.com:9443/oauth2/token` | Who issued this token | RFC 7519 |
| `sub` | `alice-001` | Who this token is about | RFC 7519 |
| `aud` | `project-repo-api` | Which system this token is for | RFC 7519 |
| `scope` | `tasks:read project:audit:read` | What Alice is allowed to access | RFC 9068 |
| `acr` | `urn:company:acr:basic` | Security level Alice reached at login | OpenID Connect Core 1.0 |
| `amr` | `["pwd"]` | Specific methods she used to prove identity | OpenID Connect Core 1.0 |
| `auth_time` | `1739444400` | Unix timestamp of her last active authentication | OpenID Connect Core 1.0 |
| `iat` | `1739444400` | When this token was issued | RFC 7519 |
| `exp` | `1739448000` | When this token expires | RFC 7519 |
| `jti` | `abc-123` | Unique ID for this token | RFC 7519 |

Two claims are the heart of this story:

**`acr`** — Authentication Context Class Reference. A label that summarises how strongly Alice proved she was Alice. Right now it is `urn:company:acr:basic`, meaning she used a password and nothing else. This is a URI-formatted string — the company defines its own ACR values as URIs, which is the standard practice.

**`amr`** — Authentication Methods References. The actual list of specific methods she used. Right now it is `["pwd"]` — just a password. The individual values (`pwd`, `otp`, `hwk`, etc.) are defined in **draft-ietf-oauth-amr-values-02**.

Think of `acr` as the grade on a certificate. Think of `amr` as the list of tests she actually sat to earn that grade.

The company has defined three ACR levels:

| `acr` value | Meaning | AMR values that satisfy it |
|-------------|---------|---------------------------|
| `urn:company:acr:basic` | Password login | `["pwd"]` |
| `urn:company:acr:mfa` | Password + one-time code | `["pwd", "otp"]` |
| `urn:company:acr:mfa-hardware` | Password + physical security key | `["pwd", "hwk"]` or `["pwd", "otp", "hwk"]` |

Alice is at `urn:company:acr:basic` right now. Her token is fresh, genuine, and ready. The courier steps out.
 
---

### The Project Repository API — The Room Guard

This is the company's internal system that Alice's app talks to. Every time the app knocks, the guard validates the token and checks it against the rules for that specific endpoint.

Different endpoints have different requirements:

| Endpoint | `scope` required | `acr` required | `max_age` (max seconds since `auth_time`) |
|----------|-----------------|----------------|-------------------------------------------|
| `GET /tasks` | `tasks:read` | `urn:company:acr:basic` | 3600 seconds (60 min) |
| `POST /tasks/{id}/assign` | `tasks:write` | `urn:company:acr:basic` | 3600 seconds (60 min) |
| `GET /projects/{id}/audit-log` | `project:audit:read` | `urn:company:acr:mfa` | 600 seconds (10 min) |

The `max_age` column is checked by comparing the current time against the token's `auth_time` claim. If the difference is greater than `max_age`, access is denied — the login is too old.

---

### The Login Server — WSO2 Identity Server

The company runs **WSO2 Identity Server** as its OAuth 2.0 / OpenID Connect Authorization Server. It is the only system allowed to issue or upgrade tokens. When something goes wrong with a token, the app must come back here.

WSO2 Identity Server is accessible at `https://wso2is.company.example.com:9443`. Its two key endpoints are:

| Endpoint | URL | Purpose |
|----------|-----|---------|
| Authorization endpoint | `https://wso2is.company.example.com:9443/oauth2/authorize` | Where users are sent to log in and consent |
| Token endpoint | `https://wso2is.company.example.com:9443/oauth2/token` | Where authorization codes and refresh tokens are exchanged for access tokens |

The Task Manager App is registered in WSO2 Identity Server as a **Service Provider** with `client_id=task-manager-app`. This registration defines the redirect URIs the app is allowed to use, the grant types it may request, and the scopes it is permitted to ask for.

WSO2 Identity Server also supports **Adaptive Authentication** — the ability to enforce step-up authentication dynamically based on context (which resource is being accessed, how sensitive it is, or how old the user's last login is). This is what makes the third rejection in our story possible: the server can require a stronger login specifically for the audit log endpoint without requiring it everywhere else.

---

## Part One: 9:14 AM — A Normal Start

Alice clicks "View My Tasks." The app sends her token to the Project Repository API:

```http
GET https://api.projectrepo.example.com/tasks
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
```

The room guard validates the token:

```
Verify RS256 signature using WSO2 IS public key        → valid ✓
Check exp (1739448000) against current time             → not expired ✓
Check aud ("project-repo-api")                          → matches ✓
Check scope contains "tasks:read"                       → present ✓
Check acr ("urn:company:acr:basic") against requirement → sufficient ✓
Check auth_time: current_time - 1739444400 = 840s < 3600s max_age → fresh enough ✓
```

The API responds with her tasks:

```json
{
  "tasks": [
    { "id": "t-1", "title": "Fix login bug",    "status": "in_progress" },
    { "id": "t-2", "title": "Write unit tests", "status": "pending" }
  ]
}
```

Everything works. Alice reviews her tasks and heads off to a morning meeting. The app sits quietly, waiting for her to come back.

---

## Part Two: 10:15 AM — The First Rejection (Token Expired)

Alice comes back from her meeting and clicks "Refresh Tasks." The app still has her token from 9:00 AM. It has been over an hour. The `exp` claim has passed.

The app sends the request:

```http
GET https://api.projectrepo.example.com/tasks
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
```

The room guard validates:

```
Verify RS256 signature  → valid ✓
Check exp (1739448000) against current time (1739451900)
  → 1739451900 > 1739448000 — EXPIRED ✗
```

The guard stops here. The token is expired. Under **RFC 6750 Section 3**, it returns:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="project-repo-api",
                  error="invalid_token",
                  error_description="The access token expired"

{
  "error": "invalid_token",
  "error_description": "The access token expired"
}
```

`invalid_token` means: **the token itself is broken — it does not matter what claims are in it. Get a new one.**

The `acr` and `amr` are not the issue. The `scope` is not the issue. The `exp` claim failed.

### The App Silently Gets a Replacement

The app holds a **refresh token** that was issued alongside the original access token. It uses it to get a new access token without interrupting Alice:

```http
POST https://wso2is.company.example.com:9443/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=8xLOxBtZp8
&client_id=task-manager-app
&client_secret=s3cr3t
```

WSO2 Identity Server validates the refresh token and issues a new access token:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9.NEW...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "9yMPyCufQ9",
  "scope": "openid profile tasks:read project:audit:read"
}
```

The new access token, decoded:

```json
{
  "iss": "https://wso2is.company.example.com:9443/oauth2/token",
  "sub": "alice-001",
  "aud": "project-repo-api",
  "scope": "openid profile tasks:read project:audit:read",
  "acr": "urn:company:acr:basic",
  "amr": ["pwd"],
  "auth_time": 1739444400,
  "iat": 1739451900,
  "exp": 1739455500,
  "jti": "abc-456"
}
```

Notice what did **not** change:
- `acr` is still `urn:company:acr:basic`
- `amr` is still `["pwd"]`
- `auth_time` is still `1739444400` — 9:00 AM, when Alice originally authenticated

This is the critical point: **refreshing a token does not re-authenticate Alice**. The Login Server extended the `exp` and updated `iat`, but `auth_time` stays the same because Alice has not proved her identity again — the Login Server is simply saying "here is a fresh token, but it still reflects the same login from 9:00 AM."

The app retries the request with the new token. It works. Alice never saw any error.

---

## Part Three: 10:31 AM — The Second Rejection (Missing Scope)

Alice's manager asks her to assign the "Fix login bug" task to her colleague Bob. She clicks "Assign Task." This is the first time the app has ever tried to assign a task on her behalf.

Her current token has `scope: "openid profile tasks:read project:audit:read"`. Assigning a task requires `tasks:write`, which is not on her token — she was never asked to consent to it.

The app sends:

```http
POST https://api.projectrepo.example.com/tasks/t-1/assign
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.NEW...
Content-Type: application/json

{
  "assignee_id": "bob-002"
}
```

The room guard validates:

```
Verify RS256 signature          → valid ✓
Check exp                       → not expired ✓
Check aud                       → matches ✓
Check scope contains "tasks:write"
  → scope is "tasks:read project:audit:read" — "tasks:write" NOT PRESENT ✗
Check acr                       → "urn:company:acr:basic" — would be sufficient ✓
                                   (but scope check already failed)
```

The token is valid. The `acr` is fine. But the `scope` claim does not include `tasks:write`. Under **RFC 6750 Section 3.1**, the guard returns:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer realm="project-repo-api",
                  error="insufficient_scope",
                  error_description="The request requires higher privileges than provided by the access token",
                  scope="tasks:write"

{
  "error": "insufficient_scope",
  "error_description": "The access token does not contain the required scope"
}
```

`insufficient_scope` means: **the token is valid and the authentication was fine, but the scope needed for this endpoint was never granted.** The `scope="tasks:write"` parameter in the `WWW-Authenticate` header tells the app exactly which scope is missing.

### The App Requests the Missing Scope

The app saves the original request and sends Alice to the Login Server to consent to the missing scope. This is **incremental authorization** — requesting an additional scope without starting from scratch.

```http
GET https://wso2is.company.example.com:9443/oauth2/authorize
  ?response_type=code
  &client_id=task-manager-app
  &redirect_uri=https://app.taskmanager.example.com/callback
  &scope=openid%20profile%20tasks%3Aread%20tasks%3Awrite%20project%3Aaudit%3Aread
  &state=consent_7f3k2p
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
  &prompt=consent
```

`prompt=consent` — defined in OpenID Connect Core 1.0 — forces the Login Server to show the consent screen even though Alice has an active session. Without it, the server might skip the screen if she has previously consented to a subset.

The Login Server shows Alice only the new permission:

```
+---------------------------------------------------------------+
|  Task Manager App is requesting additional access:            |
|                                                               |
|  Already approved:                                            |
|  [x] View your task list            (tasks:read)             |
|  [x] View project audit log         (project:audit:read)     |
|                                                               |
|  NEW — This app now wants to:                                |
|  [ ] Create and assign tasks        (tasks:write)            |
|                                                               |
|  [ Allow ]   [ Deny ]                                        |
+---------------------------------------------------------------+
```

Alice clicks **Allow**. WSO2 Identity Server issues a new access token. The app exchanges the authorization code:

```http
POST https://wso2is.company.example.com:9443/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=Qcb0Orv1XpHworNernFznA
&redirect_uri=https://app.taskmanager.example.com/callback
&client_id=task-manager-app
&client_secret=s3cr3t
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

New access token, decoded:

```json
{
  "iss": "https://wso2is.company.example.com:9443/oauth2/token",
  "sub": "alice-001",
  "aud": "project-repo-api",
  "scope": "openid profile tasks:read tasks:write project:audit:read",
  "acr": "urn:company:acr:basic",
  "amr": ["pwd"],
  "auth_time": 1739444400,
  "iat": 1739453460,
  "exp": 1739457060,
  "jti": "def-456"
}
```

**Comparing old token vs. new token:**

| Claim | Old token | New token |
|-------|-----------|-----------|
| `scope` | `tasks:read project:audit:read` | `tasks:read tasks:write project:audit:read` ← added |
| `acr` | `urn:company:acr:basic` | `urn:company:acr:basic` ← unchanged |
| `amr` | `["pwd"]` | `["pwd"]` ← unchanged |
| `auth_time` | `1739444400` (9:00 AM) | `1739444400` (9:00 AM) ← unchanged |
| `iat` | `1739451900` | `1739453460` ← new issuance time |
| `exp` | `1739455500` | `1739457060` ← new expiry |

Alice only clicked a consent button — she did not authenticate again. So `acr`, `amr`, and `auth_time` all stay exactly the same. The Login Server added `tasks:write` to the `scope` claim and issued a new token, but nothing about *how Alice proved her identity* changed.

The app retries the saved request. The room guard checks `scope` and finds `tasks:write` present. It succeeds:

```json
{
  "task_id": "t-1",
  "assignee_id": "bob-002",
  "assigned_at": "2026-02-16T10:31:00Z",
  "message": "Task successfully assigned to bob-002"
}
```

Alice sees: "Task assigned to Bob."

---

## Part Four: 11:02 AM — The Third Rejection (Authentication Not Strong Enough)

Alice's manager asks her to pull up the **Project Audit Log** — a sensitive record of every action ever taken in the project. She clicks "View Audit Log."

Her current token already has `project:audit:read` in the `scope` claim. That permission has been there since the beginning. And the token is still valid — `exp` has not passed.

The app sends:

```http
GET https://api.projectrepo.example.com/projects/proj-1/audit-log
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.def-456...
```

The room guard validates:

```
Verify RS256 signature                                → valid ✓
Check exp                                             → not expired ✓
Check aud                                             → matches ✓
Check scope contains "project:audit:read"             → present ✓
Check acr claim vs endpoint requirement:
  Token acr:       "urn:company:acr:basic"
  Required acr:    "urn:company:acr:mfa"
  → acr DOES NOT MEET REQUIREMENT ✗

Check auth_time for max_age:
  Current time:    1739451720  (11:02 AM)
  auth_time:       1739444400  (09:00 AM)
  Elapsed:         7320 seconds
  max_age:         600 seconds
  → 7320 > 600 — AUTHENTICATION TOO OLD ✗
```

This is different from both previous rejections. The token is valid. The `scope` is correct. But:
- The `acr` claim shows only `urn:company:acr:basic` (password only) — the audit log endpoint requires `urn:company:acr:mfa` (password + one-time code)
- The `auth_time` shows Alice last actively authenticated over two hours ago — the endpoint only allows logins within 600 seconds

The room guard returns a challenge defined by **RFC 9470 — OAuth 2.0 Step Up Authentication Challenge Protocol**:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="project-repo-api",
                  error="insufficient_user_authentication",
                  error_description="Audit log access requires MFA authentication within the last 10 minutes.",
                  acr_values="urn:company:acr:mfa",
                  max_age="600"

{
  "error": "insufficient_user_authentication",
  "error_description": "Audit log requires MFA login within the last 10 minutes."
}
```

The `WWW-Authenticate` header carries two parameters defined by RFC 9470:
- **`acr_values`** — the ACR value the token's `acr` claim must satisfy
- **`max_age`** — the maximum allowed number of seconds since the token's `auth_time` claim

### How This Differs from the Other Two Rejections

| Rejection | Error | Token valid? | `scope` present? | `acr` sufficient? | Root cause |
|-----------|-------|-------------|-----------------|------------------|------------|
| 10:15 AM | `invalid_token` | NO | — | — | `exp` exceeded |
| 10:31 AM | `insufficient_scope` | YES | NO | YES | `tasks:write` not in `scope` |
| 11:02 AM | `insufficient_user_authentication` | YES | YES | NO | `acr` too weak, `auth_time` too old |

A useful analogy: think of a bank.
- `invalid_token` = your card is expired. The machine will not even read it.
- `insufficient_scope` = your account does not have international transfers enabled. Call the bank and enable it.
- `insufficient_user_authentication` = your account has international transfers enabled, but to execute one over a certain amount you must verify with your banking app — and that verification must have happened within the last 10 minutes.

### The App Reads the Challenge

The app reads the rejection. It sees `error="insufficient_user_authentication"` — this is **not** a scope problem, so it does not go to a consent screen. This is an authentication strength problem, defined by RFC 9470, so it must redirect Alice to WSO2 Identity Server to re-authenticate at a higher level.

The `acr_values` and `max_age` parameters from the `WWW-Authenticate` header are passed directly into the new authorization request — exactly as RFC 9470 specifies. WSO2 Identity Server's Adaptive Authentication engine reads these parameters and determines what step-up challenge to present:

```http
GET https://wso2is.company.example.com:9443/oauth2/authorize
  ?response_type=code
  &client_id=task-manager-app
  &redirect_uri=https://app.taskmanager.example.com/callback
  &scope=openid%20profile%20tasks%3Aread%20tasks%3Awrite%20project%3Aaudit%3Aread
  &acr_values=urn%3Acompany%3Aacr%3Amfa
  &max_age=600
  &state=stepup_kP9mR2qX
  &nonce=nonce_7Rp3mKqZ
  &code_challenge=S5CzMhYLBdPSZiCpR3XGWkqg2Yx6NnTeDfaJIuAoE0Y
  &code_challenge_method=S256
  &prompt=login
```

Key parameters:
- `acr_values` — taken directly from the challenge; tells the Login Server what `acr` level must be reached
- `max_age` — taken directly from the challenge; the Login Server must ensure `auth_time` is within 600 seconds
- `prompt=login` — OpenID Connect Core 1.0 parameter; forces a fresh authentication even if Alice has an active session
- `nonce` — binds the resulting ID Token to this specific request, preventing replay attacks

### Alice Re-Authenticates with a One-Time Code

The Login Server checks Alice's current session. Her `auth_time` is from 9:00 AM and her session `acr` is `urn:company:acr:basic` — this does not satisfy `acr_values=urn:company:acr:mfa` or `max_age=600`. It skips her password (already known) and challenges her for the second factor:

```
+--------------------------------------------------------------------+
|   Company Identity Provider                                        |
|                                                                    |
|   Additional Verification Required                                 |
|                                                                    |
|   Task Manager App is requesting access to the Project Audit Log.  |
|   This action requires a second factor of authentication.          |
|                                                                    |
|   Your password is already verified.                               |
|                                                                    |
|   Enter the 6-digit code from your authenticator app:             |
|                                                                    |
|   Code: [ __ __ __ __ __ __ ]                                      |
|                                                                    |
|   [ Verify ]                                                       |
|                                                                    |
+--------------------------------------------------------------------+
```

Alice enters the code. The Login Server verifies it. It records the updated authentication:

```json
{
  "sub": "alice-001",
  "auth_time": 1739451750,
  "acr": "urn:company:acr:mfa",
  "amr": ["pwd", "otp"]
}
```

The `amr` list now contains two values, both defined in **draft-ietf-oauth-amr-values-02**:

| AMR value | Meaning from spec | What Alice did |
|-----------|------------------|----------------|
| `pwd` | Password-based authentication | Her password from 9:00 AM |
| `otp` | One-time password | The 6-digit code from her authenticator app |

### A New Token Is Issued — Shorter-Lived

The Login Server redirects Alice back with an authorization code. The app exchanges it:

```http
POST https://idp.company.example.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=StepUpCode_auditLog_9Xp
&redirect_uri=https://app.taskmanager.example.com/callback
&client_id=task-manager-app
&client_secret=s3cr3t
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

New access token, decoded:

```json
{
  "iss": "https://idp.company.example.com",
  "sub": "alice-001",
  "aud": "project-repo-api",
  "scope": "openid profile tasks:read tasks:write project:audit:read",
  "acr": "urn:company:acr:mfa",
  "amr": ["pwd", "otp"],
  "auth_time": 1739451750,
  "iat": 1739451750,
  "exp": 1739452350,
  "jti": "ghi-789"
}
```

**Comparing old token vs. step-up token:**

| Claim | Old token | Step-up token |
|-------|-----------|---------------|
| `scope` | `tasks:read tasks:write project:audit:read` | same ← unchanged |
| `acr` | `urn:company:acr:basic` | `urn:company:acr:mfa` ← upgraded |
| `amr` | `["pwd"]` | `["pwd", "otp"]` ← upgraded |
| `auth_time` | `1739444400` (9:00 AM) | `1739451750` (11:02 AM) ← fresh |
| `iat` | `1739453460` | `1739451750` |
| `exp` | `1739457060` | `1739452350` — expires in **600 seconds** ← much shorter |

The `scope` did not change. The permissions Alice holds are the same as before. Only `acr`, `amr`, and `auth_time` changed — because Alice re-authenticated. And the token's `exp` is set to match the `max_age` requirement: 600 seconds from now.

### The App Retries — Success

```http
GET https://api.projectrepo.example.com/projects/proj-1/audit-log
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.ghi-789...
```

The room guard validates:

```
Verify RS256 signature                       → valid ✓
Check exp                                    → not expired ✓
Check aud                                    → matches ✓
Check scope contains "project:audit:read"    → present ✓
Check acr: "urn:company:acr:mfa"
  vs required "urn:company:acr:mfa"          → meets requirement ✓
Check auth_time for max_age:
  Current time: 1739451798
  auth_time:    1739451750
  Elapsed:      48 seconds < 600 seconds     → fresh enough ✓
```

The API returns the audit log:

```json
{
  "project_id": "proj-1",
  "audit_log": [
    { "action": "branch_created", "by": "alice-001", "at": "2026-02-16T08:45:00Z" },
    { "action": "task_assigned",  "by": "alice-001", "at": "2026-02-16T10:31:00Z" },
    { "action": "config_changed", "by": "bob-002",   "at": "2026-02-16T10:55:00Z" }
  ]
}
```

Alice sees the audit log.

---

## The Full Picture — One Day, Three Blocks, Three Different Causes

```
Alice          Task Manager App       Project Repo API       Login Server
  |                   |                      |                     |
  | 9:00 AM — logs in with password          |                     |
  |---> sent to login server for auth --------------------------> |
  |<--- token: acr=basic, amr=["pwd"], auth_time=9:00 ---------  |
  |------------------>|                      |                     |
  |                   |                      |                     |
  | 9:14 AM — reads tasks                    |                     |
  |                   |-- Bearer token ------>|                     |
  |                   |   scope=tasks:read ✓  |                     |
  |                   |   acr=basic ✓         | all checks pass ✓  |
  |                   |<-- 200 tasks ----------|                     |
  |<-- shows tasks ---|                      |                     |
  |                   |                      |                     |
  | 10:15 AM — token exp has passed          |                     |
  |                   |-- Bearer token ------>|                     |
  |                   |<-- 401 invalid_token  | exp exceeded ✗     |
  |                   |--- refresh_token ---------------------------> |
  |                   |<-- new token: acr=basic, amr=["pwd"]        |
  |                   |    auth_time=9:00 unchanged                 |
  |                   |-- new token --------->| exp ok ✓           |
  |<-- shows tasks (no error shown to Alice)  |                    |
  |                   |                      |                     |
  | 10:31 AM — assigns task                  |                     |
  |                   |-- Bearer token ------>|                     |
  |                   |   scope missing       |                     |
  |                   |   tasks:write ✗       |                     |
  |                   |<-- 403 insufficient_scope                   |
  |                   |   scope="tasks:write"                       |
  |                   | [saves request]       |                     |
  |<-- redirect: /authorize?scope=tasks:write&prompt=consent ----  |
  |-- GET /authorize ------------------------------------------>  |
  |<-- consent screen: "Allow tasks:write?"                        |
  | clicks Allow                                                   |
  |-- POST consent ------------------------------------------->   |
  |<-- auth code                                                   |
  |   POST /token (code exchange)                              ->  |
  |                   |<-- new token: scope+=tasks:write            |
  |                   |    acr=basic UNCHANGED                      |
  |                   |    amr=["pwd"] UNCHANGED                    |
  |                   |    auth_time=9:00 UNCHANGED                 |
  |                   |-- new token --------->| tasks:write ✓      |
  |<-- "Task assigned!"|                     |                     |
  |                   |                      |                     |
  | 11:02 AM — tries to view audit log       |                     |
  |                   |-- Bearer token ------>|                     |
  |                   |   scope ok ✓          |                     |
  |                   |   acr=basic           | needs acr=mfa ✗    |
  |                   |   auth_time=9:00      | max_age=600 ✗      |
  |                   |<-- 401 insufficient_user_authentication     |
  |                   |   acr_values="urn:company:acr:mfa"         |
  |                   |   max_age="600"                            |
  |                   | [saves request]       |                     |
  |<-- redirect: /authorize?acr_values=mfa&max_age=600&prompt=login|
  |-- GET /authorize ------------------------------------------>  |
  |<-- MFA prompt (enter OTP code)                                 |
  | enters 6-digit OTP                                            |
  |-- POST OTP ------------------------------------------->       |
  |<-- auth code                                                   |
  |   POST /token (code exchange)                              ->  |
  |                   |<-- new token: acr=mfa, amr=["pwd","otp"]   |
  |                   |    scope UNCHANGED                          |
  |                   |    auth_time=11:02 (fresh)                  |
  |                   |    exp = now + 600s                         |
  |                   |-- new token --------->| acr=mfa ✓          |
  |                   |                       | auth_time: 48s ago ✓|
  |                   |<-- 200 audit log -----|                     |
  |<-- shows audit log|                      |                     |
```

---

## What Changed in the Token Each Time

```
After Problem 1 — token expired (10:15 AM):
  acr:       urn:company:acr:basic   → urn:company:acr:basic   (unchanged)
  amr:       ["pwd"]                 → ["pwd"]                  (unchanged)
  auth_time: 1739444400              → 1739444400               (unchanged)
  scope:     tasks:read              → tasks:read                (unchanged)
  exp:       1739448000              → 1739455500               (extended)

After Problem 2 — scope missing (10:31 AM):
  acr:       urn:company:acr:basic   → urn:company:acr:basic   (unchanged)
  amr:       ["pwd"]                 → ["pwd"]                  (unchanged)
  auth_time: 1739444400              → 1739444400               (unchanged)
  scope:     tasks:read              → tasks:read + tasks:write  (added)

After Problem 3 — auth not strong enough (11:02 AM):
  acr:       urn:company:acr:basic   → urn:company:acr:mfa      (upgraded)
  amr:       ["pwd"]                 → ["pwd", "otp"]           (upgraded)
  auth_time: 1739444400              → 1739451750               (fresh)
  scope:     tasks:read tasks:write  → tasks:read tasks:write    (unchanged)
  exp:       1739457060              → 1739452350               (600s window)
```

Three problems. Each one changed only the claim that was broken. Nothing else moved.

---

## How the App Decides What to Do

```
API returns an error response
          |
    HTTP status code?
          |                    |
         401                  403
          |                    |
    Read WWW-Authenticate   Read WWW-Authenticate
          |                    |
    error value?           error="insufficient_scope"?
          |                    |              |
    +-----+-------+           YES             NO
    |             |            |         True access denial
"invalid_   "insufficient_  Extract       (business logic,
  token"     user_auth"     missing        not a token
    |             |          scope          problem)
  Try         Read from      value          Show error
  refresh     WWW-Auth:          |
  token       acr_values    Send to Login
  silently    max_age       Server:
    |             |         prompt=consent
  Works?      Send to       scope includes
  |     |     Login Server: missing scope
 YES    NO    prompt=login       |
  |     |     acr_values    Alice consents?
 New   Full   max_age        |          |
 token  re-       |         YES         NO
 retry  auth  Alice           |          |
        flow  verifies    New token  Show
              (OTP/hwk)   scope      "permission
                  |       added      needed"
              New token:  Retry
              acr upgraded
              amr upgraded
              auth_time fresh
              exp = max_age window
              Retry
```

---

## What the Audit Log Records

The Project Repository API writes a log entry for every successful access, always including the `acr` and `amr` claims from the token that was used:

```json
[
  {
    "event": "tasks_read",
    "timestamp": "2026-02-16T09:14:00Z",
    "sub": "alice-001",
    "client_id": "task-manager-app",
    "acr": "urn:company:acr:basic",
    "amr": ["pwd"],
    "auth_time": "2026-02-16T09:00:00Z",
    "decision": "PERMIT"
  },
  {
    "event": "task_assigned",
    "timestamp": "2026-02-16T10:31:00Z",
    "sub": "alice-001",
    "client_id": "task-manager-app",
    "acr": "urn:company:acr:basic",
    "amr": ["pwd"],
    "auth_time": "2026-02-16T09:00:00Z",
    "decision": "PERMIT"
  },
  {
    "event": "audit_log_read",
    "timestamp": "2026-02-16T11:02:30Z",
    "sub": "alice-001",
    "client_id": "task-manager-app",
    "acr": "urn:company:acr:mfa",
    "amr": ["pwd", "otp"],
    "auth_time": "2026-02-16T11:02:10Z",
    "decision": "PERMIT"
  }
]
```

A security team can read this log and see not just who did what and when, but exactly how strongly each person proved their identity for each action — right down to the specific methods in the `amr` claim.

---

## What This Story Teaches

**One: Having the right scope is not the same as proving yourself strongly enough.**
Alice had `project:audit:read` in her `scope` from the beginning. But the endpoint checked `acr` and `auth_time` independently — and both failed. Permission and authentication strength are two separate dimensions of access control.

**Two: Each rejection is a precise, machine-readable signal.**
The room guard did not just say "no." It said exactly why, using error codes defined in published standards, and in the case of RFC 9470 it told the app exactly what `acr_values` and `max_age` are needed. The app could respond correctly without any guesswork.

**Three: Only the broken thing gets fixed — everything else stays the same.**
When scope was missing: only `scope` changed in the new token. When `acr` was insufficient: only `acr`, `amr`, and `auth_time` changed. The system is surgical.

**Four: Stronger access means a shorter window.**
The audit log token `exp` was set to `auth_time + 600` — exactly matching the `max_age` the endpoint required. High-assurance tokens are short-lived by design. A stolen token from a high-security operation has a small window to do damage.

**Five: `acr` and `amr` travel with every token, in every request, into every audit log.**
Whether or not the endpoint checked them, they were always there. They are the permanent, verifiable record of how identity was proved — not a policy note, but a cryptographically signed claim in the token itself.

---

*End of Story*

---

**Specifications referenced in this story:**

| Standard | What it defines |
|----------|----------------|
| **RFC 7519** | JWT structure and claims: `iss`, `sub`, `aud`, `exp`, `iat`, `jti` |
| **RFC 6749** | OAuth 2.0 authorization framework: scopes, grant types, token endpoint |
| **RFC 6750** | Bearer token usage and error responses: `invalid_token`, `insufficient_scope`, `WWW-Authenticate` header |
| **RFC 9068** | JWT Profile for OAuth 2.0 Access Tokens: `scope` claim in access tokens |
| **RFC 9470** | OAuth 2.0 Step Up Authentication Challenge Protocol: `insufficient_user_authentication`, `acr_values`, `max_age` in `WWW-Authenticate` |
| **RFC 7636** | PKCE: `code_challenge`, `code_verifier` |
| **OpenID Connect Core 1.0** | `acr`, `amr`, `auth_time`, `nonce` claims; `prompt` parameter; `acr_values` and `max_age` in authorization requests |
| **draft-ietf-oauth-amr-values-02** | AMR value registry: `pwd`, `otp`, `hwk`, `swk`, `mfa`, and others |
