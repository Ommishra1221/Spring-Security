# Spring Security — Lecture 1 Notes
## Security Foundations, Identity, Authentication, Authorization & Spring Boot Defaults

> **Goal of Lecture 1:** Build the mental model needed to understand Spring Security before learning filters, `UserDetailsService`, authentication providers, JWT, sessions, authorization rules, and advanced security configuration.

---

# 1. The Big Picture

Before learning annotations and configuration, understand what Spring Security is trying to do.

For a normal Spring MVC request:

```text
HTTP Request
    ↓
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

With security:

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
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

The two biggest questions are:

```text
1. WHO is making this request?
   → Authentication

2. WHAT is this identity allowed to do?
   → Authorization
```

This is the foundation of almost everything that follows.

---

# 2. What Is Application Security?

Suppose an application exposes:

```http
DELETE /api/users/57
```

A normal application thinks:

```text
Request → Controller → Service → Repository → DB
```

Security asks additional questions before allowing the operation:

```text
Who sent the request?
Has the identity been verified?
Is the identity allowed to perform DELETE?
Is the identity allowed to delete user 57 specifically?
Has the request been manipulated?
Could it be forged or replayed?
```

### Core definition

> **Application security protects valuable resources and operations from unauthorized access, modification, destruction, disclosure, and misuse.**

Security is therefore much broader than a login page.

---

# 3. What Can Be an Asset?

An **asset** is anything valuable that should be protected.

## Data Assets

```text
Passwords
Email addresses
Phone numbers
Health records
Payment information
Source code
API keys
Business documents
```

## Operational Assets

These are valuable actions:

```text
Create an order
Transfer money
Approve a refund
Change another user's password
Delete an account
Publish content
```

## Infrastructure Assets

```text
Database
Server
Cloud account
Message broker
File storage
Deployment pipeline
```

## Security Assets

```text
Session IDs
JWT signing keys
Encryption keys
OAuth client secrets
Password-reset tokens
CSRF tokens
```

## Availability Is an Asset Too

An attacker does not have to steal data to cause harm.

Examples:

```text
Thousands of expensive requests
Repeated login attempts
Resource exhaustion
Database connection exhaustion
```

So:

> **Availability itself is a security asset.**

---

# 4. Security Is a Collection of Controls

Security can involve:

```text
Authentication
Authorization
Input validation
Password protection
Session protection
CSRF protection
CORS policies
Secure communication
Rate limiting
Audit logging
Error handling
Secret management
```

Spring Security primarily helps with:

```text
Authentication
Authorization
Security-context management
Session security
Common web-security protections
Protocol integration
```

But Spring Security does NOT automatically fix:

```text
SQL injection caused by unsafe queries
Insecure file uploads
Weak business rules
Secrets committed to Git
Vulnerable dependencies
Incorrect database permissions
Sensitive information written into logs
```

### Remember

> **Spring Security is one part of application security, not the entire security strategy.**

---

# 5. Asset, Threat, Vulnerability

These terms must not be mixed up.

### Asset

Something valuable that needs protection.

### Threat

Something that could harm an asset.

### Vulnerability

A weakness that can make an attack possible.

### Attack

An attempt to exploit a weakness.

A useful relationship:

```text
Threat + Vulnerability
        ↓
   Possible Attack
        ↓
      Impact
```

Example:

```text
Threat:
Attacker wants admin access

Vulnerability:
Unlimited password attempts

Attack:
Automated password guessing

Impact:
Admin account compromised
```

---

# 6. Trust Boundary

A **trust boundary** is a point where data moves from one trust level to another.

Example:

```text
Browser
   ↓
Internet
   ↓
Spring Boot Application
   ↓
Database
```

Possible boundaries:

```text
Browser → Server
Server → Database
Service A → Service B
Mobile App → API Gateway
Public Network → Internal Network
Identity Provider → Application
```

Whenever data crosses a trust boundary, ask:

```text
Who produced it?
Can it be modified?
Should it be authenticated?
Should it be authorized?
Should it be validated?
Should it be encrypted?
```

---

# 7. Why the Browser Is Untrusted

This is one of the most important beginner concepts.

Suppose frontend code says:

```javascript
if (user.role === "ADMIN") {
    showDeleteButton();
}
```

This changes the UI.

It does **not** secure the endpoint.

A user controls the browser and can:

```text
Change JavaScript
Modify HTML
Use Postman
Use curl
Change parameters
Call hidden endpoints
Replay requests
```

So even if the button is hidden, the client can manually send:

```http
DELETE /api/users/57
```

### Golden Rule

> **Frontend restrictions improve UX; server-side authorization provides the actual security boundary.**

---

# 8. Authentication

Authentication answers:

> **Who are you, and what evidence proves that claim?**

Example:

```text
Person says:
"I am Aditya."

Evidence:
Employee card

Verification:
Card is checked

Result:
Accepted / Rejected
```

That is authentication.

### Simple definition

> **Authentication is the process of verifying an identity claim.**

---

# 9. Username + Password Authentication

Typical flow:

```text
Username
   ↓
Identity claim

Password
   ↓
Evidence

Verification
   ↓
Compare with stored password representation

Result
   ↓
Success / Failure
```

A password should be represented/stored securely rather than as plain text.

---

# 10. Authentication Factors

Three common categories:

## Something You Know

```text
Password
PIN
Security answer
```

## Something You Have

```text
Smart card
Authenticator application
Private cryptographic key
```

## Something You Are

```text
Fingerprint
Face
Iris
```

---

# 11. Multi-Factor Authentication (MFA)

MFA combines evidence from multiple categories.

Example:

```text
Password
+
Authenticator App OTP
```

This combines:

```text
Something you know
+
Something you have
```

Two credentials from the same category are not necessarily strong multi-factor authentication.

---

# 12. Authentication vs Authorization

### Authentication

```text
WHO are you?
```

### Authorization

```text
WHAT are you allowed to do?
```

Example:

```text
Authentication:
You are Aditya.

Authorization:
Aditya may enter the training room,
but may not enter the finance department.
```

### Interview answer

> **Authentication establishes identity. Authorization decides access for that identity.**

---

# 13. Authorization Is More Than Checking a Role

A real authorization decision can depend on:

```text
Subject
Action
Resource
Context
```

## Subject

```text
User 51
Administrator
Manager
Service account
Anonymous visitor
```

## Action

```text
Read
Create
Update
Delete
Approve
Transfer
Download
```

## Resource

```text
Order 101
Employee record 57
Bank account 9001
Admin dashboard
Course video
```

## Context

```text
Is the user the owner?
Is it within business hours?
Is the account active?
Is amount within approval limit?
Has MFA been completed?
```

### Formula

```text
Can Subject S
perform Action A
on Resource R
under Context C?
```

---

# 14. Public Access Is Still Authorization

Suppose:

```http
GET /api/courses/public
```

is available to everyone.

That is still an authorization policy:

```text
Permit all requests
```

So:

> **"Public" is also an authorization decision.**

---

# 15. Normal Security Flow

A simplified flow:

```text
Request
   ↓
Extract credentials
   ↓
Authenticate identity
   ↓
Determine authorities
   ↓
Evaluate authorization rule
   ↓
Allow / Deny
```

Usually, authentication information must be established before identity-based authorization can be evaluated.

---

# 16. Authentication Failure vs Security Attack

They are not the same.

## Authentication Failure

Examples:

```text
Wrong password
Unknown username
Expired token
Invalid token signature
Malformed Authorization header
Disabled account
Expired account
Expired session
Incorrect OTP
```

This means acceptable identity evidence was not established.

## Security Attack

A deliberate attempt to defeat or misuse security controls.

Example:

```text
password1
password2
password3
...
```

One failed login = authentication failure.

Thousands of deliberate attempts = possible brute-force attack.

---

# 17. Credential Stuffing

Credential stuffing uses username/password combinations stolen from another service.

Important point:

```text
Credentials may be correct
        ↓
Authentication may succeed
        ↓
Request can still be malicious
```

This is why additional controls may be needed:

```text
MFA
Anomaly detection
Login notifications
Device verification
Rate limiting
Credential-breach detection
```

### Important principle

> **Successful authentication proves acceptable credentials were presented; it does not prove the overall request is trustworthy in every sense.**

---

# 18. Do Not Reveal Unnecessary Authentication Details

Avoid exposing:

```text
"Username exists, but password is wrong."
```

Prefer a generic external response such as:

```text
Invalid username or password.
```

Internally, diagnostics may be more precise:

```text
User not found
Password mismatch
Account disabled
Token expired
```

This separates:

```text
External security message
vs.
Internal diagnostic information
```

---

# 19. Spring Security Identity Model

The important vocabulary is:

```text
Identity
Credentials
Principal
Authorities
Roles
Permissions
Authentication
SecurityContext
SecurityContextHolder
```

They are connected, but they are not interchangeable.

---

# 20. Identity

Identity is the entity whose actions the system wants to recognize.

It may represent:

```text
Human user
Administrator
Customer
Another application
Microservice
Scheduled job
IoT device
```

### Identity vs Account

An account is a stored application representation.

Example:

```text
users table

id | username | password_hash | enabled
42 | aditya   | ...           | true
```

The account is one way the application represents/manages the identity.

> **Identity is the conceptual subject; an account is one representation of it.**

---

# 21. Credentials

Credentials are evidence used to prove an identity claim.

Examples:

```text
Password
OTP
Private key
Certificate
API key
Bearer token
```

Mental model:

```text
Identity
→ "Who I claim to be"

Credential
→ "Evidence that I am that identity"
```

---

# 22. Principal

The **principal** is the identity currently represented inside the security system.

Examples:

```text
principal = "aditya"
```

or after successful authentication:

```text
principal = UserDetails("aditya")
```

### Easy distinction

```text
Identity
→ conceptual entity

Principal
→ representation of that identity inside the security system
```

---

# 23. Authorities

An authority represents a capability or security attribute granted to a principal.

Examples:

```text
COURSE_READ
COURSE_WRITE
USER_DELETE
PAYMENT_REFUND
ROLE_ADMIN
SCOPE_orders.read
```

Spring Security represents these using:

```java
GrantedAuthority
```

and stores them in the current `Authentication`.

---

# 24. Never Trust a Client's Claimed Role

A client cannot safely become an admin just by sending:

```json
{
  "username": "aditya",
  "role": "ADMIN"
}
```

The authority must come from trusted security infrastructure, for example:

```text
Trusted database
Verified token
Trusted identity provider
AuthenticationProvider
```

### Golden Rule

> **The client can claim an identity or role, but the server must independently establish trust.**

---

# 25. Role vs Authority

A **role** is usually a broader grouping.

Examples:

```text
USER
INSTRUCTOR
MANAGER
ADMIN
SUPER_ADMIN
```

Specific authorities can represent capabilities:

```text
COURSE_READ
COURSE_CREATE
COURSE_UPDATE
STUDENT_PROGRESS_READ
```

Example:

```text
ROLE_INSTRUCTOR
   ├── COURSE_READ
   ├── COURSE_CREATE
   ├── COURSE_UPDATE
   └── STUDENT_PROGRESS_READ
```

### Mental model

```text
Role
→ broader grouping of privileges

Authority
→ specific capability/security attribute
```

A role is commonly represented as an authority using the `ROLE_` prefix.

---

# 26. `Authentication` Object

`Authentication` is one of the central Spring Security abstractions.

It can represent two different things.

## A. Authentication Request

Before verification:

```text
principal     = "aditya"
credentials   = "raw-password"
authorities   = []
authenticated = false
```

Meaning:

```text
"Someone claims to be Aditya and supplied credentials."
```

This does not mean the identity has been accepted.

## B. Authenticated Principal

After successful verification:

```text
principal     = UserDetails("aditya")
credentials   = null

authorities   = [ROLE_USER, COURSE_READ]
authenticated = true
```

Now the security system has accepted the identity.

---

# 27. Important `Authentication` Methods

Conceptually:

```java
public interface Authentication {

    Object getPrincipal();

    Object getCredentials();

    Collection<? extends GrantedAuthority>
        getAuthorities();

    Object getDetails();

    boolean isAuthenticated();

    String getName();
}
```

Understand them like this:

| Method | Meaning |
|---|---|
| `getPrincipal()` | Current identity |
| `getCredentials()` | Evidence used for authentication |
| `getAuthorities()` | Granted capabilities |
| `getDetails()` | Additional authentication/request details |
| `isAuthenticated()` | Whether authentication has been accepted |
| `getName()` | Name associated with the authentication |

---

# 28. Authentication Transformation

Think about the process as:

```text
Unauthenticated Authentication
            ↓
     AuthenticationManager
            ↓
      Verify credentials
            ↓
Authenticated Authentication
```

The important conceptual change is:

```text
Before:
principal = username
credentials = supplied evidence
authenticated = false

After:
principal = authenticated representation
credentials = commonly cleared
authorities = granted permissions
authenticated = true
```

---

# 29. `SecurityContext`

The `SecurityContext` is a container for the current:

```text
Authentication
```

Conceptually:

```java
SecurityContext context = ...;
Authentication authentication =
    context.getAuthentication();
```

### Mental model

```text
SecurityContext
      ↓
contains Authentication
```

---

# 30. Why Do We Need `SecurityContext`?

Without a shared security context, we would have to pass the current user through every layer:

```java
service.createCourse(courseRequest, currentUser);
```

Instead, Spring Security provides request-associated security state that other components can access where appropriate.

The context can be:

```text
Loaded at request start
Updated after authentication
Stored for future requests when applicable
Cleared after request completion
```

---

# 31. `SecurityContextHolder`

`SecurityContextHolder` provides access to the current security context.

Example:

```java
SecurityContext context =
    SecurityContextHolder.getContext();

Authentication authentication =
    context.getAuthentication();

String username = authentication.getName();

Object principal = authentication.getPrincipal();

Collection<? extends GrantedAuthority> authorities =
    authentication.getAuthorities();
```

### Mental model

```text
SecurityContextHolder
        ↓
SecurityContext
        ↓
Authentication
```

---

# 32. Why Is `SecurityContextHolder` Important?

A typical request can look like:

```text
HTTP Request
      ↓
Security Filters authenticate user
      ↓
Controller
      ↓
Service
      ↓
Method-security check / authorization
      ↓
Audit logic
```

The filter may discover the identity early.

Later components may need to know:

```text
Who is the current user?
Which authorities were granted?
Who performed this action?
```

`SecurityContextHolder` provides access without manually passing the identity through every method.

---

# 33. Default `ThreadLocal` Strategy

In the servlet model, a request is commonly processed by one thread at a time.

Spring Security therefore uses a `ThreadLocal`-based strategy by default for the security context in this model.

Conceptually:

```text
Request 1 → Thread A
               ↓
        SecurityContext
             Aditya

Request 2 → Thread B
               ↓
        SecurityContext
             Rohit
```

Each thread sees its own current context.

---

# 34. Why Must the SecurityContext Be Cleared?

Application servers reuse threads.

Imagine:

```text
Request 1
Thread 7
User = ADMIN
```

Later:

```text
Request 2
Same Thread 7
User = anonymous
```

If the old context stayed in the ThreadLocal:

```text
Request 2
    ↓
may accidentally inherit
Request 1's authentication
```

Therefore:

> **Security context must be established and cleared around request processing.**

---

# 35. Complete Identity Model

Memorize this chain:

```text
Identity
   │
   │ claimed using
   ↓
Credentials
   │
   │ verified by authentication system
   ↓
Authentication
   ├── Principal
   ├── Credentials
   ├── Authorities
   └── Authentication Status
            │
            │ stored inside
            ↓
      SecurityContext
            │
            │ accessed through
            ↓
      SecurityContextHolder
```

This is the single most important architecture diagram in Lecture 1.

---

# 36. HTTP Requests Are Independent

Consider:

```text
POST /login
GET  /profile
GET  /orders
POST /logout
```

HTTP requests do not automatically remember a previous authentication.

The application therefore needs a strategy for carrying authentication across requests.

Two broad models:

```text
Stateful Authentication
Stateless Authentication
```

---

# 37. Stateful Authentication

In a stateful design, the server retains authentication-related state between requests.

A common implementation uses an HTTP session.

Flow:

```text
User logs in
    ↓
Credentials verified
    ↓
Authentication created
    ↓
SecurityContext created
    ↓
Authentication/security state saved in session
    ↓
Session identifier sent to client
    ↓
Client sends session identifier later
    ↓
Server loads session
    ↓
Authentication restored
```

---

# 38. Stateful Login Example

Initial request:

```http
POST /login
Content-Type: application/x-www-form-urlencoded

username=aditya&password=secret
```

After successful authentication, conceptually:

```text
Session ID = ABC123

Session:
    SecurityContext
    Authentication
    principal = aditya
    authorities = [ROLE_USER]
```

The response may contain:

```http
Set-Cookie: JSESSIONID=ABC123
```

---

# 39. Stateful Subsequent Request

Later:

```http
GET /profile
Cookie: JSESSIONID=ABC123
```

Server can follow:

```text
ABC123
  ↓
Session
  ↓
SecurityContext
  ↓
Authentication
```

The original password does not need to be resent.

---

# 40. Cookie Does Not Normally Contain the Full User Session

A common misconception is:

> "The browser cookie contains the logged-in user."

In ordinary server-side session authentication, the cookie generally contains a:

```text
Session identifier
```

The important authentication state lives on the server/session store.

Example:

```text
Browser:
JSESSIONID=ABC123

Server:
ABC123 → Session → SecurityContext
```

---

# 41. Where Can Session State Live?

Session state may live in:

```text
Application memory
Distributed cache
Redis
Database
External session store
```

---

# 42. Advantages of Stateful Authentication

```text
Authenticate once
Server-side invalidation
Server controls session lifetime
Can track session activity
Can manage concurrent sessions
```

If the server invalidates:

```text
Session ABC123
```

future requests using that session can fail.

---

# 43. Costs of Stateful Authentication

### Server storage

Every active session consumes resources.

### Distributed-system complexity

With multiple servers, you need a strategy such as:

```text
Shared session storage
Sticky sessions
Another consistent session strategy
```

---

# 44. Cookie ≠ Stateful Authentication

Cookies can carry many things:

```text
Session ID
Language preference
Theme preference
Analytics identifier
CSRF token
Authentication token
```

A stateless application can put an access token in a cookie.

Therefore:

> **Cookie does not automatically mean stateful authentication.**

---

# 45. Session ID as a Bearer Credential

If an attacker obtains:

```text
JSESSIONID=ABC123
```

they may not know the password, but possession of the session identifier can still grant access to the authenticated session.

Therefore the session ID must be protected like a credential.

---

# 46. Session Security Protections

Important protections include:

```text
HTTPS
Secure cookie attribute
HttpOnly
SameSite policy
Session expiration
Session invalidation
Session ID rotation
Session fixation protection
```

---

# 47. Session Fixation

Conceptual attack:

```text
Attacker knows session ID = XYZ
        ↓
Victim logs in using XYZ
        ↓
XYZ becomes authenticated
        ↓
Attacker reuses XYZ
```

Mitigation idea:

```text
Before authentication → XYZ
After authentication  → NEW-789
```

Changing the session identifier after authentication helps prevent the attacker from reusing a pre-authentication identifier.

---

# 48. Stateless Authentication

In stateless authentication, the server does not rely on a previously stored HTTP login session to reconstruct authentication for the next request.

Each request carries a credential.

Example:

```http
Authorization: Bearer TOKEN
```

---

# 49. Stateless Request Flow

```text
Request
  ↓
Extract credential
  ↓
Validate credential
  ↓
Create Authentication
  ↓
Place Authentication in SecurityContext
  ↓
Authorize request
  ↓
Controller
  ↓
Clear request context
```

Next request:

```text
Start authentication again
using the credential carried by that request
```

---

# 50. Stateless Does Not Mean "The Server Stores Nothing"

A stateless application can still store:

```text
Users
Passwords
Roles
Permissions
Signing keys
Refresh tokens
Revoked-token records
Audit logs
Rate-limit counters
Business data
```

### Correct definition

> **Stateless authentication means the server does not depend on a previously stored HTTP login session to reconstruct authentication for the next request.**

---

# 51. Self-Contained vs Opaque Token

## Self-Contained Token

Contains signed information that can often be validated locally.

JWT is a common example.

Example claims:

```json
{
  "sub": "aditya",
  "scope": "course.read course.write",
  "iat": 1785980000,
  "exp": 1785983600
}
```

## Opaque Token

May look like:

```text
a8f2b9137c45d...
```

The token itself does not reveal meaningful application information.

The application may ask a token service:

```text
Is this token active?
Which user does it represent?
What scopes does it contain?
```

---

# 52. JWT

JWT stands for:

```text
JSON Web Token
```

### Critical point

> **JWT is a token format, not a synonym for stateless authentication.**

---

# 53. Bearer Token

A bearer token effectively means:

> **Whoever possesses the token can present it.**

Think of it like cash:

```text
Possession
   ↓
Ability to present
```

The server generally does not require the current holder to prove they were the original recipient.

---

# 54. Bearer vs Opaque

These describe different dimensions:

```text
Bearer
→ how possession of the credential is used

Opaque
→ how the token is represented
```

Therefore this can be both:

```text
Bearer token
+
Opaque token
```

Example:

```http
Authorization: Bearer a8f2b9137c45d...
```

---

# 55. Protect Bearer Tokens

Bearer credentials must be protected from:

```text
Logging
URL leakage
Browser storage attacks
Network interception
Accidental sharing
Source-code commits
```

A JWT is not automatically safe simply because it is a JWT.

Security also depends on:

```text
Strong signing keys
Correct signature validation
Expiry validation
Issuer validation
Audience validation
Secure transport
Safe client storage
Appropriate scopes
```

---

# 56. JWT Can Be Used With Sessions

### Stateless JWT design

```text
JWT sent every request
       ↓
JWT validated every request
       ↓
No HTTP-session authentication used
```

### Session-backed design

```text
JWT used during authentication
       ↓
Authentication stored in HTTP session
       ↓
Later requests use session cookie
```

Therefore:

> **Token format does not determine session policy.**

---

# 57. Authentication Mechanism vs Session Policy

This distinction is extremely important.

## Authentication Mechanism

Answers:

> **How are credentials obtained and verified?**

Examples:

```text
Form login
HTTP Basic
Bearer token
```

## Session Policy

Answers:

> **After authentication succeeds, should the resulting SecurityContext be persisted in an HTTP session?**

Therefore:

```text
Authentication mechanism
        ≠
Session policy
```

---

# 58. Form Login

Example:

```http
POST /login

username=aditya&password=secret
```

The application may then:

```text
Authenticate
   ↓
Create SecurityContext
   ↓
Persist it in HTTP session
   ↓
Send session cookie
```

This is a traditional stateful design.

---

# 59. HTTP Basic

HTTP Basic transports credentials in the Authorization header:

```http
Authorization: Basic <credentials>
```

The client may send these credentials repeatedly.

Important distinction:

```text
HTTP Basic
→ credential transport mechanism

STATELESS
→ session persistence policy
```

They are not the same concept.

---

# 60. Stateful Request Lifecycle

Initial authentication:

```text
POST /login
   ↓
Username/password extracted
   ↓
Credentials authenticated
   ↓
Authentication created
   ↓
SecurityContext created
   ↓
SecurityContext saved in HttpSession
   ↓
Session cookie returned
```

Later:

```text
GET /profile
   ↓
Cookie: JSESSIONID=ABC123
   ↓
Session loaded
   ↓
SecurityContext loaded
   ↓
Authentication placed in SecurityContextHolder
   ↓
Authorization
   ↓
Controller
   ↓
SecurityContextHolder cleared
```

---

# 61. Stateless Request Lifecycle

```text
GET /profile
Authorization: Bearer TOKEN
   ↓
Token extracted
   ↓
Token authenticated
   ↓
Authentication created
   ↓
SecurityContext created
   ↓
Authentication placed in SecurityContextHolder
   ↓
Authorization
   ↓
Controller
   ↓
SecurityContextHolder cleared
   ↓
No authentication saved in HTTP session
```

---

# 62. Spring Boot Security Auto-Configuration

Now connect the security model to Spring Boot.

Spring Security provides the security framework.

Spring Boot observes what is available and supplies sensible defaults.

Simplified flow:

```text
Security dependency added
        ↓
Security classes found on classpath
        ↓
Security auto-configuration activates
        ↓
Security beans created
        ↓
Security Filter Chain registered
        ↓
Requests pass through security processing
```

---

# 63. Two Different Default Problems

Spring Boot conceptually solves two separate problems.

## Problem 1

```text
Which requests are protected?
Which security mechanisms are available?
```

→ `SecurityFilterChain`

## Problem 2

```text
Which user can authenticate?
Where is that user loaded from?
```

→ `UserDetailsService`

### Important mental model

```text
SecurityFilterChain
→ How requests are secured

UserDetailsService
→ How user information is loaded for authentication
```

---

# 64. Default Security Configuration

A simplified representation is:

```java
@Bean
SecurityFilterChain securityFilterChain(
        HttpSecurity http) throws Exception {

    http
        .authorizeHttpRequests(authorize ->
            authorize
                .anyRequest()
                .authenticated()
        )
        .formLogin(Customizer.withDefaults())
        .httpBasic(Customizer.withDefaults());

    return http.build();
}
```

Meaning:

```text
Every request requires authentication
+
Form login available
+
HTTP Basic available
```

---

# 65. `anyRequest().authenticated()`

This rule means:

> **Every matching request requires an authenticated identity.**

It is asking:

```text
"Is the requester authenticated?"
```

It is not yet asking:

```text
"Does the requester have ROLE_ADMIN?"
```

That comes under authorization/authority checks.

---

# 66. `formLogin()`

```java
.formLogin(Customizer.withDefaults())
```

Enables the standard username/password form-login mechanism with default behavior.

---

# 67. `httpBasic()`

```java
.httpBasic(Customizer.withDefaults())
```

Enables HTTP Basic authentication.

Conceptually:

```http
Authorization: Basic <encoded-credentials>
```

---

# 68. Conditional Auto-Configuration

Spring Boot follows this important rule:

> **Provide defaults when the developer has not supplied their own configuration.**

Conceptually:

```text
SecurityFilterChain bean exists?
        │
   ┌────┴────┐
  NO        YES
   │          │
   ↓          ↓
Default     Back off
security    and use
created     custom config
```

---

# 69. `SecurityFilterChain` vs `UserDetailsService`

### `SecurityFilterChain`

Controls the request-security rules and filter processing.

### `UserDetailsService`

Loads user information used by the authentication system.

Other authentication components can also participate, such as:

```text
AuthenticationProvider
AuthenticationManager
```

### Important nuance

Creating a custom `SecurityFilterChain` changes the default web-security configuration.

It does **not automatically mean the generated default user configuration disappears**.

To replace the generated user/authentication setup, provide an appropriate user/authentication component.

---

# 70. Default User

Spring Boot's default security setup provides a development-time in-memory user.

Conceptually:

```text
InMemoryUserDetailsManager
      ↓
UserDetails
      ├── username = "user"
      ├── password = generated password
      ├── authorities = ROLE_USER
      └── enabled = true
```

The default username is:

```text
user
```

---

# 71. Where Is the Default User Stored?

It is not:

```text
saved in MySQL
saved through JPA
created by your controller
part of a registration system
intended as a permanent production account
```

It exists to give the default development configuration an identity that can authenticate.

---

# 72. Default Login Transformation

Simplified:

```text
Login request
   ↓
Load UserDetails
   ↓
Verify password
   ↓
Authentication success
   ↓
Authenticated Authentication
   ↓
SecurityContext
```

Conceptually the resulting object contains:

```text
principal = UserDetails("user")
credentials = cleared
roles/authorities = ROLE_USER
authenticated = true
```

---

# 73. Default Role

The default user normally has:

```text
ROLE_USER
```

Remember:

```text
Role name:
USER

Authority representation:
ROLE_USER
```

But for:

```java
anyRequest().authenticated()
```

the rule only asks:

```text
Is the user authenticated?
```

It is not requiring `ROLE_ADMIN`.

---

# 74. Configuring the Default User

For learning/testing, properties can change the generated credentials:

```properties
spring.security.user.name=aditya
spring.security.user.password=secret123
spring.security.user.roles=USER,ADMIN
```

Conceptually:

```text
username = aditya
password = secret123
authorities = ROLE_USER, ROLE_ADMIN
```

---

# 75. Generated Password

Spring Boot can generate a random development password and log it during application startup.

Important:

> **The generated password is intended for development/testing, not as a production authentication strategy.**

---

# 76. The CSRF `403 Forbidden` Surprise

A common beginner experience is:

```text
Login succeeds
    ↓
POST /api/courses
    ↓
403 Forbidden
```

The first thought may be:

```text
"The user does not have permission."
```

But another reason may be CSRF protection.

Spring Security enables CSRF protection by default for servlet web applications.

A state-changing request such as:

```text
POST
PUT
PATCH
DELETE
```

may be rejected when the expected CSRF token is missing.

---

# 77. `401` vs `403`

## `401 Unauthorized`

Typically relates to:

```text
Authentication missing
or
Authentication failed
```

## `403 Forbidden`

The request was understood but rejected.

A `403` on a state-changing request may be caused by:

```text
Missing CSRF token
```

and not only by an insufficient role/authority.

### Interview memory trick

```text
401 → "Who are you?" problem
403 → "Request understood, but access/rejection" problem
```

---

# 78. End-to-End Security Flow

Now connect the entire lecture:

```text
                   CLIENT
                     │
                     ↓
                HTTP REQUEST
                     │
                     ↓
              TRUST BOUNDARY
                     │
                     ↓
        SPRING SECURITY FILTER CHAIN
                     │
                     ↓
              AUTHENTICATION
                     │
                     ↓
           Authentication object
                     │
                     ↓
              SecurityContext
                     │
                     ↓
          SecurityContextHolder
                     │
                     ↓
              AUTHORIZATION
                     │
                     ↓
                 CONTROLLER
                     │
                     ↓
                  SERVICE
                     │
                     ↓
                REPOSITORY
                     │
                     ↓
                 DATABASE
```

---

# 79. Stateful End-to-End Flow

```text
POST /login
    ↓
Authenticate
    ↓
Authentication
    ↓
SecurityContext
    ↓
HTTP Session
    ↓
JSESSIONID

Later request
    ↓
JSESSIONID
    ↓
Load Session
    ↓
Restore SecurityContext
    ↓
Authentication
    ↓
Authorization
    ↓
Controller
```

---

# 80. Stateless End-to-End Flow

```text
Request
    ↓
Bearer TOKEN
    ↓
Token validation
    ↓
Authentication
    ↓
SecurityContext
    ↓
Authorization
    ↓
Controller
    ↓
Clear SecurityContext
```

Next request repeats authentication using that request's credential.

---

# 81. Most Important Comparisons

## Authentication vs Authorization

```text
Authentication → Who are you?
Authorization  → What are you allowed to do?
```

## Identity vs Principal

```text
Identity
→ conceptual entity

Principal
→ identity represented in the security system
```

## Credential vs Authentication

```text
Credential
→ evidence

Authentication
→ authentication attempt/result + identity + authorities
```

## Authentication vs Authentication Object

```text
Authentication
→ the process/concept

Authentication object
→ Spring Security representation of the attempt or authenticated principal
```

## SecurityContext vs SecurityContextHolder

```text
SecurityContext
→ container holding Authentication

SecurityContextHolder
→ access point to the current SecurityContext
```

## Role vs Authority

```text
Role
→ broader privilege grouping

Authority
→ specific capability/security attribute
```

## Stateful vs Stateless

```text
Stateful
→ authentication carried across requests using server-side HTTP session state

Stateless
→ each request carries the credential used to reconstruct authentication
```

## JWT vs Stateless

```text
JWT
→ token format

Stateless
→ authentication/session design
```

## Cookie vs Session

```text
Cookie
→ browser transport/storage mechanism

Session
→ server-side state
```

## Authentication Mechanism vs Session Policy

```text
Mechanism
→ how credentials are verified

Session Policy
→ whether authentication is persisted in HTTP session
```

## 401 vs 403

```text
401 → authentication missing/failed
403 → request rejected
```

---

# 82. 🎤 Interview Questions

## Q1. What is Spring Security?

Spring Security is a framework used to secure Spring applications, primarily providing authentication, authorization, security-context management, session security, common web-security protections, and integration with security protocols.

## Q2. What is authentication?

Authentication is the process of verifying an identity claim.

## Q3. What is authorization?

Authorization is the decision about whether an identity can perform an action on a resource under the applicable rules and context.

## Q4. Authentication vs authorization?

```text
Authentication → identity
Authorization  → access
```

## Q5. What is a principal?

The identity currently represented inside the security system.

## Q6. What are credentials?

Evidence used to prove an identity claim.

## Q7. What are authorities?

Granted capabilities/security attributes associated with a principal.

## Q8. What is a role?

A broader grouping of privileges, commonly represented with a `ROLE_` prefix in Spring Security authority conventions.

## Q9. What is `Authentication`?

It can represent an authentication attempt and, after trusted verification, the authenticated principal, authorities, details, and authentication state.

## Q10. What is `SecurityContext`?

A container holding the current `Authentication`.

## Q11. What is `SecurityContextHolder`?

A central access point for the current `SecurityContext`.

## Q12. Why is ThreadLocal involved?

In the default servlet model, security context is associated with the current request-processing thread, allowing application components on that thread to access the current security context.

## Q13. Why clear the SecurityContext?

Because server threads are reused; clearing prevents one request's authentication from leaking into another request.

## Q14. What is stateful authentication?

Authentication that relies on server-side HTTP session state being persisted between requests.

## Q15. What is stateless authentication?

Authentication where the next request carries the credential needed to establish the authentication instead of relying on a previously stored HTTP login session.

## Q16. Does JWT automatically mean stateless?

No. JWT is a token format. Session policy is a separate design decision.

## Q17. Does a cookie automatically mean stateful authentication?

No. A cookie can carry many kinds of data, including an access token.

## Q18. What is a bearer token?

A credential that can generally be presented by whoever possesses it.

## Q19. What is an opaque token?

A token whose contents are not meaningful to the consuming application and may be checked through a token service/introspection mechanism.

## Q20. What is `SecurityFilterChain`?

A configuration describing how incoming web requests are processed and protected by Spring Security filters and authorization rules.

## Q21. What does `UserDetailsService` do?

It loads user-specific data used by the authentication process.

## Q22. What happens when you define your own `SecurityFilterChain`?

Spring Boot backs off from its default web-security configuration, while user/authentication configuration is a separate concern.

## Q23. What is the default Spring Boot user?

Conceptually:

```text
username = user
password = generated
authority = ROLE_USER
```

for the default development configuration.

## Q24. Why can a logged-in POST return 403?

Because authentication success does not disable CSRF protection; a missing CSRF token can cause rejection of a state-changing request.

---

# 83. Common Beginner Mistakes

### Mistake 1 — Authentication = Authorization

Wrong.

```text
Authentication → identity
Authorization → access
```

### Mistake 2 — Hiding an admin button is security

Wrong.

A malicious client can call the endpoint directly.

### Mistake 3 — JWT means stateless

Wrong.

JWT is a format; session policy is separate.

### Mistake 4 — Cookie means session

Wrong.

A cookie can carry a session ID, token, CSRF token, or unrelated data.

### Mistake 5 — Stateless means no server-side data

Wrong.

It means authentication does not depend on a previously stored HTTP login session.

### Mistake 6 — Authenticated user can do everything

Wrong.

Authorization still determines access.

### Mistake 7 — Client can send `role=ADMIN`

Wrong.

Authorities must come from trusted security infrastructure.

### Mistake 8 — `SecurityContextHolder` permanently stores the logged-in user

Wrong.

The normal servlet model establishes and clears request security context.

### Mistake 9 — 403 always means wrong role

Wrong.

Missing CSRF protection can also cause 403 for state-changing requests.

### Mistake 10 — `SecurityFilterChain` and `UserDetailsService` do the same job

Wrong.

```text
SecurityFilterChain → request security
UserDetailsService  → user loading for authentication
```

---

# 84. ⭐ If You Remember Only 20 Things

```text
1. Spring Security is not just a login system.

2. Security protects data, operations, infrastructure, security material,
   identities, and availability.

3. Threat = potential danger.

4. Vulnerability = weakness that can enable an attack.

5. Browser/client is untrusted.

6. Frontend authorization is not a security boundary.

7. Authentication = WHO are you?

8. Authorization = WHAT can you do?

9. Authorization can depend on subject + action + resource + context.

10. Credential = evidence used to prove identity.

11. Principal = identity represented in the security system.

12. Authorities = granted capabilities/security attributes.

13. Authentication contains the current security identity and authorities.

14. SecurityContext contains Authentication.

15. SecurityContextHolder provides access to the current SecurityContext.

16. Default servlet security context uses a ThreadLocal-based strategy.

17. Context must be cleared because server threads are reused.

18. Stateful = server-side HTTP session persists authentication state.

19. Stateless = each request carries the credential needed to establish authentication.

20. JWT is a token format; it does not automatically determine statelessness.
```

---

# 85. 🔥 One-Minute Revision

> **Spring Security is a security framework that helps protect Spring applications from unauthorized access. The foundation begins with assets, threats, vulnerabilities, and trust boundaries. Authentication answers "Who are you?", while authorization answers "What are you allowed to do?". Spring Security represents authentication through an `Authentication` object containing information such as the principal, credentials, authorities, details, and authentication state. The current `Authentication` is held inside a `SecurityContext`, which is accessed through `SecurityContextHolder`. In the default servlet model, the security context is associated with the current request-processing thread using a ThreadLocal-based strategy and must be cleared when request processing finishes. Across HTTP requests, authentication can be stateful using a server-side HTTP session or stateless using a credential carried on each request. JWT is a token format, not automatically a stateless design, and cookies do not automatically imply sessions. Spring Boot provides default security through auto-configuration, including a default `SecurityFilterChain` and development-time in-memory user. A custom `SecurityFilterChain` controls web-security rules, while `UserDetailsService` is concerned with loading user information for authentication. Finally, a successful login does not guarantee that every state-changing request succeeds because CSRF protection can still reject a request with `403 Forbidden`.**

---

# 86. 📌 Final Concept Map

```text
                     SPRING SECURITY
                            │
                            ↓
                  APPLICATION SECURITY
                            │
             ┌──────────────┼──────────────┐
             │              │              │
           Assets         Threats      Trust Boundaries
             │              │              │
             └──────────────┼──────────────┘
                            ↓
                     AUTHENTICATION
                            │
                     "Who are you?"
                            ↓
                        Credentials
                            ↓
                      Authentication
                  ┌─────────┼─────────┐
                  │         │         │
              Principal Authorities Status
                            │
                            ↓
                     SecurityContext
                            │
                            ↓
                  SecurityContextHolder
                            │
                            ↓
                     AUTHORIZATION
                            │
             Subject + Action + Resource + Context
                            │
                            ↓
                   Security Filter Chain
                            │
                ┌───────────┴───────────┐
                │                       │
            STATEFUL                 STATELESS
                │                       │
          HTTP Session             Request Credential
                │                       │
           JSESSIONID              Bearer Token
                                        │
                                  JWT / Opaque
                │                       │
                └───────────┬───────────┘
                            ↓
                       Controller
                            ↓
                         Service
                            ↓
                       Repository
                            ↓
                        Database
```

---

# 87. 🎯 Final Mental Model

Whenever you see a protected request, ask these questions in order:

```text
1. Where did this request come from?
        ↓
2. What crosses the trust boundary?
        ↓
3. Who is the requester?
        ↓
4. What credential proves that identity?
        ↓
5. Has authentication succeeded?
        ↓
6. What principal and authorities were granted?
        ↓
7. Where is the Authentication stored?
        ↓
8. Is this identity authorized for this action/resource?
        ↓
9. Are additional protections such as CSRF relevant?
        ↓
10. Only then → application business logic
```

---

# 88. What Lecture 1 Prepares You For

The next topics can now be understood as implementations of the concepts above:

```text
SecurityFilterChain
       ↓
Security Filters
       ↓
AuthenticationManager
       ↓
AuthenticationProvider
       ↓
UserDetailsService
       ↓
PasswordEncoder
       ↓
SecurityContext
       ↓
Session Management / JWT
       ↓
Authorization
```

The goal of Lecture 1 is therefore not to memorize every API. It is to make this architecture feel natural before learning how Spring Security implements each part.

---

# Lecture 1 Core Principle

> **Spring Security ultimately exists to make safe decisions around requests: establish who the requester is, preserve that identity in the security context, determine what the identity is allowed to do, and reject requests that do not satisfy the application's security requirements.**
