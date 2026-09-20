# Spring Security — Final Master Revision Notes

> **One file. Four lectures. Beginner → Advanced.**
> Synthesized from Lecture 1 (Security Foundations & Authentication), Lecture 2 (Database Authentication & Password Security), Lecture 3 (Database Authentication Internals & Complete Login Flow), and Lecture 4 (JWT Authentication: Stateful → Stateless).
> Nothing here is invented — every concept is supported by those notes. Repetition across lectures has been merged into a single authoritative explanation, and additional emphasis has been added on the interview-critical areas: filters, `SecurityContext`, `AuthenticationManager`, `UserDetailsService`, sessions, JWT, and security architecture.

---

## How to Use This Document

| If you have… | Read |
|---|---|
| 10 minutes | [Part 12 — Final Cheat Sheet](#part-12--final-cheat-sheet) + [Part 13 — Complete Concept Map](#part-13--complete-concept-map) |
| 1 hour | The "Core Principle" callout at the end of each part, then the cheat sheet |
| A full study session | Everything, in order. Each part ends with a one-minute revision |
| An interview tomorrow | [Part 9 — Master Comparison Tables](#part-9--master-comparison-tables), [Part 10 — Interview Q&A](#part-10--interview-qa), [Part 11 — Common Mistakes](#part-11--common-mistakes) |

**Contents**

1. [Part 1 — Security Foundations: Assets, Threats & Trust](#part-1--security-foundations-assets-threats--trust)
2. [Part 2 — Authentication vs Authorization](#part-2--authentication-vs-authorization)
3. [Part 3 — The Spring Security Identity Model](#part-3--the-spring-security-identity-model)
4. [Part 4 — Stateful vs Stateless Authentication](#part-4--stateful-vs-stateless-authentication)
5. [Part 5 — Spring Boot Security Defaults](#part-5--spring-boot-security-defaults)
6. [Part 6 — Database Authentication & Password Security](#part-6--database-authentication--password-security)
7. [Part 7 — The Authentication Engine & Complete Login Flow](#part-7--the-authentication-engine--complete-login-flow)
8. [Part 8 — JWT Authentication & Stateless Design](#part-8--jwt-authentication--stateless-design)
9. [Part 9 — Master Comparison Tables](#part-9--master-comparison-tables)
10. [Part 10 — Interview Q&A](#part-10--interview-qa)
11. [Part 11 — Common Mistakes](#part-11--common-mistakes)
12. [Part 12 — Final Cheat Sheet](#part-12--final-cheat-sheet)
13. [Part 13 — Complete Concept Map](#part-13--complete-concept-map)

---

# Part 1 — Security Foundations: Assets, Threats & Trust

## 1.1 The Big Picture

A normal Spring MVC request looks like this:

```text
HTTP Request → Controller → Service → Repository → Database
```

With security inserted, it becomes:

```text
HTTP Request
    ↓
Spring Security Filter Chain
    ↓
Authentication
    ↓
SecurityContext
    ↓
Authorization
    ↓
Controller → Service → Repository → Database
```

Everything in Spring Security exists to answer these two questions **before** your business logic runs:

```text
1. WHO is making this request?      → Authentication
2. WHAT is this identity allowed to do? → Authorization
```

> **Spring Security ultimately exists to make safe decisions around requests: establish who the requester is, preserve that identity in the security context, determine what the identity is allowed to do, and reject requests that do not satisfy the application's security requirements.**

## 1.2 What Application Security Actually Protects

Consider `DELETE /api/users/57`. A naive application thinks `Request → Controller → Service → Repository → DB`. Security adds:

```text
Who sent the request?
Has the identity been verified?
Is this identity allowed to perform DELETE?
Is it allowed to delete user 57 specifically?
Has the request been manipulated?
Could it be forged or replayed?
```

> **Application security protects valuable resources and operations from unauthorized access, modification, destruction, disclosure, and misuse.**

Security is therefore much broader than a login page.

### Asset Categories

| Category | Examples |
|---|---|
| **Data assets** | Passwords, emails, phone numbers, health records, payment info, source code, API keys, business documents |
| **Operational assets** (valuable *actions*) | Create order, transfer money, approve refund, change another user's password, delete account, publish content |
| **Infrastructure assets** | Database, server, cloud account, message broker, file storage, deployment pipeline |
| **Security assets** | Session IDs, JWT signing keys, encryption keys, OAuth client secrets, password-reset tokens, CSRF tokens |
| **Availability** | Anything that can be exhausted — request floods, repeated login attempts, database connection exhaustion |

> **Availability itself is a security asset.** An attacker does not have to steal data to cause harm.

## 1.3 Security Is a Collection of Controls

Security involves authentication, authorization, input validation, password protection, session protection, CSRF protection, CORS policies, secure communication, rate limiting, audit logging, error handling, and secret management.

Spring Security primarily helps with:

```text
Authentication · Authorization · Security-context management
Session security · Common web-security protections · Protocol integration
```

Spring Security does **NOT** automatically fix:

```text
SQL injection from unsafe queries      Insecure file uploads
Weak business rules                    Secrets committed to Git
Vulnerable dependencies                Incorrect database permissions
Sensitive information written into logs
```

> **Spring Security is one part of application security, not the entire security strategy.**

## 1.4 Asset, Threat, Vulnerability, Attack

| Term | Meaning |
|---|---|
| **Asset** | Something valuable that needs protection |
| **Threat** | Something that could harm an asset |
| **Vulnerability** | A weakness that can make an attack possible |
| **Attack** | An attempt to exploit a weakness |

```text
Threat + Vulnerability → Possible Attack → Impact
```

Worked example:

```text
Threat:        Attacker wants admin access
Vulnerability: Unlimited password attempts
Attack:        Automated password guessing
Impact:        Admin account compromised
```

## 1.5 Trust Boundaries

A **trust boundary** is a point where data moves from one trust level to another.

```text
Browser → Internet → Spring Boot Application → Database
```

Other examples: Server → Database, Service A → Service B, Mobile App → API Gateway, Public Network → Internal Network, Identity Provider → Application.

At every boundary, ask: Who produced this data? Can it be modified? Should it be authenticated / authorized / validated / encrypted?

### The Browser Is Untrusted

```javascript
if (user.role === "ADMIN") { showDeleteButton(); }   // UI only — NOT security
```

A user controls the browser and can change JavaScript, modify HTML, use Postman or curl, change parameters, call hidden endpoints, and replay requests. Even with the button hidden, they can send `DELETE /api/users/57` directly.

> **Golden Rule: Frontend restrictions improve UX; server-side authorization provides the actual security boundary.**

---

# Part 2 — Authentication vs Authorization

## 2.1 Authentication

> **Authentication is the process of verifying an identity claim.**

```text
Username → Identity claim
Password → Evidence
Verification → Compare with stored representation
Result → Success / Failure
```

A password must be represented/stored securely rather than as plain text.

### Authentication Factors

| Factor | Examples |
|---|---|
| Something you **know** | Password, PIN, security answer |
| Something you **have** | Smart card, authenticator app, private cryptographic key |
| Something you **are** | Fingerprint, face, iris |

**MFA** combines evidence from *different* categories (e.g. password + authenticator OTP). Two credentials from the same category are not necessarily strong multi-factor authentication.

## 2.2 Authorization

```text
Authentication → WHO are you?
Authorization  → WHAT are you allowed to do?
```

Authorization is more than a role check. A real decision involves four elements:

| Element | Examples |
|---|---|
| **Subject** | User 51, Administrator, Manager, service account, anonymous visitor |
| **Action** | Read, Create, Update, Delete, Approve, Transfer, Download |
| **Resource** | Order 101, employee record 57, bank account 9001, admin dashboard, course video |
| **Context** | Is the user the owner? Within business hours? Is the account active? Within approval limit? Has MFA completed? |

```text
Can Subject S perform Action A on Resource R under Context C?
```

Even a public endpoint is an authorization decision — `Permit all requests` is still a policy.

> **Interview answer: Authentication establishes identity. Authorization decides access for that identity.**

## 2.3 Normal Security Flow

```text
Request → Extract credentials → Authenticate identity → Determine authorities
        → Evaluate authorization rule → Allow / Deny
```

Authentication information must generally be established **before** identity-based authorization can be evaluated.

## 2.4 Authentication Failure vs Security Attack

They are not the same thing.

**Authentication failure** — wrong password, unknown username, expired token, invalid token signature, malformed `Authorization` header, disabled account, expired account, expired session, incorrect OTP. This means acceptable identity evidence was not established.

**Security attack** — a deliberate attempt to defeat or misuse security controls. One failed login is a failure; thousands of deliberate attempts (`password1`, `password2`, `password3`…) is brute force.

### Credential Stuffing

Uses username/password pairs stolen from *another* service.

```text
Credentials may be correct → Authentication may succeed
                          → Request can still be malicious
```

Controls that help: MFA, anomaly detection, login notifications, device verification, rate limiting, credential-breach detection.

> **Successful authentication proves acceptable credentials were presented; it does not prove the overall request is trustworthy in every sense.**

### Do Not Reveal Unnecessary Authentication Details

External response: `Invalid username or password.`
Never: `Username exists, but password is wrong.`

Internally, diagnostics may be precise (user not found, password mismatch, account disabled, token expired) — keep public-facing messages generic to prevent account enumeration.

---

# Part 3 — The Spring Security Identity Model

## 3.1 Vocabulary

These are connected but **not interchangeable**:

```text
Identity · Credentials · Principal · Authorities · Roles · Permissions
Authentication · SecurityContext · SecurityContextHolder
```

## 3.2 Identity

The entity whose actions the system wants to recognize — human user, administrator, customer, another application, microservice, scheduled job, IoT device.

**Identity vs Account:** an account is a stored application representation.

```text
users table
id | username | password_hash | enabled
42 | aditya   | ...           | true
```

> **Identity is the conceptual subject; an account is one representation of it.**

## 3.3 Credentials

Evidence used to prove an identity claim: password, OTP, private key, certificate, API key, bearer token.

```text
Identity   → "Who I claim to be"
Credential → "Evidence that I am that identity"
```

## 3.4 Principal

The identity currently represented **inside the security system**.

```text
Before:  principal = "aditya"
After:   principal = UserDetails("aditya")
```

```text
Identity  → conceptual entity
Principal → representation of that identity inside the security system
```

## 3.5 Authorities

A capability or security attribute granted to a principal:

```text
COURSE_READ · COURSE_WRITE · USER_DELETE · PAYMENT_REFUND · ROLE_ADMIN · SCOPE_orders.read
```

Spring Security represents these with `GrantedAuthority` and stores them in the current `Authentication`.

> **Golden Rule: The client can claim an identity or role, but the server must independently establish trust.** A JSON body containing `"role": "ADMIN"` cannot make anyone an admin — authority must come from a trusted database, verified token, trusted identity provider, or `AuthenticationProvider`.

## 3.6 Role vs Authority

```text
Role      → broader grouping of privileges
Authority → specific capability/security attribute
```

```text
ROLE_INSTRUCTOR
   ├── COURSE_READ
   ├── COURSE_CREATE
   ├── COURSE_UPDATE
   └── STUDENT_PROGRESS_READ
```

A role is commonly represented as an authority using the `ROLE_` prefix.

## 3.7 The `Authentication` Object

`Authentication` is one of the central Spring Security abstractions and can represent **two different things**:

**A. Authentication request (before verification)**

```text
principal     = "aditya"
credentials   = "raw-password"
authorities   = []
authenticated = false
```

Meaning: *"Someone claims to be Aditya and supplied credentials."* Not yet accepted.

**B. Authenticated principal (after successful verification)**

```text
principal     = UserDetails("aditya")
credentials   = null
authorities   = [ROLE_USER, COURSE_READ]
authenticated = true
```

### Key Methods

```java
public interface Authentication {
    Object getPrincipal();
    Object getCredentials();
    Collection<? extends GrantedAuthority> getAuthorities();
    Object getDetails();
    boolean isAuthenticated();
    String getName();
}
```

| Method | Meaning |
|---|---|
| `getPrincipal()` | Current identity |
| `getCredentials()` | Evidence used for authentication |
| `getAuthorities()` | Granted capabilities |
| `getDetails()` | Additional authentication/request details |
| `isAuthenticated()` | Whether authentication has been accepted |
| `getName()` | Name associated with the authentication |

### Authentication Transformation

```text
Unauthenticated Authentication
            ↓
     AuthenticationManager
            ↓
      Verify credentials
            ↓
Authenticated Authentication

Before: principal = username,  credentials = evidence,        authenticated = false
After:  principal = trusted,   credentials = commonly cleared, authorities = granted
```

## 3.8 `SecurityContext`

A container for the current `Authentication`.

```java
SecurityContext context = ...;
Authentication authentication = context.getAuthentication();
```

```text
SecurityContext → contains Authentication
```

**Why it exists:** without shared context, you would have to pass the current user through every layer (`service.createCourse(request, currentUser)`). Instead, Spring Security provides request-associated security state. The context is loaded at request start, updated after authentication, stored for future requests when applicable, and cleared after request completion.

## 3.9 `SecurityContextHolder`

The central access point to the current context.

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication authentication = context.getAuthentication();

String username = authentication.getName();
Object principal = authentication.getPrincipal();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();
```

```text
SecurityContextHolder → SecurityContext → Authentication
```

This lets the controller, service, method-security checks, and audit logic discover the current user without threading it manually through every signature.

## 3.10 `ThreadLocal` Strategy & Why Clearing Matters

In the servlet model, a request is commonly processed by one thread at a time, so Spring Security uses a `ThreadLocal`-based strategy by default.

```text
Request 1 → Thread A → SecurityContext(Aditya)
Request 2 → Thread B → SecurityContext(Rohit)
```

But **application servers reuse threads**. If the old context stayed in the `ThreadLocal`, a later request on the same thread could accidentally inherit the previous authentication.

> **Security context must be established and cleared around request processing.**

## 3.11 Complete Identity Model (most important diagram in Part 3)

```text
Identity
   │ claimed using
   ↓
Credentials
   │ verified by authentication system
   ↓
Authentication
   ├── Principal
   ├── Credentials
   ├── Authorities
   └── Authentication Status
            │ stored inside
            ↓
      SecurityContext
            │ accessed through
            ↓
      SecurityContextHolder
```

## 3.12 Final Mental Model — Ask These In Order

```text
1. Where did this request come from?
2. What crosses the trust boundary?
3. Who is the requester?
4. What credential proves that identity?
5. Has authentication succeeded?
6. What principal and authorities were granted?
7. Where is the Authentication stored?
8. Is this identity authorized for this action/resource?
9. Are additional protections such as CSRF relevant?
10. Only then → application business logic
```

> **Part 3 Core Principle:** Spring Security represents authentication through an `Authentication` object containing principal, credentials, authorities, details, and authentication state. The current `Authentication` is held in a `SecurityContext`, accessed through `SecurityContextHolder`. In the default servlet model this context is associated with the request-processing thread via a `ThreadLocal`-based strategy and must be cleared when request processing finishes.

---

# Part 4 — Stateful vs Stateless Authentication

## 4.1 HTTP Requests Are Independent

```text
POST /login → GET /profile → GET /orders → POST /logout
```

HTTP does not automatically remember a previous authentication.

> **Authenticating one HTTP request does not automatically authenticate future requests.**

The application therefore needs a strategy to carry authentication across requests:

```text
Stateful Authentication   VS   Stateless Authentication
server remembers                request carries enough material to reconstruct
```

## 4.2 Stateful Authentication (Session-Based)

After successful login, the server associates the authenticated `SecurityContext` with an HTTP session.

```text
CLIENT                                    SERVER
username + password  ────────────────→   Authentication succeeds
                                               ↓
                                          HttpSession
                                               ↓
                                          SecurityContext
                                               ↓
                                          Authentication
       ←────────────────────────────────  Set-Cookie: JSESSIONID=ABC123
```

Later request:

```http
GET /profile
Cookie: JSESSIONID=ABC123
```

```text
ABC123 → Find HttpSession → SecurityContext → Authentication → user is authenticated
```

The original password never needs to be resent.

### What Is Actually Stored

The session ID is normally only an identifier. The browser generally does **not** contain username, password, or role inside `JSESSIONID`; the important authentication state lives on the server.

```text
Browser: JSESSIONID = ABC123
Server:  ABC123 → Session → SecurityContext → Authentication → Aditya, ROLE_USER
```

> **Session-based authentication is stateful because the server maintains authentication state between requests.**

### Where Session State Can Live

```text
Application memory · Distributed cache · Redis · Database · External session store
```

### Advantages

```text
Authenticate once · Server-side invalidation · Server controls session lifetime
Can track session activity · Can manage concurrent sessions
```

If the server invalidates session `ABC123`, future requests using it can fail immediately.

### Costs

```text
Server storage   → every active session consumes resources
Distributed complexity → multiple servers need shared session storage,
                         sticky sessions, or another consistent strategy
```

### Cookie ≠ Stateful Authentication

Cookies can carry a session ID, language preference, theme, analytics identifier, CSRF token, or an authentication token. A stateless application can put an access token in a cookie.

> **Cookie does not automatically mean stateful authentication.**

### Session ID Is a Bearer Credential

If an attacker obtains `JSESSIONID=ABC123`, they may not know the password but can still access the authenticated session. Protect it like a credential.

**Session security protections:**

```text
HTTPS · Secure cookie attribute · HttpOnly · SameSite policy
Session expiration · Session invalidation · Session ID rotation · Session fixation protection
```

**Session fixation:**

```text
Attacker knows session ID = XYZ → Victim logs in using XYZ
                                → XYZ becomes authenticated
                                → Attacker reuses XYZ

Mitigation: Before authentication → XYZ
            After authentication  → NEW-789   (rotate the identifier)
```

## 4.3 Stateless Authentication

The server does not rely on a previously stored HTTP login session to reconstruct authentication. Each request carries a credential.

```http
Authorization: Bearer TOKEN
```

```text
Request → Extract credential → Validate credential → Create Authentication
        → Place in SecurityContext → Authorize → Controller → Clear request context
```

Next request repeats authentication using **that request's** credential.

> **Stateless authentication means the server does not depend on a previously stored HTTP login session to reconstruct authentication for the next request.**

### Stateless ≠ "The Server Stores Nothing"

A stateless application can still store users, passwords, roles, permissions, signing keys, refresh tokens, revoked-token records, audit logs, rate-limit counters, and business data.

## 4.4 Self-Contained vs Opaque Tokens

**Self-contained** — carries signed information that can often be validated locally. JWT is the common example:

```json
{ "sub": "aditya", "scope": "course.read course.write",
  "iat": 1785980000, "exp": 1785983600 }
```

**Opaque** — looks like `a8f2b9137c45d...`; reveals no meaningful application information and may require a token service: *Is this token active? Which user does it represent? What scopes does it contain?*

## 4.5 JWT as a Format, Bearer as a Presentation Style

> **JWT is a token format, not a synonym for stateless authentication.**

A **bearer token** means *whoever possesses the token can present it* — like cash. The server generally does not require the holder to prove they were the original recipient.

```text
Bearer  → how possession of the credential is used
Opaque  → how the token is represented
```

These are different dimensions — a token can be bearer **and** opaque:

```http
Authorization: Bearer a8f2b9137c45d...
```

**Protect bearer tokens from:** logging, URL leakage, browser storage attacks, network interception, accidental sharing, source-code commits. A JWT is not automatically safe simply because it is a JWT — safety also depends on strong signing keys, correct signature validation, expiry/issuer/audience validation, secure transport, safe client storage, and appropriate scopes.

## 4.6 JWT Can Be Used With Sessions

```text
Stateless JWT design:      JWT sent every request → validated every request → no HTTP-session auth
Session-backed JWT design: JWT used during authentication → Authentication stored in HTTP session
                           → later requests use session cookie
```

> **Token format does not determine session policy.**

## 4.7 Authentication Mechanism vs Session Policy

| Concept | Question it answers | Examples |
|---|---|---|
| **Authentication mechanism** | How are credentials obtained and verified? | Form login, HTTP Basic, Bearer token |
| **Session policy** | After authentication, is the resulting `SecurityContext` persisted in an HTTP session? | `IF_REQUIRED`, `STATELESS` |

```text
Authentication mechanism ≠ Session policy
```

### Form Login

```http
POST /login
username=aditya&password=secret
```

```text
Authenticate → Create SecurityContext → Persist in HTTP session → Send session cookie
```

Traditional stateful design.

### HTTP Basic

```http
Authorization: Basic <credentials>
```

Credentials may be resent on every request.

> **HTTP Basic is a credential transport mechanism; `STATELESS` is a session persistence policy. They are not the same concept.**

## 4.8 Request Lifecycles

**Stateful**

```text
POST /login → extract username/password → authenticate → create Authentication
            → create SecurityContext → save in HttpSession → return session cookie

GET /profile → Cookie: JSESSIONID=ABC123 → load session → load SecurityContext
             → place Authentication in SecurityContextHolder → authorize → controller
             → clear SecurityContextHolder
```

**Stateless**

```text
GET /profile → Authorization: Bearer TOKEN → extract token → authenticate token
             → create Authentication → create SecurityContext
             → place Authentication in SecurityContextHolder → authorize → controller
             → clear SecurityContextHolder → (no authentication saved in HTTP session)
```

> **Part 4 Core Principle:** Stateful authentication persists server-side HTTP session state between requests; stateless authentication carries the credential on each request and rebuilds authentication from it. JWT is a token format, not automatically a stateless design, and cookies do not automatically imply sessions.

---

# Part 5 — Spring Boot Security Defaults

## 5.1 Auto-Configuration

```text
Security dependency added → Security classes found on classpath
→ Security auto-configuration activates → Security beans created
→ Security Filter Chain registered → Requests pass through security processing
```

Spring Security provides the framework; Spring Boot observes what is available and supplies sensible defaults.

## 5.2 The Two Default Problems

| Problem | Solved by |
|---|---|
| Which requests are protected? Which security mechanisms are available? | `SecurityFilterChain` |
| Which user can authenticate? Where is that user loaded from? | `UserDetailsService` |

```text
SecurityFilterChain → How requests are secured
UserDetailsService  → How user information is loaded for authentication
```

## 5.3 Default Security Configuration

```java
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(authorize ->
            authorize.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults())
        .httpBasic(Customizer.withDefaults());
    return http.build();
}
```

Meaning: every request requires authentication + form login available + HTTP Basic available.

**`anyRequest().authenticated()`** asks only *"Is the requester authenticated?"* — not *"Does the requester have ROLE_ADMIN?"* That belongs to authority checks.

## 5.4 Conditional Auto-Configuration

```text
SecurityFilterChain bean exists?
   ┌────┴────┐
  NO        YES
   ↓          ↓
Default   Back off and use custom config
created
```

### `SecurityFilterChain` vs `UserDetailsService`

- `SecurityFilterChain` controls request-security rules and filter processing.
- `UserDetailsService` loads user information used by the authentication system.
- Other components can also participate: `AuthenticationProvider`, `AuthenticationManager`.

> **Nuance:** creating a custom `SecurityFilterChain` changes the default web-security configuration. It does **not** automatically mean the generated default user disappears. To replace the generated user/authentication setup, provide an appropriate user/authentication component.

## 5.5 The Default User

```text
InMemoryUserDetailsManager
      ↓
UserDetails
      ├── username = "user"
      ├── password = generated password
      ├── authorities = ROLE_USER
      └── enabled = true
```

It is **not** saved in MySQL, not created by JPA, not created by your controller, not part of a registration system, and not intended as a permanent production account. It exists so the default development configuration has an identity that can authenticate.

**Configuring it for learning/testing:**

```properties
spring.security.user.name=aditya
spring.security.user.password=secret123
spring.security.user.roles=USER,ADMIN
```

> **The generated password is intended for development/testing, not as a production authentication strategy.**

## 5.6 The CSRF `403 Forbidden` Surprise

```text
Login succeeds → POST /api/courses → 403 Forbidden
```

Your first thought may be *"the user does not have permission."* Another reason may be **CSRF protection**, which Spring Security enables by default for servlet web applications. A state-changing request (`POST`, `PUT`, `PATCH`, `DELETE`) may be rejected when the expected CSRF token is missing.

## 5.7 `401` vs `403`

| Status | Meaning |
|---|---|
| **401 Unauthorized** | Authentication missing or failed |
| **403 Forbidden** | Request was understood but rejected — may be missing CSRF token, not only an insufficient role |

```text
401 → "Who are you?" problem
403 → "Request understood, but access/rejection" problem
```

## 5.8 End-to-End Security Flow

```text
CLIENT → HTTP REQUEST → TRUST BOUNDARY → SPRING SECURITY FILTER CHAIN
→ AUTHENTICATION → Authentication object → SecurityContext → SecurityContextHolder
→ AUTHORIZATION → CONTROLLER → SERVICE → REPOSITORY → DATABASE
```

> **Part 5 Core Principle:** Spring Boot supplies default security through auto-configuration, including a default `SecurityFilterChain` and a development-time in-memory user. A custom `SecurityFilterChain` controls web-security rules, while `UserDetailsService` loads user information for authentication. A successful login does not guarantee every state-changing request succeeds, because CSRF protection can still reject a request with `403 Forbidden`.

---

# Part 6 — Database Authentication & Password Security

## 6.1 From Default User to Database

The default setup is useful for learning, testing, and prototypes — but a real application needs many users, persistent accounts, passwords, roles, account status, and data that survives restarts, deployments, and scaling.

```text
Default User → Database Authentication
```

## 6.2 Authentication Needs a Source of Truth

```text
USER-SUBMITTED DATA          TRUSTED DATA
username = aditya            username = aditya
password = secret123         passwordHash = ...
                             roles = ...
                             accountStatus = ...

→ Claimed identity + Credential → Validation → Trusted user information → Result
```

The source of truth can live in application memory, a relational database, MongoDB, LDAP, an external identity provider, or a separate authentication service. These notes use **MySQL**.

## 6.3 Why Spring Security Doesn't Understand Your Database

Every application differs:

```text
users(id, email, password, phone, created_at, enabled)
employees(employee_id, corporate_email, credential_hash, department, active)
customers / members / accounts ...
```

Spring Security cannot force the same table names, column names, database technology, or schema design. So it provides common abstractions — most importantly `UserDetails`.

```text
Application-specific User → Adapter → Spring Security UserDetails
```

> **Domain `User` and Spring Security `UserDetails` are related, but they are not automatically the same object or concept.**

## 6.4 Identity, Credential, and the Login Identifier

```text
Who are you?     → Identity   → username / email
Can you prove it? → Credential → password
```

Spring Security method names are historical (`loadUserByUsername(...)`, `getUsername()`), but your login identifier can be a username, email, employee ID, phone number, customer ID, or any unique account identifier. `findByEmail(email)` can be used instead of `findByUsername(username)`.

> **Important rule: the login identifier must uniquely identify exactly one account.** Duplicate emails make `login = aditya@gmail.com` ambiguous — so the identifier should normally have a unique database constraint.

## 6.5 Minimum Database Information

```text
User
 ├── username
 └── password

users
--------------------------------
id        BIGINT PRIMARY KEY
username  VARCHAR UNIQUE
password  VARCHAR
```

A realistic application may also add `enabled`, roles, account state, timestamps, and profile data.

## 6.6 `UserDetails` Interface

```java
public interface UserDetails {
    String getUsername();
    String getPassword();
    Collection<? extends GrantedAuthority> getAuthorities();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

```text
UserDetails
   ├── Identity
   ├── Credential
   ├── Authorities
   └── Account Status
```

| Method | Meaning |
|---|---|
| `getUsername()` | The login identifier (name is historical — can be an email) |
| `getPassword()` | The **stored encoded** password — never the raw password |
| `getAuthorities()` | Granted authorities (e.g. `ROLE_USER`, `COURSE_READ`) — used later by authorization |
| `isAccountNonExpired()` | Relates to the **account** itself (e.g. contract expiring) |
| `isAccountNonLocked()` | Account temporarily blocked (e.g. after repeated failed logins) |
| `isCredentialsNonExpired()` | Relates to the **credential** (e.g. 90-day password policy) |
| `isEnabled()` | Whether the account is enabled for authentication |

### Enabled vs Locked

```text
Disabled → the account exists, but it is not enabled for authentication
Locked   → the account exists, but authentication is temporarily blocked
```

### Account Expiration vs Credential Expiration

```text
Account expired    → the account itself is no longer active
Credential expired → the account can exist, but the credential is no longer valid
```

### You Don't Need Every Status in the Database

If your application doesn't support account expiration, `isAccountNonExpired()` can simply return `true`.

> **Design principle: Spring Security provides the abstraction; your application decides which account states it actually needs.**

## 6.7 Domain Layer vs Security Layer

```text
Application / Domain Layer          Security Layer
User                                UserDetails
 ├── email                           ├── username
 ├── profile                         ├── password
 ├── createdAt                       ├── authorities
 ├── password                        └── account status
 └── roles

Domain User → Adapter → UserDetails        (User == UserDetails is NOT safe to assume)
```

## 6.8 Users, Roles, and the Many-to-Many Model

```text
STUDENT → view courses
TEACHER → create courses
ADMIN   → delete users
```

Users can hold multiple roles, which naturally produces `User ↔ Role` as many-to-many.

```text
users                              roles
--------------------------------   ------------------
id | username | password | enabled  id | name
1  | aditya   | ...      | true     1  | ROLE_USER
2  | rohit    | ...      | true     2  | ROLE_ADMIN

user_roles
--------------------------------
user_id | role_id
1       | 1        → Aditya → ROLE_USER
1       | 2        → Aditya → ROLE_ADMIN
2       | 1        → Rohit  → ROLE_USER
```

### Entity Mapping

```java
@Entity @Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String password;

    private boolean enabled = true;

    @ManyToMany
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<Role> roles = new HashSet<>();
}

@Entity @Table(name = "roles")
public class Role {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;          // ROLE_USER / ROLE_ADMIN
}
```

> **Why unidirectional?** Authentication only needs `user.getRoles()`; it does not need `role.getUsers()`. **Add a bidirectional relationship only when the application actually needs navigation in both directions.**

## 6.9 Repository Layer

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    boolean existsByUsername(String username);
}
```

```text
findByUsername()     → load user during authentication
existsByUsername()   → prevent duplicate registration
```

## 6.10 The Missing Bridge: `UserDetailsService`

```text
Spring Security → ?????? → UserRepository → Database
```

`UserDetailsService` is the bridge.

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found"));
        return new CustomUserDetails(user);   // or User.withUsername(...).build()
    }
}
```

```text
Login identifier → find domain user → adapt domain user → return UserDetails
```

**Why it exists:** every application has a different user model, but Spring Security wants one common abstraction — this prevents Spring Security from being tightly coupled to your database structure.

## 6.11 Where Password Verification Happens

```text
username = aditya, password = secret123     (client)
username = aditya, password = $2a$...       (database)

NOT: secret123 == $2a$...
BUT: Raw password + Stored encoded password → PasswordEncoder.matches() → true/false
```

## 6.12 Why Raw Passwords Must Never Be Stored

```text
Database breach → raw passwords exposed
                → users may have reused the same password elsewhere
                → damage spreads to other services
```

> **Golden rule: Never intentionally store a user's raw password.**

## 6.13 Why Encryption Is the Wrong Model

```text
Plaintext → Encryption + Key → Ciphertext → Decryption + Key → Plaintext
```

If attackers obtain both the database **and** the decryption key, they may recover all passwords. But login never needs the original password — it only needs *"does the password supplied now match the password registered earlier?"*

> **Passwords should use one-way password hashing rather than reversible encryption.**

## 6.14 Password Hashing

```text
secret123 → One-way password function → Encoded representation → Database
```

There should be no normal operation that turns the encoded password back into the original.

## 6.15 Why SHA-256 Alone Is Not Enough

General-purpose hashes are intentionally fast. For password storage, fast hashing is dangerous because an attacker with the hash database can perform offline guessing:

```text
guess password → calculate hash → compare → repeat...
```

A fast algorithm allows many guesses per second.

## 6.16 Adaptive Password Hashing

```text
Fast general-purpose hash → cheap guess → attacker tests many guesses quickly
Adaptive password hash    → expensive guess → slows large-scale offline guessing
```

These notes use **BCrypt**.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

```text
PasswordEncoder      ← abstraction
      ↑
BCryptPasswordEncoder ← concrete implementation
```

Depend on `PasswordEncoder` in application code rather than hard-coding `BCryptPasswordEncoder`:

```java
private final PasswordEncoder passwordEncoder;   // concrete impl configured separately
```

## 6.17 `encode()` vs `matches()`

```text
REGISTRATION:  encode(rawPassword) → store encoded value
LOGIN:         matches(rawPassword, storedEncodedValue) → true / false
```

```java
String encoded = passwordEncoder.encode("secret123");     // → $2a$10$... (varies each run)

boolean valid = passwordEncoder.matches("secret123", storedEncodedPassword);  // true
boolean bad   = passwordEncoder.matches("wrongPassword", storedEncodedPassword); // false
```

> **Never do this:** `passwordEncoder.encode(rawPassword).equals(storedEncodedPassword)` for BCrypt-style salted storage.

## 6.18 Salt — Why the Same Password Hashes Differently

```text
Aditya + secret123 + randomSaltA → BCrypt → encodedValueA
Rohit  + secret123 + randomSaltB → BCrypt → encodedValueB
```

```text
Same password → different salts → different encoded values
```

**Why salt matters:** without unique salts, attackers can exploit precomputed password → hash dictionaries / rainbow tables. With salts, one precomputed list becomes much less useful.

**Do you need a separate salt column?** No. Standard BCrypt encoded values already contain the algorithm information, cost parameters, salt, and derived password data inside the encoded value.

### How Does `matches()` Work If Every Hash Is Different?

This is a top interview question. `matches()` does **not** re-encode and compare strings. Instead:

```text
Stored BCrypt value → read its salt + parameters → use them to verify the raw password → true/false
```

## 6.19 Registration Flow

```text
POST /register { "username": "aditya", "password": "secret123" }
      ↓
RegisterRequest → Validate input → Check duplicate username → PasswordEncoder → BCrypt
→ Encoded password → User entity → UserRepository → Database
```

### Raw Password Lifetime

The raw password should exist only for the short time needed to process registration. It must **not** be logged, returned in a response, stored in the database, or unnecessarily copied elsewhere.

```text
raw password → encode immediately → encoded password → persist
```

### Registration DTO

```java
public class RegisterRequest {
    @NotBlank private String username;
    @NotBlank @Size(min = 8) private String password;
}
```

### Validation vs Encoding

| | Question | Examples |
|---|---|---|
| **Validation** | Is the submitted password acceptable? | Not blank, min/max length, password policy |
| **Encoding** | How do we safely store the password representation? | BCrypt → encoded value |

> **Validation ≠ Password Encoding.**

### Registration Service

```java
@Service
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public User register(RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new IllegalArgumentException("Username is already registered");
        }
        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword()));   // ← the critical line
        user.setEnabled(true);
        return userRepository.save(user);
    }
}
```

Responsibilities: validate incoming data → check duplicate identity → encode password → create user entity → save user.

## 6.20 Roles Flow Into Spring Security

```text
User → user_roles → Role → ROLE_USER / ROLE_ADMIN
     → UserDetails → GrantedAuthority → Authentication → Authorization
```

The database relationship is not isolated from security — it eventually feeds authorization decisions.

## 6.21 Complete Authentication Architecture (Part 6 master diagram)

```text
                       CLIENT
                          │ username + password
                          ↓
                 AuthenticationManager
                          ↓
               DaoAuthenticationProvider
                    ┌─────┴─────┐
                    ↓           ↓
           UserDetailsService  PasswordEncoder
                    │           │
                    ↓           ↓
             UserRepository    BCrypt
                    │
                    ↓
                 Database → User Entity → UserDetails
                    │
                    └── password verification
                              ↓
                    Authentication → SecurityContext
```

> **Part 6 Core Principle:** Spring Security does not need to understand your entire database. Your application provides a bridge through `UserDetailsService`, which loads and adapts trusted database information into `UserDetails`. `DaoAuthenticationProvider` coordinates that with `PasswordEncoder` to verify credentials and produce an authenticated `Authentication`. For new users, raw passwords are validated, encoded with an adaptive one-way password-hashing algorithm such as BCrypt, and only the encoded representation is stored.

---

# Part 7 — The Authentication Engine & Complete Login Flow

> **Main goal:** understand what actually happens between an incoming HTTP request and the moment Spring Security produces an authenticated user.

## 7.1 The Most Important Rule

```text
HTTP Request
     ↓
Servlet Filters
     ↓
Spring Security Filters
     ↓
DispatcherServlet
     ↓
Controller
```

Spring Security authentication happens **before the request reaches your controller**. For `GET /admin/users` the chain must answer:

```text
Who is this? → Is authentication valid? → What authorities do they have?
→ Are they allowed to access /admin/users? → Only then → Controller
```

Conceptual stage order:

```text
Protection → Authentication → Exception Handling → Authorization → Application
```

> **Authentication must happen before authorization because Spring Security cannot decide whether a user has a particular authority until it knows who the user is.**

## 7.2 Servlet Filters

A servlet filter sits between the client and the web application and can inspect, modify, or reject requests before they reach the servlet.

```text
Request → Filter 1 → Filter 2 → Filter 3 → Servlet
```

Spring Security builds its architecture around this mechanism.

## 7.3 `DelegatingFilterProxy`

A real servlet filter known by the servlet container, but it does not itself contain all of Spring Security's logic. Its job is to **delegate** to a Spring-managed security filter bean.

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
    Filter springFilter = findFilterBeanInSpring();
    springFilter.doFilter(request, response, chain);
}
```

```text
Tomcat → DelegatingFilterProxy → Spring-managed security filter
```

**Why it is needed:** there are two worlds — the servlet container (Tomcat / servlet filter lifecycle) and the Spring `ApplicationContext` (Spring-managed beans). `DelegatingFilterProxy` is the bridge.

```text
Tomcat says: "I have a request."
DelegatingFilterProxy says: "I know which Spring bean should handle security."
Spring Security says: "I'll process the request."
```

## 7.4 `FilterChainProxy`

The central Spring Security filter standing in front of one or more `SecurityFilterChain`s.

```text
Tomcat → DelegatingFilterProxy → FilterChainProxy → SecurityFilterChain
```

**Why it exists:** an application may need different security rules for different URLs. `FilterChainProxy` chooses and executes a suitable `SecurityFilterChain`.

## 7.5 `SecurityFilterChain`

```text
Request matcher + Ordered security filters
```

```java
public interface SecurityFilterChain {
    boolean matches(HttpServletRequest request);
    List<Filter> getFilters();
}
```

The real implementation is framework-internal, but this mental model is extremely useful.

### Example Chains

```text
SecurityFilterChain 1   Matcher: /api/**
  SecurityContextHolderFilter · BearerTokenAuthenticationFilter
  ExceptionTranslationFilter · AuthorizationFilter

SecurityFilterChain 2   Matcher: /**
  SecurityContextHolderFilter · CsrfFilter · UsernamePasswordAuthenticationFilter
  BasicAuthenticationFilter · AnonymousAuthenticationFilter
  ExceptionTranslationFilter · AuthorizationFilter
```

The exact filter list depends on which security features you configure.

## 7.6 `RequestMatcher`

Decides whether a `SecurityFilterChain` applies. It can evaluate request path, HTTP method, headers, servlet path, parameters, and dispatcher type.

```text
GET /api/users   +   matcher /api/**   → true
GET /api/users   +   matcher /admin/** → false
```

## 7.7 First Matching `SecurityFilterChain` Wins

```text
Chain 1 → /api/**
Chain 2 → /**

GET /api/users → both match → SELECT Chain 1 → STOP SEARCHING
```

The request does **not** receive `Chain 1 filters + Chain 2 filters`.

> **Golden rule: First matching SecurityFilterChain wins.**

### Order Specific → General

```text
1. /api/**
2. /admin/**
3. /**
```

Because `/**` matches almost everything — if it appears first, it captures requests intended for another chain.

### If No Chain Matches

Spring Security simply does not apply a matching `SecurityFilterChain`. A fallback chain is therefore useful:

```java
@Bean
SecurityFilterChain fallback(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
    return http.build();
}
```

If you do not configure a narrower `securityMatcher`, a chain normally applies to all requests.

## 7.8 `HttpSecurity`

> **`HttpSecurity` is a builder used to describe how a `SecurityFilterChain` should behave.**

```java
http
    .csrf(Customizer.withDefaults())
    .formLogin(Customizer.withDefaults())
    .httpBasic(Customizer.withDefaults())
    .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
```

It controls: which security features are enabled, which filters are required, how those filters are configured and ordered, which `AuthenticationProvider`s are used, and which authorization rules apply. `http.build()` turns the builder into a concrete `SecurityFilterChain`.

```text
HttpSecurity         → configuration builder
SecurityFilterChain  → built security pipeline
```

## 7.9 Authentication Filters

Different mechanisms receive credentials differently.

```http
POST /login
Content-Type: application/x-www-form-urlencoded

username=aditya&password=secret123        → UsernamePasswordAuthenticationFilter
```

```http
Authorization: Basic YWRpdHlhOnNlY3JldDEyMw==   → BasicAuthenticationFilter
```

### Critical: A Filter Does Not Authenticate Every Request

The presence of `UsernamePasswordAuthenticationFilter` in the chain does **not** mean it tries username/password login for `GET /courses`. It looks for the request pattern it understands (`POST /login`) and otherwise continues:

```java
chain.doFilter(request, response);
```

> **Golden rule: Authentication filter exists in the chain ≠ authentication attempt happens on every request.**

## 7.10 Same Authentication Backend

```text
Form Login → UsernamePasswordAuthenticationFilter
HTTP Basic → BasicAuthenticationFilter

Both → AuthenticationManager → ProviderManager → DaoAuthenticationProvider → Database
```

> **Credential transport can differ while the actual database-authentication engine remains the same.**

## 7.11 From HTTP Request to Authentication Token

```text
username = aditya, password = secret123
        ↓
UsernamePasswordAuthenticationToken
```

### Unauthenticated Token

```text
UsernamePasswordAuthenticationToken
principal     = "aditya"
credentials   = "secret123"
authorities   = []
authenticated = false
```

This is **not yet an authenticated user** — it means *"Someone claims to be Aditya and has supplied this credential."* Only after verification can a trusted result be produced.

```java
UsernamePasswordAuthenticationToken token =
    UsernamePasswordAuthenticationToken.unauthenticated(username, password);
```

### What the Filter Should Do

The filter should **not** contain database logic, password verification logic, or role lookup logic. Its responsibilities are:

```text
1. Extract credentials
2. Create Authentication request
3. Send it to the authentication engine
```

## 7.12 `AuthenticationManager`

> **The main entry point into Spring Security's authentication engine.**

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

```text
Authentication request
        ↓
AuthenticationManager
   ┌────┴────┐
Success    Failure
   ↓          ↓
Trusted Authentication   AuthenticationException
```

## 7.13 `ProviderManager`

`AuthenticationManager` is an interface; the common implementation is `ProviderManager`.

**Why?** An application can support multiple mechanisms — database username/password, JWT, OTP, LDAP, SAML, remember-me, custom authentication. Instead of one giant class handling everything, Spring Security uses `AuthenticationProvider` objects, and `ProviderManager` coordinates them.

```text
ProviderManager
    ├── DaoAuthenticationProvider
    ├── JwtAuthenticationProvider
    ├── RememberMeAuthenticationProvider
    └── CustomOtpAuthenticationProvider
```

## 7.14 `AuthenticationProvider`

Each provider understands a particular authentication mechanism/token type and exposes:

```java
boolean supports(Class<?> authenticationType);   // "Can this provider handle this Authentication type?"
```

```text
DaoAuthenticationProvider → supports UsernamePasswordAuthenticationToken
JwtAuthenticationProvider → supports JWT-related authentication
```

### ProviderManager Decision

```text
UsernamePasswordAuthenticationToken → ProviderManager asks "which provider supports this?" → DaoAuthenticationProvider → authenticate
```

## 7.15 The Bridge to the Database

```text
AuthenticationManager → ProviderManager → DaoAuthenticationProvider
    ├── UserDetailsService → UserRepository → Database
    └── PasswordEncoder → matches(...)
```

### The Two Most Important Contracts

```text
UserDetails        → Spring Security's view of ONE user   (Result)
UserDetailsService → HOW to FIND that user                (Finder)
```

### Application User vs Spring Security User

`com.example.entity.User` and `org.springframework.security.core.userdetails.User` are **not the same class**. Yours is the domain/database entity; Spring Security's is a built-in `UserDetails` implementation.

### `CustomUserDetails`

```java
public class CustomUserDetails implements UserDetails {
    private final User user;

    public CustomUserDetails(User user) { this.user = user; }

    @Override public String getUsername() { return user.getUsername(); }
    @Override public String getPassword() { return user.getPassword(); }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority(role.getName()))
            .toList();
    }

    @Override public boolean isAccountNonExpired()     { return true; }
    @Override public boolean isAccountNonLocked()      { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }
    @Override public boolean isEnabled()               { return user.isEnabled(); }
}
```

```text
Role entity → role.getName() → "ROLE_ADMIN" → SimpleGrantedAuthority → GrantedAuthority
```

### Is `CustomUserDetails` Compulsory?

No. You can use Spring Security's built-in `User`:

```java
return User.withUsername(user.getUsername())
    .password(user.getPassword())
    .authorities(user.getRoles().stream()
        .map(role -> new SimpleGrantedAuthority(role.getName()))
        .toList())
    .disabled(!user.isEnabled())
    .build();
```

Custom `UserDetails` is useful when you want explicit mapping, access to additional domain properties, and a clear separation between domain and security layers.

### Email as the Login Identifier

```java
@Override
public UserDetails loadUserByUsername(String email) {
    User user = userRepository.findByEmail(email)
        .orElseThrow(() -> new UsernameNotFoundException("User not found"));
    return new CustomUserDetails(user);
}
```

The interface name is historical; the actual identifier is an application-design choice.

## 7.16 `DaoAuthenticationProvider` — The Workhorse

```text
Authentication request
        ↓
DaoAuthenticationProvider
     ┌────┴─────┐
     ↓          ↓
UserDetailsService  PasswordEncoder
     ↓          ↓
Repository       BCrypt
     ↓
Database
```

What it actually does:

```text
1. Receive authentication request
2. Extract username
3. Ask UserDetailsService for user
4. Obtain encoded password
5. Check account status
6. Compare submitted password
7. Create trusted Authentication
8. Return the authenticated result
```

**Why "DAO"?** DAO = Data Access Object. The name reflects that user information is obtained from a user-data source before credentials are verified (JPA repository, JDBC, Mongo repository, custom data access). The key idea: *provider retrieves user data, then verifies credentials.*

## 7.17 Complete Internal Login Flow (master diagram)

```text
HTTP Request
       ↓
Authentication Filter
       ↓
Unauthenticated UsernamePasswordAuthenticationToken   (authenticated = false)
       ↓
AuthenticationManager
       ↓
ProviderManager
       ↓
DaoAuthenticationProvider
       ├── UserDetailsService → UserRepository → Database
       └── PasswordEncoder → Password verification
       ↓
Authenticated UsernamePasswordAuthenticationToken     (authenticated = true)
       ↓
SecurityContext
       ↓
SecurityContextHolder
       ↓
Authorization
       ↓
Controller
```

## 7.18 The 12 Steps With Real Values

| Step | What happens |
|---|---|
| 1 | Filter extracts credentials from `POST /login` (form) or `Authorization` header (Basic) |
| 2 | Creates unauthenticated `UsernamePasswordAuthenticationToken` (`aditya` / `secret123` / `authenticated = false`) |
| 3 | Calls `authenticationManager.authenticate(token)` |
| 4 | `ProviderManager` selects `DaoAuthenticationProvider` for `UsernamePasswordAuthenticationToken` |
| 5 | Provider extracts `principal = "aditya"` |
| 6 | `userDetailsService.loadUserByUsername("aditya")` → repository → database → `UserDetails` |
| 7 | Account status check (`isEnabled()`, locked, expired, credentials expired) |
| 8 | `passwordEncoder.matches("secret123", "$2a$10$...")` → `true` |
| 9 | Provider returns trusted `Authentication` (principal = `CustomUserDetails`, credentials cleared, authorities = `[ROLE_USER]`, `authenticated = true`) |
| 10 | `Authentication` goes into `SecurityContext` → `SecurityContextHolder` |
| 11 | Authorization rules evaluated (`hasRole("ADMIN")`, `authenticated()`) |
| 12 | Controller executes — it does not need to query the user, verify the password, or rebuild `Authentication` |

### Two Points Worth Extra Attention

**Account status is part of authentication.** Even with a correct password, if `enabled = false` (or locked/expired), authentication fails.

> **Database authentication is not only password comparison. Account state is part of authentication.**

**Do not imagine `encode(rawPassword) == storedEncodedPassword`.** With BCrypt a random salt is involved; instead the stored value's salt + parameters are used to verify the supplied raw password.

**Why are credentials erased?** After success, the raw password should not linger longer than necessary — credentials become `null` or are otherwise erased.

> **Do not retain sensitive authentication material longer than necessary.**

## 7.19 Failure Paths

```text
User not found    → UsernameNotFoundException
Wrong password    → matches(...) returns false → BadCredentialsException
Invalid account   → disabled / locked / expired / expired credentials
```

In failure cases: no trusted authenticated result, and the `SecurityContext` is not populated with a successful authentication.

## 7.20 Registration vs Login (Critical Difference)

| | Flow |
|---|---|
| **Registration** (user does not exist yet) | Client → username + raw password → AuthService → `PasswordEncoder.encode(...)` → BCrypt encoded password → `UserRepository` → Database |
| **Login** (user already exists) | Client → username + raw password → Authentication Filter → `AuthenticationManager` → `DaoAuthenticationProvider` → load existing user → `PasswordEncoder.matches(...)` → authenticated `Authentication` |

> **Golden rule: Registration → `encode()`. Login → `matches()`.**

### Registration Must Not Accept a Client-Chosen Admin Role

Unsafe request: `{ "username": "hacker", "password": "123", "role": "ROLE_ADMIN" }`. If the backend trusts this, anyone can register as ADMIN — privilege escalation. The server should assign a safe default:

```java
Role role = roleRepository.findByName("ROLE_USER").orElseThrow();
user.getRoles().add(role);
```

> **The client can request an account; the server decides the privileges.**

### Registration Response Should Not Return the Entity Blindly

An entity containing username/password/roles can accidentally expose sensitive data. Use a response DTO:

```java
public record UserResponse(Long id, String username) {}
```

## 7.21 Suggested Project Structure

```text
src/main/java
├── entity/     User.java, Role.java
├── repository/ UserRepository.java, RoleRepository.java
├── dto/        RegisterRequest.java, UserResponse.java
├── security/   CustomUserDetails.java, CustomUserDetailsService.java
├── service/    AuthService.java
├── controller/ AuthController.java, TestController.java
└── config/     SecurityConfig.java
```

```text
Entity     → database/domain model
Repository → database access
DTO        → API request/response structure
Security   → Spring Security adapters
Service    → business logic
Controller → HTTP layer
Config     → security configuration
```

## 7.22 Key Implementation Pieces

**Repositories**

```java
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}

public interface RoleRepository extends JpaRepository<Role, Long> {
    Optional<Role> findByName(String name);
}
```

**DTOs**

```java
public record RegisterRequest(String username, String password) {}

// In production, add validation such as @NotBlank / @Size
// according to your password policy.
```

**PasswordEncoder + explicit provider (useful for learning)**

```java
@Bean
public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }

@Bean
public DaoAuthenticationProvider authenticationProvider(
        CustomUserDetailsService userDetailsService,
        PasswordEncoder passwordEncoder) {

    DaoAuthenticationProvider provider = new DaoAuthenticationProvider(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder);
    return provider;
}
```

```text
DaoAuthenticationProvider
     ├── CustomUserDetailsService
     └── PasswordEncoder
```

Spring Security can often auto-configure much of this when the appropriate beans exist, but explicitly defining it is useful for seeing the internals.

**Security configuration**

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }

    @Bean
    public DaoAuthenticationProvider authenticationProvider(
            CustomUserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {

        DaoAuthenticationProvider provider = new DaoAuthenticationProvider(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return provider;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http,
            DaoAuthenticationProvider provider) throws Exception {

        http
            .csrf(csrf -> csrf.disable())
            .authenticationProvider(provider)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/register").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .formLogin(Customizer.withDefaults())
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }
}
```

> **Warning about `.csrf(csrf -> csrf.disable())`:** this is a classroom demonstration decision, **not** a universal production recommendation. CSRF risk depends heavily on how credentials are transported and whether browsers automatically attach them. **CSRF disabled for learning/demo ≠ CSRF should always be disabled.**

**Role prefix convention**

```text
Database stores:  ROLE_ADMIN
.hasRole("ADMIN")  typically checks for  ROLE_ADMIN
                  …conceptually related to .hasAuthority("ROLE_ADMIN")
```

**Registration service (critical decisions)**

```java
public UserResponse register(RegisterRequest request) {
    User user = new User();
    user.setUsername(request.username());
    user.setPassword(passwordEncoder.encode(request.password()));
    user.setEnabled(true);

    Role role = roleRepository.findByName("ROLE_USER").orElseThrow();
    user.getRoles().add(role);

    User saved = userRepository.save(user);
    return new UserResponse(saved.getId(), saved.getUsername());
}
```

```text
Encode password · Assign default role on server · Return safe DTO
```

**Inspecting the current authentication (learning/debugging only)**

```java
@GetMapping("/me")
public Map<String, Object> me(Authentication authentication) {
    return Map.of(
        "username",    authentication.getName(),
        "authorities", authentication.getAuthorities(),
        "authenticated", authentication.isAuthenticated());
}
```

Do not expose sensitive internal security information in production without considering the security implications.

## 7.23 Demo Sequence (How to Prove It Works)

```text
1. Ensure ROLE_USER exists in the database.
2. POST /register { "username": "aditya", "password": "secret123" }
   → check DB: password = $2a$...  (raw password is NOT stored)
3. GET /me without credentials → authentication required.
4. GET /me with HTTP Basic aditya/secret123
   → UserDetails loaded → password verified → Authentication created
   → SecurityContext → authorization → /me response.
5. Try wrong password → authentication fails.
6. Set enabled = false in DB, retry correct password → still fails.
   Because authentication ≠ only password comparison; account state matters.
```

## 7.24 Filters in the Default Chain

Depending on enabled features, a default chain can contain:

```text
SecurityContextHolderFilter · CsrfFilter · LogoutFilter
UsernamePasswordAuthenticationFilter · BasicAuthenticationFilter
AnonymousAuthenticationFilter · ExceptionTranslationFilter · AuthorizationFilter
```

## 7.25 The Four Layers of the Entire Login Process

| Layer | Responsibility |
|---|---|
| **1 — Receive credentials** | `UsernamePasswordAuthenticationFilter` or `BasicAuthenticationFilter` |
| **2 — Represent the request** | `UsernamePasswordAuthenticationToken` |
| **3 — Authenticate the request** | `AuthenticationManager` → `ProviderManager` → `DaoAuthenticationProvider` |
| **4 — Load and verify user** | `UserDetailsService` → `UserRepository` → Database, plus `PasswordEncoder` verification |

Then: authenticated `Authentication` → `SecurityContext` → `SecurityContextHolder` → Authorization → Controller.

## 7.26 What Part 7 Adds

```text
Part 1 → Security concepts
Part 2/6 → Database user + password security
Part 7 → Internal authentication engine and request pipeline
```

```text
HTTP Request → Security Filters → Authentication Token → AuthenticationManager
→ ProviderManager → DaoAuthenticationProvider → UserDetailsService
→ Repository/Database → PasswordEncoder → Authenticated Authentication
→ SecurityContext → Authorization
```

> **Part 7 Core Principle:** Spring Security database authentication is a chain of responsibilities, not one magical login method. The filter extracts credentials, creates an authentication request, and delegates to the authentication engine. `ProviderManager` selects the appropriate `AuthenticationProvider`; `DaoAuthenticationProvider` loads the user through `UserDetailsService`, obtains the stored password and authorities, checks account state, and verifies the submitted password through `PasswordEncoder`. On success, Spring Security creates a trusted `Authentication`, stores it in the `SecurityContext`, and only then performs authorization before allowing the request to reach the controller.

---

# Part 8 — JWT Authentication & Stateless Design

> **Main goal:** understand why JWT authentication is needed, how a JWT works internally, how Spring Security issues and validates JWTs, and how a username/password login becomes a stateless bearer-token system.

> **The biggest idea: JWT does not replace username/password authentication during login. It changes how authentication is carried across future requests.**

## 8.1 Before vs With JWT

**Before JWT**

```text
Client → Username + Password → Authentication Filter → AuthenticationManager
→ DaoAuthenticationProvider → UserDetailsService → Database
→ Authenticated Authentication → SecurityContext
```

**With JWT — two distinct moments**

```text
LOGIN REQUEST
Client → Username + Password → AuthenticationManager → DaoAuthenticationProvider
→ Database + PasswordEncoder → Authenticated Authentication
→ JwtService → JwtEncoder → Signed JWT → Client

FUTURE PROTECTED REQUEST
Client → Authorization: Bearer <JWT> → JWT validation → Authentication
→ SecurityContext → Authorization → Controller
```

## 8.2 The Problem: Authentication on Every HTTP Request

```text
Request 1: GET /profile  Authorization: Basic ...  → authentication succeeds
Request 2: GET /profile  (no credentials)          → cannot be reconstructed automatically
```

> **Core principle: Authenticating one HTTP request does not automatically authenticate future requests.**

## 8.3 Two Ways to Remember Authentication

```text
Stateful:  server remembers the authenticated user        → HTTP Session + JSESSIONID cookie
Stateless: the request itself carries authentication material → Bearer JWT
```

(Full stateful mechanics are covered in [Part 4](#part-4--stateful-vs-stateless-authentication).)

## 8.4 `SessionCreationPolicy`

| Policy | Creates Session? | Uses Existing Session? | Mental Model |
|---|---:|---:|---|
| `ALWAYS` | Yes | Yes | Always session-based |
| `IF_REQUIRED` | When needed | Yes | Create when security requires it |
| `NEVER` | No | Yes | Do not create, but use an existing session |
| `STATELESS` | No for restoring security context | No for restoring it | Authenticate each request independently |

```java
// Session-based
.sessionManagement(session ->
    session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED))

// JWT
.sessionManagement(session ->
    session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
```

> **`STATELESS` ≠ "No data is ever stored by the application."** It specifically means the application does not rely on the HTTP session to restore authentication for the next request.

## 8.5 The Problem JWT Tries to Solve

If every request sends `Cookie: JSESSIONID=ABC123`, the server needs `ABC123 → session → SecurityContext → Authentication`, meaning server-side authentication state exists for every active session. JWT asks:

> *Can the client carry enough signed information for the server to reconstruct authentication without maintaining a separate HTTP session entry?*

```json
{ "sub": "aditya", "roles": ["ROLE_USER"], "iat": 1786570000, "exp": 1786573600 }
```

```text
JWT → Claims → Authentication → SecurityContext
```

## 8.6 JWT Is a Token Format, Not a Session Policy

> **JWT → token format. Stateless → authentication/session design.** They are not the same concept.

These notes focus on **JWT + STATELESS authentication**.

## 8.7 The Client-Carried Data Problem

What stops a client from changing `"roles": ["ROLE_USER"]` to `"roles": ["ROLE_ADMIN"]`? The server needs to determine: *was this issued by a trusted party, and was it modified after issuance?* The answer is a **cryptographic signature**.

### Integrity vs Authenticity

| Property | Meaning |
|---|---|
| **Integrity** | The protected data was not modified after it was signed |
| **Authenticity** | The data was created by someone possessing the trusted signing key |

```text
Payload says ROLE_USER → attacker changes to ROLE_ADMIN → signature no longer matches
```

## 8.8 Base64URL Is NOT Encryption

```json
{"sub":"42"}   →   eyJzdWIiOiI0MiJ9
```

This is **encoding**, not **encryption**. Anyone holding the token can generally decode the header and payload without the signing key.

> **JWT payload data should not be treated as secret simply because it is inside a JWT.**

Do not put passwords, secret API keys, credit card numbers, or private personal information inside a normal signed JWT payload.

## 8.9 Why Encoding Alone Is Insecure

Given `Token = Base64URL(payload)`, an attacker can decode the payload, change `STUDENT → ADMIN`, re-encode, and resend. The server would have no cryptographic evidence of unauthorized modification. So we need integrity **and** authenticity.

## 8.10 Why a Plain Hash Is Not Enough

`token = payload + SHA256(payload)` fails because the attacker can modify the payload, recalculate `SHA-256(modified payload)`, and send both. A plain hash answers *"does this hash match this data?"* — not *"was this data created by my trusted server?"*

## 8.11 What a Signature Adds

```text
Data + Trusted signing key → Signature
```

An attacker can modify the data but cannot create a valid signature without the signing key, so unauthorized modifications become detectable.

## 8.12 JWT Structure

```text
HEADER.PAYLOAD.SIGNATURE
xxxxx.yyyyy.zzzzz
```

### Header

```json
{ "alg": "HS256", "typ": "JWT" }
```

`alg` = signing algorithm; `typ` = token type.

### Payload (Claims)

```json
{
  "iss": "https://auth.coderarmy.in",
  "sub": "user-42",
  "aud": "course-api",
  "role": "STUDENT",
  "iat": 1786570000,
  "exp": 1893456000
}
```

| Claim | Meaning |
|---|---|
| `iss` | Issuer — who issued the token |
| `sub` | Subject — who/what the token represents (commonly identifies the principal) |
| `aud` | Audience — which application/API should accept it |
| `iat` | Issued At — when the token was created |
| `exp` | Expiration Time — when it should stop being accepted |
| `jti` | Unique identifier for the token (useful for tracking/revocation strategies) |

Not every claim is mandatory. The application/security protocol determines which claims exist and are validated.

**Custom claims** are allowed, e.g. `"department": "engineering"` — but remember the payload is normally readable, so avoid unnecessary secrets.

### Signature

```text
Base64URL(header) + "." + Base64URL(payload) → Signing Input → Cryptographic Signing → Signature

Final token: Base64URL(header) . Base64URL(payload) . Base64URL(signature)
```

> **Key idea: the signature covers the header and payload together.**

## 8.13 HS256 — Symmetric Signing

```text
Header.Payload + Secret Key → HMAC-SHA256 → Signature
```

The **same secret** is used for signing and verification.

```text
Issuer:   HMAC(message, secret) → Signature
Verifier: HMAC(message, same secret) → Expected signature → compare
```

**Advantages:** simple, fast, convenient — especially when one trusted application both issues and validates tokens.

**Limitation:** every service that must verify needs the **same secret**. Anyone with the shared secret can both verify tokens **and** create valid tokens. With many services (authentication server, course service, payment service, admin service), a compromised verifier could become capable of token issuance.

## 8.14 Asymmetric Signing

```text
Private Key → Signs token
Public Key  → Verifies token
```

Common algorithms: `RS256`, `ES256`.

```text
Authentication Server → Private Key + Public Key
Course Service        → Public Key only
Payment Service       → Public Key only
Admin Service         → Public Key only
```

**Why it is useful:** with one authentication server and many independent resource APIs, the issuer keeps the private key and the APIs receive only the public key, so compromising one API does not reveal the signing private key.

### Symmetric vs Asymmetric

| Property | HS256 | RS256 / ES256 |
|---|---|---|
| Key model | Shared secret | Private/public pair |
| Signing | Secret key | Private key |
| Verification | Same secret | Public key |
| Can verifier create token? | Yes, if it has the secret | No, if it only has the public key |
| Simplicity | High | More setup |
| Useful for multiple independent APIs | Less ideal | Very useful |

```text
Memory trick:
Symmetric  → SAME key for sign + verify
Asymmetric → PRIVATE signs, PUBLIC verifies
```

## 8.15 JWT Tampering Experiment

Original `A.B.C` where `A = Header`, `B = Payload`, `C = Signature`. Attacker changes `STUDENT → ADMIN`, producing new encoded payload `B2`. A correct modified token would have to be `A.B2.C2`, but the attacker does not know the signing key, so they may produce `A.B2.C`.

The server verifies `Expected Signature ≠ Token Signature` → **Invalid JWT**.

> **The signature does not physically prevent a person from editing the JWT. It makes unauthorized modifications detectable.**

## 8.16 Two Different Authentication Moments

| Moment | Flow |
|---|---|
| **Login** | Username + Password → database authentication → `Authentication` → JWT generation |
| **Future request** | JWT → JWT validation → `Authentication` → authorization |

```text
Username/password → used to obtain the JWT
JWT               → used for subsequent authentication
```

## 8.17 Spring Security Resource Server Support

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

> Resource Server support can protect an API using JWTs even when the same application is not implementing a full OAuth2 Authorization Server.

## 8.18 `JwtEncoder` vs `JwtDecoder`

| Component | Job |
|---|---|
| `JwtEncoder` | CREATE / SIGN JWT |
| `JwtDecoder` | READ / VERIFY / VALIDATE JWT |

```text
Login success → JwtEncoder → JWT
Future request → JWT → JwtDecoder → Trusted JWT claims
```

## 8.19 Secret & Key Configuration

For HS256 the application needs a strong secret:

```bash
openssl rand -base64 32
```

```properties
jwt.secret=${JWT_SECRET}
jwt.issuer=coderarmy-api
jwt.expiry=900        # 900 seconds = 15 minutes
```

> **Do not commit JWT secrets to Git.** A signing secret is sensitive — supply it via an environment variable, secret manager, or deployment configuration. **Signing secrets should be treated like passwords/private keys.**

```java
@Bean
public SecretKey jwtSecretKey(@Value("${jwt.secret}") String secret) {
    byte[] decodedKey = Base64.getDecoder().decode(secret);
    return new SecretKeySpec(decodedKey, "HmacSHA256");
}
```

```text
Base64 configuration value → Base64 decode → Raw key bytes → SecretKey
```

The same key is used by `JwtEncoder` and `JwtDecoder` for HS256.

```java
@Bean
public JwtEncoder jwtEncoder(SecretKey secretKey) {
    return NimbusJwtEncoder.withSecretKey(secretKey)
        .algorithm(MacAlgorithm.HS256)
        .build();
}
```

The application depends on the `JwtEncoder` abstraction, not the concrete `NimbusJwtEncoder`.

## 8.20 Login DTOs

```java
public class LoginRequest { String username; String password; }
public record LoginResponse(String accessToken) {}
```

```json
Request:  { "username": "aditya", "password": "123456" }
Response: { "accessToken": "eyJhbGciOiJIUzI1NiJ9..." }
```

## 8.21 `JwtService`

```java
@Service
public class JwtService {

    private final JwtEncoder jwtEncoder;
    private final String issuer;
    private final long expiry;

    public String generateToken(Authentication authentication) {
        Instant now = Instant.now();

        List<String> authorities = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList();

        JwtClaimsSet claims = JwtClaimsSet.builder()
            .issuer(issuer)
            .issuedAt(now)
            .expiresAt(now.plusSeconds(expiry))
            .subject(authentication.getName())
            .claim("authorities", authorities)
            .build();

        Jwt jwt = jwtEncoder.encode(JwtEncoderParameters.from(claims));
        return jwt.getTokenValue();
    }
}
```

The service receives an **already authenticated** `Authentication` and converts that trusted result into JWT claims.

> **`JwtService` should generate the token from an already authenticated user, not from raw client claims.**

| Claim builder | Produces |
|---|---|
| `.issuer(issuer)` | `"iss": "coderarmy-api"` — the decoder can later validate the expected issuer |
| `.subject(authentication.getName())` | `"sub": "aditya"` |
| `.issuedAt(now)` | `iat` |
| `.expiresAt(now.plusSeconds(expiry))` | `exp` — after this, the token should no longer be accepted |

### Why Put Authorities in the JWT?

```json
{ "authorities": ["ROLE_USER"] }
```

A future request can derive *who is this?* (`sub`) and *what can they do?* (`authorities`) **without** necessarily doing `JWT → username → database → UserDetails` on every request. This is a key idea behind the stateless design.

```text
Authentication
├── principal = aditya
├── authorities = ROLE_USER
└── authenticated = true
```

## 8.22 Login Controller

```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;

    @PostMapping("/login")
    public LoginResponse login(@RequestBody LoginRequest request) {

        Authentication authenticationRequest =
            UsernamePasswordAuthenticationToken.unauthenticated(
                request.username(), request.password());

        Authentication authentication =
            authenticationManager.authenticate(authenticationRequest);

        String token = jwtService.generateToken(authentication);
        return new LoginResponse(token);
    }
}
```

```text
Request DTO → Create unauthenticated Authentication → AuthenticationManager
→ Database authentication → Authenticated Authentication → JwtService → JWT → LoginResponse
```

The username/password mechanism is still exactly the same mechanism learned earlier; JWT generation happens **after** authentication succeeds.

## 8.23 JWT Validation — The Other Half

```http
Authorization: Bearer <JWT>
```

The server must not simply trust it:

```text
1. Read token
2. Verify signature
3. Validate claims
4. Treat result as trusted only after validation
5. Create Authentication
6. Continue with authorization
```

```java
@Bean
public JwtDecoder jwtDecoder(SecretKey secretKey, @Value("${jwt.issuer}") String issuer) {

    NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withSecretKey(secretKey)
        .macAlgorithm(MacAlgorithm.HS256)
        .build();

    decoder.setJwtValidator(JwtValidators.createDefaultWithIssuer(issuer));
    return decoder;
}
```

What the decoder checks:

```text
Is this structurally a valid JWT?
→ Can the signature be verified?
→ Was the correct trusted key used?
→ Are claims valid?
→ Is the issuer correct?
→ Is the token currently valid?
→ Only then → trusted JWT
```

> **Do not treat claims as trusted before cryptographic verification and claim validation succeed.**

## 8.24 `JwtAuthenticationConverter`

JWT claims are just token data; Spring Security needs `GrantedAuthority` objects. The token stores `{"authorities": ["ROLE_USER"]}`, but Spring Security Resource Server has standard authority mapping conventions commonly related to OAuth scopes — so the application explicitly tells Spring: *read the `authorities` claim, and do not add another prefix.*

```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {

    JwtGrantedAuthoritiesConverter authoritiesConverter =
        new JwtGrantedAuthoritiesConverter();
    authoritiesConverter.setAuthoritiesClaimName("authorities");
    authoritiesConverter.setAuthorityPrefix("");

    JwtAuthenticationConverter authenticationConverter =
        new JwtAuthenticationConverter();
    authenticationConverter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);

    return authenticationConverter;
}
```

**Why an empty prefix?** If Spring automatically added another prefix, the authority would become something like `SCOPE_ROLE_USER` and authorization rules would not match:

```text
JWT → ROLE_USER → GrantedAuthority("ROLE_USER") → .hasRole("USER") / .hasRole("ADMIN") work
```

## 8.25 Final JWT `SecurityFilterChain`

```java
@Bean
public SecurityFilterChain securityFilterChain(
        HttpSecurity http,
        JwtAuthenticationConverter jwtAuthenticationConverter) throws Exception {

    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/auth/login", "/register").permitAll()
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt ->
            jwt.jwtAuthenticationConverter(jwtAuthenticationConverter)));

    return http.build();
}
```

| Configuration | Meaning |
|---|---|
| `.csrf(csrf -> csrf.disable())` | Classroom demo choice — not a universal production rule |
| `SessionCreationPolicy.STATELESS` | Do not rely on the HTTP session to restore authentication |
| `/auth/login`, `/register` permitted | Users need these before they have a JWT |
| `/admin/**` `.hasRole("ADMIN")` | Requires `ROLE_ADMIN` |
| `anyRequest().authenticated()` | Requires an authenticated principal |
| `.oauth2ResourceServer(...)` | Use bearer-token JWT authentication |

### Why `oauth2ResourceServer().jwt()` for a Custom JWT?

Spring Security packages **bearer token authentication + JWT validation** inside Resource Server support.

```text
Mental model: My application issues JWT + Spring Security Resource Server validates JWT
```

## 8.26 Two Authentication Providers, Two Purposes

| Provider | Understands | Purpose |
|---|---|---|
| `DaoAuthenticationProvider` | Username + Password | Authenticate login credentials |
| `JwtAuthenticationProvider` | JWT | Authenticate future requests carrying bearer JWTs |

### Both Ultimately Produce `Authentication`

```text
Username/Password → DaoAuthenticationProvider → Authentication
JWT → JwtAuthenticationProvider → Authentication

Then: Authentication → SecurityContext → Authorization
```

This keeps the rest of Spring Security consistent.

## 8.27 Complete End-to-End Flow

**First request — login**

```text
POST /auth/login (username + password)
→ AuthenticationManager → DaoAuthenticationProvider → CustomUserDetailsService
→ Database → PasswordEncoder → Authentication → JwtService → JwtEncoder
→ Signed JWT → return token to client
```

**Future request — protected API**

```text
GET /profile, Authorization: Bearer <JWT>
→ Bearer JWT authentication → JwtDecoder → verify signature → validate claims
→ JwtAuthenticationConverter → create Authentication → SecurityContext
→ Authorization → Controller
```

## 8.28 Session vs JWT Authentication

| Session-Based | JWT-Based |
|---|---|
| Server stores authentication state | Authentication is reconstructed from the token |
| Client commonly sends `JSESSIONID` | Client sends bearer JWT |
| Session ID points to server-side state | Token carries signed claims |
| Commonly stateful | Can be stateless |
| Server restores `SecurityContext` from session | Server validates token and creates `Authentication` per request |
| Session invalidation can directly remove server-side state | Token lifetime/validation determine acceptance |

```text
Session: "Here is an ID. Server, remember me."
JWT:     "Here is my signed authentication material. Verify it and reconstruct me."
```

## 8.29 JWT vs Encryption, and What JWT Guarantees

```text
Header + Payload → readable
Signature        → protects integrity/authenticity

JWT is NOT encrypted user data.
It is primarily signed claims (unless an additional encryption mechanism is explicitly used).
```

A normal signed JWT does not hide `sub`, `roles`, `iss`, `aud`, `iat`, or `exp`.

**A signed JWT can provide:** integrity, authenticity, expiration enforcement, issuer validation, audience validation — **provided the application actually performs the relevant validation checks.**

**It does NOT automatically provide:** confidentiality, because the payload is normally readable.

## 8.30 If Someone Steals a Valid JWT

A bearer JWT is a credential — possession may be sufficient to present it. Controls:

```text
HTTPS · Short expiration · Secure storage strategy · Avoid logging tokens
Strong signing keys · Proper issuer/audience validation · Appropriate authorities/scopes
```

**Why expiration matters:** a stolen token should not remain valid forever; a short-lived access token reduces the lifetime of a stolen credential. These notes use 900 seconds = 15 minutes.

## 8.31 Authorities in the JWT — Power and Trade-off

```json
{ "sub": "aditya", "authorities": ["ROLE_ADMIN"] }
```

```java
.hasRole("ADMIN")   // can be satisfied without loading roles from the DB on every request
```

> **Trade-off: changes to the user's roles may not take effect for already-issued tokens until the token expires or another revocation strategy is used.**

## 8.32 Statelessness vs Immediate Revocation

```text
Session: Session exists on server → delete session → authentication immediately invalidated
JWT:     Token issued → token remains valid until validation says it is no longer valid
```

Stateless tokens often trade some easy server-side session invalidation for reduced dependence on server-side authentication session state.

## 8.33 Important Mental Model — JWT Does Not Eliminate Authentication

```text
Session model: Login → Authentication → Server session → future request uses JSESSIONID
JWT model:     Login → Authentication → JWT generated
               → future request uses Bearer JWT → JWT validated → Authentication reconstructed
```

### Token Creation vs Token Validation Are Different Jobs

```text
Creation:   Authentication → JwtService → JwtEncoder → Signed JWT
Validation: Incoming JWT → JwtDecoder → signature verification → claims validation
            → Trusted JWT → Authentication
```

## 8.34 JWT Security Checklist

```text
[ ] Strong signing key
[ ] Secret not committed to Git
[ ] HTTPS in real deployment
[ ] Signature verification enabled
[ ] Expiration validated
[ ] Issuer validated
[ ] Audience validated when applicable
[ ] Authorities sourced from trusted claims
[ ] Sensitive data kept out of payload
[ ] Short-lived access token considered
[ ] Authorization rules still enforced server-side
[ ] Appropriate token storage strategy
[ ] Avoid logging access tokens
```

> **Part 8 Core Principle:** JWT authentication is not a replacement for authentication itself. It is a way to carry trusted authentication information between requests without relying on an HTTP session. The initial username/password login still uses the database authentication pipeline. After successful authentication, a `JwtEncoder` creates a signed token containing trusted claims such as subject, issuer, expiration, and authorities. Future requests send that token as a bearer credential; Spring Security validates it with a `JwtDecoder`, converts trusted claims into an `Authentication`, places it in the `SecurityContext`, and then performs authorization. The central security property is not that JWT data cannot be edited — it is that unauthorized edits make the signature invalid and therefore detectable.

---

# Part 9 — Master Comparison Tables

## 9.1 Security Concepts

| A | B | Distinction |
|---|---|---|
| Authentication | Authorization | Authentication → *who are you?* · Authorization → *what are you allowed to do?* |
| Identity | Principal | Identity → conceptual entity · Principal → identity represented inside the security system |
| Identity | Account | Identity → conceptual subject · Account → one stored representation of it |
| Credential | Authentication | Credential → evidence · Authentication → attempt/result + identity + authorities |
| Role | Authority | Role → broader grouping · Authority → specific capability/security attribute |
| `SecurityContext` | `SecurityContextHolder` | Context → container holding `Authentication` · Holder → access point to the current context |
| Threat | Vulnerability | Threat → potential danger · Vulnerability → weakness that enables an attack |
| Authentication failure | Security attack | Failure → acceptable evidence not established · Attack → deliberate attempt to defeat controls |

## 9.2 Statefulness & Tokens

| A | B | Distinction |
|---|---|---|
| Stateful | Stateless | Stateful → server-side HTTP session persists authentication · Stateless → each request carries the credential |
| JWT | Stateless | JWT → token format · Stateless → authentication/session design |
| Cookie | Session | Cookie → browser transport/storage · Session → server-side state |
| Authentication mechanism | Session policy | Mechanism → how credentials are verified · Policy → whether auth is persisted in the HTTP session |
| Bearer | Opaque | Bearer → how possession is used · Opaque → how the token is represented |
| Self-contained | Opaque token | Self-contained → signed, locally verifiable claims (JWT) · Opaque → meaningless value, checked via token service |

## 9.3 Architecture Components

| A | B | Distinction |
|---|---|---|
| `Authentication` filter | `AuthenticationManager` | Filter extracts credentials & creates the request · Manager authenticates that request |
| `AuthenticationManager` | `ProviderManager` | Interface / entry point · Common implementation delegating to providers |
| `ProviderManager` | `DaoAuthenticationProvider` | Chooses/delegates to a provider · Performs database username/password authentication |
| `AuthenticationProvider` | `DaoAuthenticationProvider` | Contract for handling a type of authentication · The database username/password implementation |
| `FilterChainProxy` | `SecurityFilterChain` | Manages/selects a chain · Matcher + ordered security filters |
| `SecurityFilterChain` | `UserDetailsService` | How requests are secured · How user information is loaded for authentication |
| `HttpSecurity` | `SecurityFilterChain` | Configuration builder · Built security pipeline |
| `UserDetails` | `UserDetailsService` | *What* one security user looks like (Result) · *How* Spring Security obtains it (Finder) |
| `UserDetailsService` | `UserRepository` | Spring Security adapter/bridge · Domain/data-access component |
| Domain `User` | Spring Security `UserDetails` | Application/domain entity · Security representation |
| `PasswordEncoder` | `BCryptPasswordEncoder` | Abstraction · Concrete implementation |
| `JwtEncoder` | `JwtDecoder` | Create/sign tokens · Decode/verify/validate tokens |
| `DaoAuthenticationProvider` | `JwtAuthenticationProvider` | Understands username/password · Understands JWT |

## 9.4 Password Operations

| A | B | Distinction |
|---|---|---|
| `encode()` | `matches()` | Create stored representation (registration) · Verify raw password (login) |
| Password validation | Password encoding | Is the input acceptable? · How is the representation stored? |
| Password hashing | Password encryption | One-way, irreversible · Reversible with a key |
| Fast hash (SHA-256) | Adaptive hash (BCrypt) | Cheap guesses → attacker tests fast · Expensive guesses → large-scale offline guessing slowed |
| Form Login | HTTP Basic | `UsernamePasswordAuthenticationFilter` · `BasicAuthenticationFilter` — but both can share the same authentication backend |

## 9.5 HTTP Status

| Status | Meaning |
|---|---|
| `401 Unauthorized` | Authentication missing or failed |
| `403 Forbidden` | Request understood but rejected — may be a missing CSRF token, not only an insufficient role |

## 9.6 JWT Signing

| Property | HS256 | RS256 / ES256 |
|---|---|---|
| Key model | Shared secret | Private/public pair |
| Signing | Secret key | Private key |
| Verification | Same secret | Public key |
| Verifier can create tokens? | Yes, if it has the secret | No, if it only has the public key |
| Simplicity | High | More setup |
| Multiple independent APIs | Less ideal | Very useful |

---

# Part 10 — Interview Q&A

## 10.1 Foundations & Identity

**Q1. What is Spring Security?**
A framework used to secure Spring applications, primarily providing authentication, authorization, security-context management, session security, common web-security protections, and integration with security protocols.

**Q2. What is authentication?**
The process of verifying an identity claim.

**Q3. What is authorization?**
The decision about whether an identity can perform an action on a resource under the applicable rules and context.

**Q4. Authentication vs authorization?**
Authentication → identity; Authorization → access.

**Q5. What is a principal?**
The identity currently represented inside the security system.

**Q6. What are credentials?**
Evidence used to prove an identity claim.

**Q7. What are authorities?**
Granted capabilities/security attributes associated with a principal.

**Q8. What is a role?**
A broader grouping of privileges, commonly represented with a `ROLE_` prefix in Spring Security authority conventions.

**Q9. What is `Authentication`?**
It can represent an authentication attempt and, after trusted verification, the authenticated principal, authorities, details, and authentication state.

**Q10. What is `SecurityContext`?**
A container holding the current `Authentication`.

**Q11. What is `SecurityContextHolder`?**
A central access point for the current `SecurityContext`.

**Q12. Why is `ThreadLocal` involved?**
In the default servlet model, security context is associated with the current request-processing thread, allowing components on that thread to access it.

**Q13. Why clear the `SecurityContext`?**
Because server threads are reused; clearing prevents one request's authentication from leaking into another request.

**Q14. What is stateful authentication?**
Authentication that relies on server-side HTTP session state persisted between requests.

**Q15. What is stateless authentication?**
Authentication where the next request carries the credential needed to establish authentication instead of relying on a previously stored HTTP login session.

**Q16. Does JWT automatically mean stateless?**
No. JWT is a token format; session policy is a separate design decision.

**Q17. Does a cookie automatically mean stateful authentication?**
No. A cookie can carry many kinds of data, including an access token.

**Q18. What is a bearer token?**
A credential that can generally be presented by whoever possesses it.

**Q19. What is an opaque token?**
A token whose contents are not meaningful to the consuming application and may be checked through a token service/introspection mechanism.

**Q20. Why can a logged-in POST return 403?**
Because authentication success does not disable CSRF protection; a missing CSRF token can cause rejection of a state-changing request.

## 10.2 Spring Boot Defaults

**Q21. What is `SecurityFilterChain`?**
A configuration describing how incoming web requests are processed and protected by Spring Security filters and authorization rules.

**Q22. What does `UserDetailsService` do?**
It loads user-specific data used by the authentication process.

**Q23. What happens when you define your own `SecurityFilterChain`?**
Spring Boot backs off from its default web-security configuration, while user/authentication configuration is a separate concern.

**Q24. What is the default Spring Boot user?**
Conceptually `username = user`, `password = generated`, `authority = ROLE_USER`, for the default development configuration.

## 10.3 Request Pipeline & Filters

**Q25. How does a request enter Spring Security?**
A servlet request passes through the servlet filter mechanism. `DelegatingFilterProxy` bridges the servlet container to Spring-managed security infrastructure, which leads to `FilterChainProxy` and the selected `SecurityFilterChain`.

**Q26. What is `DelegatingFilterProxy`?**
A servlet filter that delegates request processing to a Spring-managed filter bean.

**Q27. What is `FilterChainProxy`?**
The central Spring Security servlet filter that selects the first matching `SecurityFilterChain` and executes its filters.

**Q28. What is a `SecurityFilterChain`?**
A request matcher plus an ordered list of security filters.

**Q29. What happens if multiple `SecurityFilterChain`s match?**
The first matching chain wins. Spring Security does not combine the matching chains.

**Q30. Why should specific chains come before `/**`?**
Because the first matching chain wins, and `/**` is very broad.

**Q31. What is `HttpSecurity`?**
A builder used to configure and construct a `SecurityFilterChain`.

**Q32. What is an authentication filter's job?**
Extract credentials, create an `Authentication` request object, and hand it to the authentication engine.

**Q33. What is `UsernamePasswordAuthenticationToken`?**
An `Authentication` implementation commonly used for username/password authentication. It can represent the initial unverified request and, after success, the authenticated result.

**Q34. What is `AuthenticationManager`?**
The entry point into the authentication engine.

**Q35. What is `ProviderManager`?**
A common `AuthenticationManager` implementation that delegates authentication to configured `AuthenticationProvider` instances.

**Q36. What is `AuthenticationProvider`?**
A component responsible for authenticating a particular kind/type of authentication request.

**Q37. What is `DaoAuthenticationProvider`?**
An `AuthenticationProvider` designed for username/password authentication using `UserDetailsService` and `PasswordEncoder`.

**Q38. What does `supports()` do?**
It tells Spring Security whether a provider can handle a particular `Authentication` type.

**Q39. What is the role of `UserDetailsService`?**
Load the user's security information from the application's user store.

**Q40. What is the role of `UserDetails`?**
Represent one user's security information in a form Spring Security understands.

**Q41. Where does password verification happen?**
Conceptually through the configured `PasswordEncoder`, commonly inside `DaoAuthenticationProvider` during username/password authentication.

**Q42. What happens after successful authentication?**
An authenticated `Authentication` is placed into the `SecurityContext`, accessible through `SecurityContextHolder`, and authorization rules can then be evaluated.

**Q43. Why are credentials erased?**
To avoid keeping sensitive authentication material, such as a raw password, longer than necessary.

**Q44. Do form login and HTTP Basic need different database authentication logic?**
No. They use different filters for different credential transport formats but can share the same `AuthenticationManager`/provider/database authentication pipeline.

## 10.4 Database Authentication & Passwords

**Q45. Why is database authentication needed?**
Because the default generated user is mainly for development/testing and cannot represent persistent real-world users.

**Q46. Why doesn't Spring Security directly use our `User` entity?**
Because applications have different schemas and domain models. Spring Security uses standard abstractions such as `UserDetails`.

**Q47. What is `UserDetails`?**
It represents the username, password, authorities, and account-status information Spring Security needs for authentication.

**Q48. What is `UserDetailsService`?**
A service abstraction used to load the user's security information by the login identifier.

**Q49. What is the role of `DaoAuthenticationProvider`?**
It coordinates database-backed username/password authentication using `UserDetailsService` and password verification.

**Q50. `UserRepository` vs `UserDetailsService`?**
`UserRepository` → data access for the domain entity; `UserDetailsService` → security authentication bridge.

**Q51. Why should passwords not be encrypted?**
Because encryption is reversible. If the database and decryption key are compromised, the original passwords may be recovered.

**Q52. Why use hashing?**
Because login only needs to verify whether the presented password matches the registered password.

**Q53. Why isn't SHA-256 ideal for passwords?**
Because it is fast, allowing attackers to test password guesses rapidly offline.

**Q54. What is BCrypt?**
An adaptive password-hashing algorithm supported through Spring Security's `BCryptPasswordEncoder`.

**Q55. Why are two BCrypt hashes of the same password different?**
Because BCrypt uses random salt.

**Q56. How does `matches()` work if hashes differ?**
It uses the salt and parameters represented in the stored BCrypt value to verify the raw password.

**Q57. `encode()` vs `matches()`?**
`encode()` → registration/password creation; `matches()` → login/password verification.

**Q58. Why should the username be unique?**
Because the login identifier must map to exactly one account.

**Q59. Why use many-to-many `User ↔ Role`?**
Because one user → many roles and one role → many users.

**Q60. Why can the `User → Role` mapping remain unidirectional?**
If the application only needs `user.getRoles()` and not `role.getUsers()`, there is no need to create reverse navigation.

## 10.5 JWT

**Q61. Why do we need JWT if session authentication already works?**
JWT can carry signed authentication information with each request, allowing the application to authenticate requests without relying on server-side HTTP session state.

**Q62. What is JWT?**
JSON Web Token — a compact token format commonly used to carry signed claims.

**Q63. What are the three parts of a signed JWT?**
`HEADER.PAYLOAD.SIGNATURE`.

**Q64. Is JWT encrypted?**
A normal signed JWT is not encrypted. Its header and payload are Base64URL-encoded and readable.

**Q65. What does the JWT signature provide?**
It helps establish integrity and authenticity of the signed header/payload.

**Q66. Why is a plain SHA-256 hash not enough?**
Because an attacker can modify the payload and calculate a new SHA-256 hash. A signature depends on a secret/private key the attacker should not possess.

**Q67. What is HS256?**
HMAC using SHA-256 with a shared secret.

**Q68. Difference between symmetric and asymmetric signing?**
Symmetric: the same shared secret signs and verifies. Asymmetric: private key signs, public key verifies.

**Q69. Why use asymmetric signing in microservices?**
Multiple services can verify tokens with the public key without receiving the private signing key.

**Q70. What is `JwtEncoder`?**
Spring Security abstraction used to create/sign JWTs.

**Q71. What is `JwtDecoder`?**
Spring Security abstraction used to decode, verify, and validate incoming JWTs.

**Q72. Why use `oauth2ResourceServer().jwt()`?**
Spring Security's Resource Server support provides bearer-token JWT authentication/validation for protected APIs.

**Q73. What is `SessionCreationPolicy.STATELESS`?**
It tells Spring Security not to rely on HTTP session state for restoring authentication between requests.

**Q74. Why put authorities in a JWT?**
To allow future requests to reconstruct the user's authorities from the trusted token without necessarily querying the database for roles on every request.

**Q75. What happens if a JWT payload is modified?**
The payload changes, so the signing input changes. Unless the attacker can create a new valid signature with the trusted signing key, signature verification fails.

**Q76. Does JWT prevent editing?**
No. People can edit the encoded payload. The signature makes unauthorized editing detectable.

**Q77. Is JWT completely stateless?**
This design uses `SessionCreationPolicy.STATELESS`, so it does not rely on HTTP-session authentication state. The overall application can still store users, roles, logs, business data, etc.

**Q78. Why validate `iss` and `exp`?**
To ensure the correct issuer and that the token is not expired before trusting its claims.

**Q79. Why use a short JWT lifetime?**
To reduce the usable lifetime of a stolen access token.

**Q80. Why is `JwtAuthenticationConverter` needed in this example?**
Because the application stores authorities in a custom `authorities` claim and wants Spring Security to convert those values into `GrantedAuthority` without adding another prefix.

---

# Part 11 — Common Mistakes

## 11.1 Foundations

| # | Mistake | Correction |
|---|---|---|
| 1 | Authentication = authorization | Authentication → identity; authorization → access |
| 2 | Hiding an admin button is security | A malicious client can call the endpoint directly |
| 3 | JWT means stateless | JWT is a format; session policy is separate |
| 4 | Cookie means session | A cookie can carry a session ID, token, CSRF token, or unrelated data |
| 5 | Stateless means no server-side data | It means authentication does not depend on a previously stored HTTP login session |
| 6 | An authenticated user can do everything | Authorization still determines access |
| 7 | The client can send `role=ADMIN` | Authorities must come from trusted security infrastructure |
| 8 | `SecurityContextHolder` permanently stores the logged-in user | The servlet model establishes and clears request security context |
| 9 | 403 always means wrong role | A missing CSRF token can also cause 403 for state-changing requests |
| 10 | `SecurityFilterChain` and `UserDetailsService` do the same job | Chain → request security; Service → user loading |

## 11.2 Filters & Authentication Engine

| # | Mistake | Correction |
|---|---|---|
| 11 | `SecurityFilterChain` is the same as `FilterChainProxy` | `FilterChainProxy` manages/selects a (`SecurityFilterChain` = matcher + ordered filters) |
| 12 | Every authentication filter authenticates every request | Filters look for requests containing the mechanism they handle |
| 13 | `UsernamePasswordAuthenticationToken` always means an authenticated user | It can be an unauthenticated request and later an authenticated result — state matters |
| 14 | The authentication filter checks the database | It extracts credentials and delegates to the authentication engine |
| 15 | `AuthenticationManager` directly queries the database | Common flow: Manager → `ProviderManager` → `DaoAuthenticationProvider` → `UserDetailsService` → Repository → DB |
| 16 | `DaoAuthenticationProvider` stores users | It authenticates using `UserDetailsService` + `PasswordEncoder` |
| 17 | `UserDetailsService` is the database | It is the contract for finding/loading the user |
| 18 | `CustomUserDetails` is mandatory | The built-in `User` implementation can be used |
| 19 | If the password is correct, login always succeeds | Account state (disabled/locked/expired) can reject authentication |
| 20 | Registration can trust a client-supplied role | The server must assign privileges |
| 21 | Form login and Basic are completely different systems | Credential transport/filter differs, but they can share the same authentication engine |

## 11.3 Passwords

| # | Mistake | Correction |
|---|---|---|
| 22 | "I can store the password directly in MySQL" | Never intentionally store raw passwords |
| 23 | "I should encrypt passwords" | Authentication does not require recovering the original; use hashing |
| 24 | "SHA-256 is enough for password storage" | It is a fast general-purpose hash, helping attackers guess quickly |
| 25 | "BCrypt produces the same hash every time" | BCrypt uses random salting, so values differ |
| 26 | "During login I should call `encode()` and compare strings" | Use `matches(rawPassword, storedEncodedPassword)` |
| 27 | "`User` and `UserDetails` are the same thing" | They belong to different layers |
| 28 | "`UserDetailsService` is the repository" | It may use a repository but is the security abstraction |
| 29 | "Username must literally be a username" | It can be email, employee ID, phone, etc. (but must be unique) |
| 30 | "Every account status needs a database column" | Only model the states your application needs |
| 31 | "Password validation and encoding are the same" | They solve different problems |
| 32 | "The client can send its own `ROLE_ADMIN`" | Never trust a client-provided authority |
| 33 | "`passwordEncoder.encode(raw).equals(stored)` works for BCrypt" | Wrong for salted storage — use `matches()` |

## 11.4 JWT

| # | Mistake | Correction |
|---|---|---|
| 34 | "JWT encrypts the payload" | Normal signed JWT payloads are encoded, not encrypted |
| 35 | "Base64URL is security" | It is encoding |
| 36 | "If I change the role inside a JWT, the server will see it" | Only if the server fails to validate the signature |
| 37 | "SHA-256 is enough as a JWT signature" | A plain public hash is not a signature — an attacker can recompute it |
| 38 | "JWT replaces username/password login" | Username/password still authenticate the login request; JWT is then issued |
| 39 | "`JwtEncoder` authenticates the user" | It creates/signs a token from already trusted information |
| 40 | "`JwtDecoder` creates the JWT" | `JwtEncoder` creates/signs; `JwtDecoder` decodes/verifies/validates |
| 41 | "Every JWT verifier should have the private key" | Not with asymmetric signing — private signs, public verifies |
| 42 | "HS256 is always better because it is simpler" | It depends on architecture; asymmetric is better for many independent APIs |
| 43 | "STATELESS means my application cannot store anything" | It means authentication is not restored from the HTTP session between requests |
| 44 | "JWT means no authentication processing on future requests" | Every future request still needs token validation and authentication reconstruction |
| 45 | "JWT roles are automatically understood by Spring Security" | A custom `authorities` claim may need a `JwtAuthenticationConverter` |
| 46 | "The client can choose its own role when requesting a JWT" | The JWT must be generated from trusted server-side authentication/authority data |
| 47 | "Secrets in `application.properties` are fine to commit" | Signing secrets should be treated like passwords — use env vars/secret managers |

---

# Part 12 — Final Cheat Sheet

## 12.1 The Two Questions

```text
Authentication → WHO are you?
Authorization  → WHAT are you allowed to do?
```

## 12.2 The Identity Chain

```text
Identity → Credentials → Authentication (Principal, Credentials, Authorities, Status)
         → SecurityContext → SecurityContextHolder
```

## 12.3 The Filter Pipeline

```text
Tomcat → DelegatingFilterProxy → FilterChainProxy
       → first matching SecurityFilterChain (matcher + ordered filters)
       → Authentication Filter → AuthenticationManager → ProviderManager
       → DaoAuthenticationProvider → UserDetailsService → Repository → Database
       → PasswordEncoder → Authenticated Authentication
       → SecurityContext → SecurityContextHolder → Authorization → Controller
```

## 12.4 Class → Responsibility

| Component | One-line role |
|---|---|
| `DelegatingFilterProxy` | Bridges the servlet container to Spring-managed security beans |
| `FilterChainProxy` | Selects and executes the first matching `SecurityFilterChain` |
| `SecurityFilterChain` | Matcher + ordered security filters |
| `HttpSecurity` | Builder that constructs a `SecurityFilterChain` |
| Authentication filter | Extracts credentials and creates an `Authentication` request |
| `UsernamePasswordAuthenticationToken` | Username/password `Authentication` (unauthenticated or authenticated) |
| `AuthenticationManager` | Entry point into the authentication engine |
| `ProviderManager` | Delegates to the provider that `supports()` the token type |
| `AuthenticationProvider` | Authenticates one kind of authentication request |
| `DaoAuthenticationProvider` | Database username/password authentication |
| `JwtAuthenticationProvider` | Authenticates bearer JWTs |
| `UserDetails` | Security view of one user (Result) |
| `UserDetailsService` | How to find that user (Finder) |
| `UserRepository` | Domain data access |
| `PasswordEncoder` | `encode()` for registration, `matches()` for login |
| `BCryptPasswordEncoder` | Adaptive, salted, one-way password hashing |
| `SecurityContext` | Holds the current `Authentication` |
| `SecurityContextHolder` | Access point to the current `SecurityContext` |
| `JwtService` | Builds claims from an authenticated `Authentication` |
| `JwtEncoder` / `JwtDecoder` | Sign tokens / verify and validate tokens |
| `JwtAuthenticationConverter` | Maps JWT claims to `GrantedAuthority` |

## 12.5 Password Rules

```text
NEVER store raw passwords          → store BCrypt encoded values
Never encrypt passwords            → use one-way adaptive hashing
Registration → encode()            Login → matches()
BCrypt uses random salt            → same password → different encoded values
matches() reads salt + parameters from the stored value
Validation (is input acceptable?) ≠ Encoding (how is it stored?)
```

## 12.6 The `hasRole` Convention

```text
Database: ROLE_ADMIN
hasRole("ADMIN")     → checks ROLE_ADMIN
hasAuthority("ROLE_ADMIN") → equivalent conceptual check
```

## 12.7 Status Codes

```text
401 → authentication missing or failed
403 → request understood but rejected (role OR missing CSRF token)
```

## 12.8 Session vs JWT

```text
Session: state on server, JSESSIONID points to it, easy server-side invalidation, stateful
JWT:     signed claims carried by the client, verified per request, stateless-friendly,
         revocation is harder (expiry limits the damage)
```

## 12.9 JWT in Three Lines

```text
HEADER.PAYLOAD.SIGNATURE
Header & Payload  → Base64URL encoded (readable — NOT encrypted)
Signature         → protects header + payload (integrity + authenticity)
```

## 12.10 JWT Signing

```text
HS256        → one shared secret for sign + verify (simple, single trusted app)
RS256/ES256  → private key signs, public key verifies (many independent APIs)
```

## 12.11 JWT Request Lifecycle

```text
Login:   username+password → AuthenticationManager → DaoAuthenticationProvider
         → Authentication → JwtService → JwtEncoder → JWT → client

Request: Authorization: Bearer <JWT> → JwtDecoder → verify signature + validate claims
         → JwtAuthenticationConverter → Authentication → SecurityContext
         → Authorization → Controller
```

## 12.12 The 20 Things Worth Memorizing

```text
1.  Spring Security is not just a login system.
2.  Authentication = WHO · Authorization = WHAT.
3.  Authorization depends on subject + action + resource + context.
4.  The browser is untrusted; frontend checks are UX, not security.
5.  Authentication runs before authorization, which runs before the controller.
6.  First matching SecurityFilterChain wins; order specific before /**.
7.  Authentication filters extract credentials — they do not query the database.
8.  AuthenticationManager is the entry point; ProviderManager is the common implementation.
9.  ProviderManager picks a provider via supports().
10. DaoAuthenticationProvider = UserDetailsService + PasswordEncoder.
11. UserDetails = one user (Result). UserDetailsService = how to find it (Finder).
12. Account state is part of authentication, not just the password.
13. Registration → encode(). Login → matches().
14. Never store raw passwords; never encrypt them — hash them adaptively (BCrypt).
15. BCrypt salts randomly, so hashes differ; matches() reads the salt from the stored value.
16. SecurityContext holds Authentication; SecurityContextHolder exposes it; it must be cleared.
17. Stateful = server-side HTTP session. Stateless = credential on each request.
18. JWT is a token format — not automatically stateless.
19. Base64URL is encoding, not encryption; the payload is readable.
20. The signature does not prevent editing — it makes unauthorized edits detectable.
```

---

# Part 13 — Complete Concept Map

## 13.1 The Whole System on One Diagram

```text
                        CLIENT (untrusted)
                              │
                              ↓
                        HTTP REQUEST
                              │
                              ↓
                    TRUST BOUNDARY  ────────►  assets, threats, vulnerabilities
                              │
                              ↓
                          TOMCAT
                              │
                              ↓
                  DelegatingFilterProxy          (bridges servlet container ↔ Spring beans)
                              │
                              ↓
                    FilterChainProxy              (selects a chain)
                              │
                     first matching chain wins
                              │
                              ↓
                   SecurityFilterChain           (RequestMatcher + ordered filters)
                    (built by HttpSecurity)
                              │
                              ↓
                  Authentication Filter           (UsernamePasswordAuthenticationFilter
                              │                    or BasicAuthenticationFilter)
                              ↓
              UsernamePasswordAuthenticationToken (authenticated = false)
                              │
                              ↓
                   AuthenticationManager          (entry point; interface)
                              │
                              ↓
                     ProviderManager              (picks a provider via supports())
                              │
                              ↓
                 DaoAuthenticationProvider        (the workhorse)
                     ┌────────┴────────┐
                     ↓                 ↓
           UserDetailsService     PasswordEncoder
                     │                 │
                     ↓                 ↓
             UserRepository          BCrypt
                     │
                     ↓
                  Database
                     │
                     ↓
              User Entity → (adapter) → UserDetails
                     │
                     └──────────► password verification (matches)
                              │
                              ↓
              Authenticated Authentication (authenticated = true, credentials cleared)
                              │
                              ↓
                      SecurityContext             (holds the Authentication)
                              │
                              ↓
                   SecurityContextHolder          (ThreadLocal access point)
                              │
                              ↓
                       AUTHORIZATION               (hasRole / hasAuthority / authenticated)
                              │
                     ┌────────┴────────┐
                   DENY             ALLOW
                     │                 │
                     ↓                 ↓
                 401 / 403      DispatcherServlet
                                       │
                                       ↓
                                   Controller → Service → Repository → Database
```

## 13.2 Stateful Path

```text
POST /login → authenticate → Authentication → SecurityContext → HTTP Session
            → JSESSIONID to client

GET /profile + JSESSIONID → find session → SecurityContext → Authentication
            → authorize → controller → clear context
```

## 13.3 Stateless JWT Path

```text
LOGIN
username + password → AuthenticationManager → DaoAuthenticationProvider
                    → Database + PasswordEncoder → Authenticated Authentication
                    → JwtService → JwtEncoder → Signed JWT → client

FUTURE REQUEST
Authorization: Bearer <JWT> → JwtDecoder → verify signature → validate claims
                            → JwtAuthenticationConverter → Authentication
                            → SecurityContext → Authorization → Controller
```

## 13.4 JWT Anatomy

```text
                        JWT
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
       HEADER         PAYLOAD       SIGNATURE
          │              │              │
      alg / typ       claims      protects the
                       │          encoded header
      ┌────────────────┼────────────────┐   + payload
      ↓        ↓       ↓        ↓       ↓
     iss      sub     aud      iat     exp     (jti)
                       │
                       ↓
                 authorities

Signing:  Base64URL(header) + "." + Base64URL(payload)
          → Signing Input → Cryptographic Sign → Signature

Final:    Base64URL(header) . Base64URL(payload) . Base64URL(signature)
```

## 13.5 Authorization Inputs

```text
AUTHORIZATION
      │
Subject + Action + Resource + Context
      │
      └──► e.g. hasRole("ADMIN"), hasAuthority("COURSE_WRITE"),
                authenticated(), permitAll()
```

## 13.6 Database Model

```text
users                     roles
id | username | password  id | name
   | enabled                 | ROLE_USER / ROLE_ADMIN
        │                    │
        └──── user_roles ────┘
              (user_id, role_id)

Role → GrantedAuthority → Authentication → Authorization
```

## 13.7 Lecture Progression (How Everything Connects)

```text
Lecture 1  → Security concepts, identity model, stateful vs stateless
Lecture 2  → Database users, roles, UserDetails, password hashing (BCrypt)
Lecture 3  → Filters, AuthenticationManager, ProviderManager, complete login flow
Lecture 4  → JWT issuance & validation, STATELESS design, authentication reconstruction
```

```text
Security concepts
      ↓
Database authentication
      ↓
Authentication internals
      ↓
JWT stateless authentication
```

---

## Final One-Minute Revision (All Four Lectures)

> **Spring Security protects assets by establishing identity before business logic runs. A browser is untrusted, so authorization must live on the server. `Authentication` answers "who are you" and holds principal, credentials, authorities and status; it is stored in the `SecurityContext` and reached through `SecurityContextHolder`, which is `ThreadLocal`-backed and must be cleared because server threads are reused. In a real system, users live in a database behind the `UserDetails` / `UserDetailsService` abstraction, with roles mapped through a many-to-many join table into `GrantedAuthority`. Passwords are never stored raw and never encrypted — they are hashed adaptively with BCrypt, using `encode()` at registration and `matches()` at login, with the salt stored inside the encoded value. Requests enter through the servlet filter chain: Tomcat → `DelegatingFilterProxy` → `FilterChainProxy` → the first matching `SecurityFilterChain`, which is built by `HttpSecurity`. An authentication filter — `UsernamePasswordAuthenticationFilter` for form login or `BasicAuthenticationFilter` for HTTP Basic — extracts credentials, creates an unauthenticated `UsernamePasswordAuthenticationToken`, and hands it to the `AuthenticationManager`. `ProviderManager` selects a provider via `supports()`, and `DaoAuthenticationProvider` loads the user through `UserDetailsService`, checks account status, and verifies the password with `PasswordEncoder`, producing a trusted authenticated `Authentication` that goes into the `SecurityContext` before authorization runs. Across requests, authentication can be stateful — server-side HTTP session with `JSESSIONID`, easily revocable — or stateless: after normal username/password login, a `JwtEncoder` signs a token containing issuer, subject, expiry and authorities, and future requests carry it as a bearer token. A `JwtDecoder` verifies the signature and validates the claims, a `JwtAuthenticationConverter` maps claims to authorities, and Spring Security reconstructs `Authentication` per request. JWT does not remove authentication, and its Base64URL payload is readable — the signature does not stop editing, it makes unauthorized edits detectable.**
