# The Scope Challenge — Now with Identity Proof

### The same story of 401 and 403, but this time we also track *how* Alice proved who she was

---

## Before We Begin: Two Different Questions

Every time an application tries to access a protected resource, the system is actually asking two separate questions:

**Question 1 — What are you allowed to do?**
This is the **scope** question. Does your pass say you can read tasks? Write tasks? Access secrets?

**Question 2 — How strongly did you prove you are you?**
This is the **identity proof** question. Did you just type a password? Or did you also use a second factor, like a phone app or a physical key?

The original story only showed Question 1 being checked. This version shows both questions being checked at every step — using the **ACR** (security level label) and **AMR** (list of methods used) values that travel inside every digital pass.

---

## Quick Reference: What ACR and AMR Mean

**AMR — Authentication Method References**
The list of specific things Alice did to prove her identity. Each method has a short code:

| Code | What it means |
|------|--------------|
| `pwd` | She typed her password |
| `otp` | She entered a one-time code (from an app or SMS) |
| `hwk` | She used a physical hardware key (like a YubiKey) |
| `mfa` | She used more than one method (general label) |

**ACR — Authentication Context Class Reference**
A single label that summarises what security level she reached:

| Label | Meaning | What achieves it |
|-------|---------|-----------------|
| `urn:company:acr:basic` | Basic login | Just a password — `["pwd"]` |
| `urn:company:acr:mfa` | Two-factor | Password + one-time code — `["pwd", "otp"]` |
| `urn:company:acr:mfa-hardware` | Hardware MFA | Password + physical key — `["pwd", "hwk"]` |

Think of **ACR** as the grade on a certificate and **AMR** as the list of tests she actually sat.

---

## The Cast of Characters

### Alice — The User

Alice is a software engineer. She uses the **Task Manager App** to organise her work. The app is connected to her company's **Project Repository API**.

When she first logs in today, she uses her **email and password** — nothing else. So her identity proof record starts like this:

```json
{
  "who": "alice-001",
  "logged_in_at": "09:00:00",
  "security_level": "urn:company:acr:basic",
  "how_she_proved_identity": ["pwd"]
}
```

`"how_she_proved_identity"` is the **AMR** list. `"security_level"` is the **ACR** label. Both are stamped onto every digital pass (token) she gets.

---

### The Task Manager App — The Client Application

The app holds a digital pass for Alice. Here is what that pass looks like on the inside:

```json
{
  "issued_by": "https://idp.company.example.com",
  "belongs_to": "alice-001",
  "intended_for": "project-repo-api",
  "what_she_can_access": "openid profile tasks:read",
  "security_level_acr": "urn:company:acr:basic",
  "how_she_proved_identity_amr": ["pwd"],
  "logged_in_at": 1739444400,
  "expires_at": 1739448000,
  "pass_id": "abc-123"
}
```

Notice the two identity proof fields sitting alongside the permissions (scope). They travel everywhere the pass goes.

---

### The Project Repository API — The Gatekeeper

This is the system Alice's app talks to. It checks every pass against two things:

1. **Does this pass include the right permissions?** (scope check)
2. **Was this pass earned with a strong enough login?** (ACR check)

Today's story mainly features the scope check — but the ACR and AMR values are visible the whole time, which is what this version adds.

---

### The Login Server — The Issuer

The company's login system. It is the only one that can issue or renew passes. It lives at `https://idp.company.example.com`.

---

## Chapter 1: Alice Logs In — The Pass Is Created

Alice opens the Task Manager App and logs in with her password. The app sends her to the Login Server:

```
GET https://idp.company.example.com/authorize
  ?response_type=code
  &client_id=task-manager-app
  &redirect_uri=https://app.taskmanager.example.com/callback
  &scope=openid profile tasks:read
  &state=xK9mP2qR
```

The Login Server shows Alice a login page. She types her password. Then it shows her a consent screen:

```
+---------------------------------------------------------------+
|  Task Manager App is requesting access to:                    |
|                                                               |
|  [x] Your name and profile picture  (profile)                |
|  [x] View your task list            (tasks:read)             |
|                                                               |
|  [ Allow ]   [ Deny ]                                        |
+---------------------------------------------------------------+
```

Alice clicks **Allow**. The Login Server records her consent and what she will be allowed to do:

```json
{
  "user": "alice-001",
  "app": "task-manager-app",
  "permissions_granted": ["openid", "profile", "tasks:read"],
  "consent_recorded_at": "2026-02-16T09:00:00Z"
}
```

It redirects Alice back to the app with a short-lived code. The app exchanges that code for a digital pass:

```http
POST https://idp.company.example.com/token

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&client_id=task-manager-app
&client_secret=s3cr3t
```

The Login Server responds with the pass:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "8xLOxBtZp8",
  "scope": "openid profile tasks:read"
}
```

Inside that access pass, the full picture looks like this:

```json
{
  "issued_by": "https://idp.company.example.com",
  "belongs_to": "alice-001",
  "intended_for": "project-repo-api",
  "what_she_can_access": "openid profile tasks:read",
  "security_level_acr": "urn:company:acr:basic",
  "how_she_proved_identity_amr": ["pwd"],
  "logged_in_at": 1739444400,
  "expires_at": 1739448000,
  "pass_id": "abc-123"
}
```

**What this tells us:**
- She can read tasks (`tasks:read`) — but not write or assign them
- She proved her identity with just a password (`"pwd"`)
- Her security level is `basic`

Alice is now logged in. The app works normally.

---

## Chapter 2: Normal Use — Reading Tasks

Alice clicks "View My Tasks." The app sends her pass to the Project Repository API:

```http
GET https://api.projectrepo.example.com/tasks
Authorization: Bearer [Alice's pass]
```

The API checks the pass:

```
Is the pass genuine?           → Yes ✓
Is it still valid?             → Yes, not expired ✓
Is it meant for us?            → Yes ✓
Does she have tasks:read?      → Yes ✓
Security level required?       → Basic is fine for reading tasks ✓
ACR on pass:                   → "basic" — meets the requirement ✓
```

The API sends back her tasks:

```json
{
  "tasks": [
    { "id": "t-1", "title": "Fix login bug", "status": "in_progress" },
    { "id": "t-2", "title": "Write unit tests", "status": "pending" }
  ]
}
```

Everything works. Alice is happy. Notice the ACR check passed without any fuss — reading tasks only needs basic security level, and that is exactly what her password-only login gave her.

---

## Chapter 3: The First Problem — An Expired Pass (401)

A few hours later, Alice comes back from a meeting. Her pass has expired. The app still has the old pass in memory and uses it when she clicks "Refresh Tasks":

```http
GET https://api.projectrepo.example.com/tasks
Authorization: Bearer [expired pass]
```

The API checks the pass:

```
Is the pass genuine?  → Yes, signature valid
Is it still valid?    → NO — it expired one hour ago ✗
```

The pass itself is broken. The API sends a rejection:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="project-repo-api",
                  error="invalid_token",
                  error_description="The access token expired"
```

The error `invalid_token` means: **"This pass does not work at all — get a new one."**

This is not a scope problem. It is not an identity proof problem. The pass is simply too old.

### The App Silently Gets a Fresh Pass

The app uses its long-lived refresh pass to ask the Login Server for a replacement:

```http
POST https://idp.company.example.com/token

grant_type=refresh_token
&refresh_token=8xLOxBtZp8
&client_id=task-manager-app
&client_secret=s3cr3t
```

The Login Server checks that the refresh pass is still valid and issues a fresh access pass:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9.NEW_TOKEN...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "9yMPyCufQ9",
  "scope": "openid profile tasks:read"
}
```

The new pass looks like this inside:

```json
{
  "issued_by": "https://idp.company.example.com",
  "belongs_to": "alice-001",
  "intended_for": "project-repo-api",
  "what_she_can_access": "openid profile tasks:read",
  "security_level_acr": "urn:company:acr:basic",
  "how_she_proved_identity_amr": ["pwd"],
  "logged_in_at": 1739444400,
  "expires_at": 1739455800,
  "pass_id": "abc-456"
}
```

**Important — notice what did NOT change:**
- The ACR is still `basic`
- The AMR still shows only `["pwd"]`
- The permissions (scope) are still the same

Refreshing a pass does not upgrade identity proof. The new pass still knows she only used a password. The Login Server just extended the clock — it did not ask her to re-verify her identity.

The app retries the request with the new pass. It works. Alice never saw any error. The 401 was handled silently in the background.

---

## Chapter 4: The Second Problem — Missing Permission (403)

This is the main event. Alice's colleague Bob has just been promoted. The company wants Alice to use the Task Manager App to **assign tasks to team members**. She clicks the "Assign Task" button for the first time.

Her current pass only has `tasks:read`. Writing and assigning tasks requires `tasks:write`. She was never asked to grant that permission, so it is not on her pass.

The app sends the request:

```http
POST https://api.projectrepo.example.com/tasks/t-1/assign
Authorization: Bearer [Alice's pass — has tasks:read only]
Content-Type: application/json

{
  "assignee_id": "bob-002"
}
```

The API checks the pass:

```
Is the pass genuine?             → Yes ✓
Is it still valid?               → Yes ✓
Is it meant for us?              → Yes ✓
Does she have tasks:write?       → NO ✗ — pass only has tasks:read
Security level (ACR) on pass?    → "basic"
ACR required for this action?    → "basic" is fine for write operations here ✓

→ The pass is valid. The identity is proved strongly enough.
  But she was NEVER GIVEN PERMISSION to write tasks.
```

This is a **permission problem**, not an identity proof problem. Her ACR is fine. But her scope is wrong.

The API sends back a very specific rejection:

```http
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer realm="project-repo-api",
                  error="insufficient_scope",
                  error_description="The request requires higher privileges than provided by the access token",
                  scope="tasks:write"
```

`insufficient_scope` means: **"Your pass is real and your login was fine — but your pass was never given this permission. You need to go ask for it."**

The `scope="tasks:write"` field is the most important part. It tells the app exactly which permission is missing.

---

## Understanding the Three Rejections Side by Side

At this point we have seen two of the three types of rejection. Here they all are together:

| What went wrong | Error code | Is the pass real? | Is the login strong enough? | Is the permission there? |
|-----------------|-----------|------------------|---------------------------|-------------------------|
| Pass is expired | `invalid_token` | broken | n/a | n/a |
| Permission never granted | `insufficient_scope` | yes ✓ | yes ✓ | NO ✗ |
| Login was not strong enough | `insufficient_user_authentication` | yes ✓ | NO ✗ | yes ✓ |

This chapter is about the middle row — the scope problem. The identity proof (ACR/AMR) is fine. The permission is the issue.

---

## Chapter 5: The App Requests the Missing Permission

The app reads the 403 rejection and sees:
- `error = "insufficient_scope"` — permission problem, not an identity problem
- `scope = "tasks:write"` — this exact permission is what is missing

It saves Alice's original request so it can retry it later:

```json
{
  "saved_request": {
    "action": "POST /tasks/t-1/assign",
    "body": { "assignee_id": "bob-002" },
    "missing_permission": "tasks:write"
  }
}
```

Then it sends Alice to the Login Server to grant the new permission. This is called **incremental authorisation** — asking for more permissions on top of what was already granted, without starting from scratch.

The app asks for all the permissions it needs — the ones already granted, plus the new one:

```
GET https://idp.company.example.com/authorize
  ?response_type=code
  &client_id=task-manager-app
  &redirect_uri=https://app.taskmanager.example.com/callback
  &scope=openid profile tasks:read tasks:write
  &state=newState_7f3k2p
  &prompt=consent
```

Notice: `prompt=consent` forces the Login Server to show Alice a consent screen even though she already has a session. Without this, the Login Server might skip straight past the consent step.

---

## Chapter 6: Alice Grants the New Permission

Alice's browser arrives at the Login Server. The Login Server looks up her session — she is already logged in (her password login from this morning is still active). It shows her only the **new** permission she has not yet granted:

```
+---------------------------------------------------------------+
|  Task Manager App is requesting additional access:            |
|                                                               |
|  Already granted:                                             |
|  [x] Your name and profile picture  (profile)               |
|  [x] View your task list            (tasks:read)             |
|                                                               |
|  NEW — This app now wants to:                                |
|  [ ] Create and assign tasks        (tasks:write)            |
|                                                               |
|  [ Allow ]   [ Deny ]                                        |
+---------------------------------------------------------------+
```

Alice reads it and clicks **Allow**.

Notice something important here: **the Login Server does not ask Alice to re-enter her password**. It did not ask her to verify her identity again. It already has her session from this morning. It only needed her to approve the new permission.

This means the ACR and AMR values on the new pass will stay exactly the same as before — because she did not do any new authentication. She only gave new consent.

The Login Server updates its record:

```json
{
  "user": "alice-001",
  "app": "task-manager-app",
  "permissions_granted": ["openid", "profile", "tasks:read", "tasks:write"],
  "consent_recorded_at": "2026-02-16T10:30:00Z"
}
```

---

## Chapter 7: A New Pass Is Issued with the Extra Permission

The Login Server sends Alice back to the app with a code. The app exchanges it for a new pass:

```http
POST https://idp.company.example.com/token

grant_type=authorization_code
&code=Qcb0Orv1XpHworNernFznA
&redirect_uri=https://app.taskmanager.example.com/callback
&client_id=task-manager-app
&client_secret=s3cr3t
```

The Login Server issues the upgraded pass:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9.UPGRADED_TOKEN...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "Xk3pQlmN2w",
  "scope": "openid profile tasks:read tasks:write"
}
```

Inside the new pass:

```json
{
  "issued_by": "https://idp.company.example.com",
  "belongs_to": "alice-001",
  "intended_for": "project-repo-api",
  "what_she_can_access": "openid profile tasks:read tasks:write",
  "security_level_acr": "urn:company:acr:basic",
  "how_she_proved_identity_amr": ["pwd"],
  "logged_in_at": 1739444400,
  "expires_at": 1739455800,
  "pass_id": "def-456"
}
```

**Side-by-side — old pass vs. new pass:**

| Field | Old Pass | New Pass |
|-------|----------|----------|
| Permissions (scope) | `tasks:read` | `tasks:read tasks:write` |
| Security level (ACR) | `urn:company:acr:basic` | `urn:company:acr:basic` ← unchanged |
| Identity methods (AMR) | `["pwd"]` | `["pwd"]` ← unchanged |
| Login time | 09:00 | 09:00 ← unchanged (same session) |

**This is the key difference between a scope challenge and a step-up challenge:**

- A **scope challenge** (like this one) adds new **permissions** to the pass. The ACR and AMR stay the same, because Alice's identity proof did not change — she only clicked a consent button.
- A **step-up challenge** (like the secrets example in the other story) changes the **ACR and AMR** on the pass. The permissions stay the same, but Alice has to re-prove her identity more strongly.

In simple terms:
- Scope challenge = "Can I add this room to your key card?" — Alice just has to say yes
- Step-up challenge = "To enter this room, you need a stronger key card" — Alice has to re-verify

---

## Chapter 8: The App Replays the Original Request

The app retrieves the saved request from Chapter 5 and sends it again with the new pass:

```http
POST https://api.projectrepo.example.com/tasks/t-1/assign
Authorization: Bearer [new pass with tasks:write]
Content-Type: application/json

{
  "assignee_id": "bob-002"
}
```

The API checks the pass:

```
Is the pass genuine?             → Yes ✓
Is it still valid?               → Yes ✓
Is it meant for us?              → Yes ✓
Does she have tasks:write?       → YES ✓ — now present on the pass
Security level (ACR) on pass?    → "basic"
ACR required for this action?    → "basic" is sufficient ✓
AMR on pass?                     → ["pwd"] — noted in audit log
```

The API responds with success:

```json
{
  "task_id": "t-1",
  "assignee_id": "bob-002",
  "assigned_at": "2026-02-16T10:31:00Z",
  "message": "Task successfully assigned to bob-002"
}
```

Alice sees: "Task assigned to Bob." The whole flow was handled gracefully. She only had to click Allow once on the consent screen.

---

## Chapter 9: What Gets Written to the Audit Log

The API records what happened, including the ACR and AMR values:

```json
{
  "event": "task_assigned",
  "when": "2026-02-16T10:31:00Z",
  "what": "task t-1 assigned to bob-002",
  "who": "alice-001",
  "which_app": "task-manager-app",
  "how_identity_was_proven": {
    "security_level": "urn:company:acr:basic",
    "methods_used": ["pwd"],
    "original_login_at": "2026-02-16T09:00:00Z"
  },
  "decision": "ALLOWED"
}
```

A security team reviewing this log can see:
- Alice assigned the task
- She proved her identity with just a password (`"pwd"`)
- Her login was from earlier that morning, not from right before this action

If the company later decides that assigning tasks should require MFA, this audit log makes it easy to identify all past assignments that were done with only password-level authentication.

---

## Chapter 10: What If Alice Clicks Deny?

Going back to Chapter 6 — what if Alice does not want to give the app permission to assign tasks?

She clicks **Deny**. The Login Server sends her back to the app with an error:

```
GET https://app.taskmanager.example.com/callback
  ?error=access_denied
  &error_description=The+user+denied+the+request
  &state=newState_7f3k2p
```

The app clears the saved request and shows Alice a clear message:

```
"You need to grant 'Assign Tasks' permission for this feature to work.
 Your other features — like viewing tasks — continue to work normally."
```

Nothing breaks. Her existing pass with `tasks:read` still works fine. Only the assign feature is unavailable.

---

## Chapter 11: The Full Journey — From Start to Success

```
Alice           Task Manager App      Project Repo API      Login Server
  |                    |                    |                    |
  | logs in (password) |                    |                    |
  |-------------------> sends to login server ----------------->|
  |<---------------------- consent screen ---------------------- |
  | clicks Allow        |                    |                    |
  |--------------------> confirms consent ---------------------->|
  |<-- pass issued: scope=tasks:read, acr=basic, amr=[pwd] ---- |
  |-------------------->|                    |                    |
  |                      |                    |                    |
  | [reads tasks]        |                    |                    |
  |                      |-- pass: tasks:read ->                  |
  |                      |   acr=basic, amr=[pwd]                 |
  |                      |                    | all checks pass ✓  |
  |                      |<--- tasks list -----|                    |
  |<-- shows tasks ------|                    |                    |
  |                      |                    |                    |
  | [pass expires later] |                    |                    |
  |                      |-- old pass -------> EXPIRED ✗          |
  |                      |<-- 401 invalid_token                    |
  |                      |-- uses refresh pass ----------------->  |
  |                      |<-- new pass: same acr=basic, amr=[pwd]  |
  |                      |-- new pass -------> tasks ✓             |
  |                      |<--- tasks list -----|                    |
  |<-- shows tasks (Alice never saw the error)                     |
  |                      |                    |                    |
  | [clicks Assign Task] |                    |                    |
  |                      |-- pass: tasks:read ->                   |
  |                      |   has tasks:write? NO ✗                 |
  |                      |<-- 403 insufficient_scope               |
  |                      |    scope="tasks:write"                   |
  |                      | [saves the request]|                    |
  |<-- redirect to grant permission -------->|                    |
  |                                                               |
  |------ sent to Login Server for consent ------------------->  |
  |<--- consent screen: "Allow tasks:write?" ----------------- - |
  | clicks Allow         |                    |                    |
  |------ confirmed --------------------------------------------> |
  |<-- new pass: scope=tasks:read tasks:write                     |
  |    acr=basic (unchanged), amr=[pwd] (unchanged) ------------ |
  |-------------------->|                    |                    |
  |                      |-- new pass -------> tasks:write ✓       |
  |                      |   acr=basic ✓      |                    |
  |                      |<-- task assigned ---|                    |
  |<-- "Task assigned!" -|                    |                    |
```

---

## Chapter 12: The Decision Map — How the App Handles Each Rejection

```
API sends a rejection
        |
   What is the HTTP status code?
        |                |
       401              403
        |                |
   Read the error    Read the error
        |                |
  "invalid_token"   "insufficient_scope"   "access_denied" (no scope field)
        |                |                        |
  Pass is broken    Permission missing       True access denied
        |                |                   (business rule,
  Try refresh        Read which scope            not a pass issue)
  pass silently      is missing from            Show "Access Denied"
        |            the error message           message to user
  Refresh ok?              |
  |          |        Send Alice to
 YES         NO       Login Server
  |          |        for consent screen
 New pass   Force     (prompt=consent)
 works      full           |
            login    Alice clicks Allow?
                     |              |
                    YES             NO
                     |              |
                New pass         Show message:
                with scope       "This feature needs
                Replay           your permission.
                original         Other features
                request          still work."
```

---

## Chapter 13: The Important Difference — Scope vs. Identity Proof

By now you have seen all three rejection types. Here is the complete picture:

### When the pass itself is broken (401 — `invalid_token`)
The pass is expired, revoked, or fake. Get a new one using the refresh pass. If that fails too, ask Alice to log in again. **ACR and AMR are not the issue.**

### When the permission was never granted (403 — `insufficient_scope`)
The pass is real and the login was fine. But Alice never gave permission for this action. Send her to the Login Server to grant consent for the missing scope. **The ACR and AMR will NOT change** — she is not re-proving her identity, only adding a permission. The new pass will have the same `"pwd"` in AMR and the same `basic` in ACR.

### When the login was not strong enough (401 — `insufficient_user_authentication`)
The pass is real and the permission is there. But the room requires a stronger login — for example, a physical hardware key in addition to the password. Send Alice to the Login Server to re-authenticate at a higher level. **The ACR and AMR WILL change** — for example, from `["pwd"]` to `["pwd", "otp", "hwk"]`. The permissions (scope) will not change.

```
What changed after the challenge?

  Scope challenge (403):
    Before: scope=tasks:read,   acr=basic, amr=["pwd"]
    After:  scope=tasks:write,  acr=basic, amr=["pwd"]
            ↑ permission added   ↑ unchanged  ↑ unchanged

  Step-up challenge (RFC 9470):
    Before: scope=secrets:read, acr=basic,        amr=["pwd"]
    After:  scope=secrets:read, acr=mfa-hardware, amr=["pwd","otp","hwk"]
            ↑ unchanged          ↑ upgraded         ↑ more methods added
```

---

## Epilogue: What the ACR and AMR Values Tell the Whole System

Throughout this story, the ACR and AMR values were always present — even when they were not the reason something was blocked. They tell a story of their own:

**At login:** Alice uses just a password → `amr: ["pwd"]`, `acr: basic`

**After token refresh:** Same ACR and AMR — refreshing does not mean re-authenticating

**After scope upgrade:** Same ACR and AMR — granting permissions does not mean re-authenticating

**In the audit log:** Every action is logged with the ACR and AMR, so the company always knows not just *who* did something, but *how strongly* they proved they were who they said they were when they did it

This matters because today the rules say basic login is fine for assigning tasks. But tomorrow, the company might decide that write operations should require MFA. Having the AMR in every audit log means they can look back and see every assignment that was done with only password-level authentication — and decide what to do about it.

The ACR and AMR are not just for step-up challenges. They are a permanent record of identity assurance that follows every action, every pass, and every audit entry — whether anyone checks them or not.

---

*End of Story*

---

**Standards referenced in this story:**
- **RFC 6749** — How permissions (scopes) and passes (tokens) work
- **RFC 6750** — How passes are carried in requests and how rejections are communicated
- **RFC 9470** — The step-up authentication challenge (used when ACR/AMR need upgrading)
- **draft-ietf-oauth-amr-values-02** — The standard codes for authentication methods (`pwd`, `otp`, `hwk`, etc.)
- **RFC 7636** — Security protection for the code exchange step (PKCE)
- **OpenID Connect Core 1.0** — How `acr` and `amr` claims are defined and carried in tokens
