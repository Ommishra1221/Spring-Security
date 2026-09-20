# Spring Security — Lecture 3 Notes
# Spring Security Database Authentication — Internals & Complete Login Flow

> **Main goal of this lecture:** Understand what actually happens between an incoming HTTP request and the moment Spring Security produces an authenticated user.

The previous lecture taught us:

```text
User
↓
UserDetailsService
↓
UserDetails
↓
PasswordEncoder
↓
Database authentication
```

This lecture goes **one level deeper**:

```text
HTTP Request
   ↓
Servlet Container / Tomcat
   ↓
DelegatingFilterProxy
   ↓
FilterChainProxy
   ↓
SecurityFilterChain
   ↓
Authentication Filter
   ↓
Authentication Token
   ↓
AuthenticationManager
   ↓
ProviderManager
   ↓
DaoAuthenticationProvider
   ↓
UserDetailsService
   ↓
UserRepository
   ↓
Database
   ↓
UserDetails
   ↓
PasswordEncoder
   ↓
Authenticated Authentication
   ↓
SecurityContext
   ↓
SecurityContextHolder
   ↓
Authorization
   ↓
Controller
```

This is the **single most important mental model of Lecture 3**.

---

# 1. Why Learn the Internals?

At the beginning, Spring Security can feel like magic.

You write:

```java
http
    .authorizeHttpRequests(auth ->
        auth.anyRequest().authenticated()
    )
    .formLogin(Customizer.withDefaults());
```

and suddenly:

```text
Login works
Authentication works
Protected endpoints work
Roles can be checked
```

But how?

This lecture removes that mystery.

We want to understand:

```text
Who receives the request?
Who decides which security filters run?
Who extracts username/password?
Who creates the Authentication object?
Who checks the database?
Who checks the password?
Who creates the authenticated result?
Where is the authenticated user stored?
How does authorization happen afterward?
```

---

# 2. The Most Important Rule

Spring Security authentication happens **before the request reaches your controller**.

Basic request path:

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

This ordering is necessary because the application should establish security information before allowing business logic to execute.

For example:

```http
GET /admin/users
```

must answer:

```text
Who is this?
        ↓
Is authentication valid?
        ↓
What authorities do they have?
        ↓
Are they allowed to access /admin/users?
        ↓
Only then → Controller
```

---

# 3. Request Flow — Big Picture

Keep this diagram in your mind:

```text
Client
  ↓
Tomcat
  ↓
Spring Security Filter Chain
  ↓
Authentication
  ↓
Authorization
  ↓
DispatcherServlet
  ↓
Controller
```

A more complete conceptual sequence is:

```text
Protection
    ↓
Authentication
    ↓
Exception Handling
    ↓
Authorization
    ↓
Application
```

The real chain can contain more filters and the exact filter list changes according to configuration.

### Core principle

> **Authentication must happen before authorization because Spring Security cannot decide whether a user has a particular authority until it knows who the user is.**

---

# 4. Servlet Filters

Before Spring Security, remember what a servlet filter does.

A servlet filter sits between:

```text
Client
  ↓
Web application
```

and can inspect, modify, or reject requests before the request reaches the servlet.

Conceptually:

```text
Request
  ↓
Filter 1
  ↓
Filter 2
  ↓
Filter 3
  ↓
Servlet
```

Spring Security builds its security architecture around this filter mechanism.

---

# 5. DelegatingFilterProxy

This is the first internal class you should understand.

```text
DelegatingFilterProxy
```

is a real servlet filter known by the servlet container.

But it does not contain all of Spring Security's logic itself.

Its job is to **delegate** the request to a Spring-managed security filter bean.

Conceptually:

```java
public void doFilter(
        ServletRequest request,
        ServletResponse response,
        FilterChain chain) {

    Filter springFilter =
        findFilterBeanInSpring();

    springFilter.doFilter(
        request,
        response,
        chain
    );
}
```

### Mental model

```text
Tomcat
  ↓
DelegatingFilterProxy
  ↓
Spring-managed security filter
```

---

# 6. Why Is DelegatingFilterProxy Needed?

There are two worlds:

```text
Servlet Container
    ↓
Tomcat / Servlet Filter lifecycle
```

and:

```text
Spring ApplicationContext
    ↓
Spring-managed beans
```

`DelegatingFilterProxy` acts as the bridge.

Think:

```text
Tomcat says:
"I have a request."

DelegatingFilterProxy says:
"I know which Spring bean should handle security."

Spring Security says:
"I'll process the request."
```

---

# 7. FilterChainProxy

The next important component is:

```text
FilterChainProxy
```

It is the central Spring Security filter that stands in front of one or more:

```text
SecurityFilterChain
```

Conceptually:

```text
Tomcat
   ↓
DelegatingFilterProxy
   ↓
FilterChainProxy
```

---

# 8. Why Does FilterChainProxy Exist?

An application may need **different security rules for different URLs**.

Example:

```text
/api/**
/admin/**
/**
```

You may want:

```text
API requests
→ token-based authentication

Admin requests
→ stricter rules

Other requests
→ normal form login
```

So FilterChainProxy can choose a suitable `SecurityFilterChain`.

---

# 9. SecurityFilterChain

A `SecurityFilterChain` is basically:

```text
Request matcher
+
Ordered security filters
```

Conceptually:

```java
public interface SecurityFilterChain {

    boolean matches(
        HttpServletRequest request
    );

    List<Filter> getFilters();
}
```

The exact implementation is framework-internal, but this mental model is extremely useful.

---

# 10. A SecurityFilterChain Example

Suppose:

```text
SecurityFilterChain 1
Matcher:
    /api/**

Filters:
    SecurityContextHolderFilter
    BearerTokenAuthenticationFilter
    ExceptionTranslationFilter
    AuthorizationFilter
```

Another chain:

```text
SecurityFilterChain 2
Matcher:
    /**

Filters:
    SecurityContextHolderFilter
    CsrfFilter
    UsernamePasswordAuthenticationFilter
    BasicAuthenticationFilter
    AnonymousAuthenticationFilter
    ExceptionTranslationFilter
    AuthorizationFilter
```

The exact filter list depends on which security features you configure.

---

# 11. RequestMatcher

A:

```text
RequestMatcher
```

decides whether a particular `SecurityFilterChain` applies to the current request.

It can evaluate things such as:

```text
Request path
HTTP method
Headers
Servlet path
Parameters
Dispatcher type
```

Example:

```http
GET /api/users
```

Matcher:

```text
/api/**
```

returns:

```text
true
```

while:

```text
/admin/**
```

returns:

```text
false
```

---

# 12. First Matching SecurityFilterChain Wins

This is extremely important.

Suppose you configure:

```text
Chain 1 → /api/**
Chain 2 → /**
```

Request:

```http
GET /api/users
```

Both patterns match.

What happens?

```text
Chain 1
   ↓
matches
   ↓
SELECT Chain 1
   ↓
STOP SEARCHING
```

The request does **not** receive:

```text
Chain 1 filters
+
Chain 2 filters
```

Only the selected chain is used.

### Golden rule

> **First matching SecurityFilterChain wins.**

---

# 13. Therefore: Order Specific → General

A safe mental ordering is:

```text
1. /api/**
2. /admin/**
3. /**
```

Why?

Because:

```text
/** 
```

matches almost everything.

If it appears first, it can capture requests that you intended another chain to handle.

---

# 14. What If No SecurityFilterChain Matches?

If no configured chain matches a request:

```text
Spring Security does not apply
a matching SecurityFilterChain
```

Therefore a fallback chain is useful.

Example:

```java
@Bean
SecurityFilterChain fallback(
        HttpSecurity http)
        throws Exception {

    http.authorizeHttpRequests(auth ->
        auth.anyRequest().authenticated()
    );

    return http.build();
}
```

If you do not configure a narrower `securityMatcher`, a chain normally applies to all requests.

---

# 15. HttpSecurity

Now we reach a class you will use constantly:

```java
HttpSecurity
```

Think of `HttpSecurity` as:

> **A builder used to describe how a SecurityFilterChain should behave.**

Example:

```java
http
    .csrf(Customizer.withDefaults())
    .formLogin(Customizer.withDefaults())
    .httpBasic(Customizer.withDefaults())
    .authorizeHttpRequests(auth ->
        auth.anyRequest().authenticated()
    );
```

Each configuration method contributes something to the final security chain.

---

# 16. What HttpSecurity Controls

Conceptually, `HttpSecurity` describes:

```text
Which security features are enabled
Which filters are required
How those filters are configured
How the filters are ordered
Which AuthenticationProviders are used
Which authorization rules apply
```

Finally:

```java
return http.build();
```

turns the builder configuration into a concrete:

```text
SecurityFilterChain
```

---

# 17. HttpSecurity Mental Model

Think:

```text
HttpSecurity
      ↓
"How should security work?"
      ↓
http.csrf(...)
http.formLogin(...)
http.httpBasic(...)
http.authorizeHttpRequests(...)
      ↓
http.build()
      ↓
SecurityFilterChain
```

So:

```text
HttpSecurity
→ configuration builder

SecurityFilterChain
→ built security pipeline
```

---

# 18. Authentication Filters

Different authentication mechanisms receive credentials differently.

Two important examples:

```text
Form Login
HTTP Basic
```

The filter changes because the credential transport changes.

---

# 19. Form Login

Typical login request:

```http
POST /login
Content-Type: application/x-www-form-urlencoded

username=aditya&password=secret123
```

Spring Security's:

```text
UsernamePasswordAuthenticationFilter
```

handles this form-login authentication request.

---

# 20. Important: Authentication Filter Does Not Run Authentication on Every Request

This is a very common misunderstanding.

Suppose the chain contains:

```text
UsernamePasswordAuthenticationFilter
```

That does **not** mean it tries to perform username/password login for:

```http
GET /courses
```

It looks for the request pattern it understands.

For example:

```http
POST /login
```

can be an authentication attempt.

For:

```http
GET /courses
```

the filter normally does not treat it as a new form-login request.

It continues:

```java
chain.doFilter(request, response);
```

### Golden rule

> **Authentication filter exists in the chain ≠ authentication attempt happens on every request.**

---

# 21. HTTP Basic

HTTP Basic sends credentials in the:

```http
Authorization
```

header.

Example:

```http
Authorization: Basic YWRpdHlhOnNlY3JldDEyMw==
```

The relevant filter is:

```text
BasicAuthenticationFilter
```

So:

```text
Form Login
→ UsernamePasswordAuthenticationFilter

HTTP Basic
→ BasicAuthenticationFilter
```

---

# 22. Same Authentication Backend

This is a very important concept.

Form login and HTTP Basic use different filters:

```text
Form Login
      ↓
UsernamePasswordAuthenticationFilter

HTTP Basic
      ↓
BasicAuthenticationFilter
```

But both can eventually use:

```text
AuthenticationManager
      ↓
ProviderManager
      ↓
DaoAuthenticationProvider
      ↓
UserDetailsService
      ↓
Database
```

So:

> **Credential transport can differ while the actual database-authentication engine remains the same.**

---

# 23. From HTTP Request to Authentication Token

Initially, the request contains plain values:

```text
username = aditya
password = secret123
```

Spring Security converts the credentials into a standard authentication object.

For username/password authentication this is commonly:

```text
UsernamePasswordAuthenticationToken
```

---

# 24. Unauthenticated UsernamePasswordAuthenticationToken

Before verification, conceptually:

```text
UsernamePasswordAuthenticationToken

principal     = "aditya"
credentials   = "secret123"
authorities   = []
authenticated = false
```

This is **not yet an authenticated user**.

It means:

> "Someone claims to be Aditya and has supplied this credential."

---

# 25. Why Is the Token Unauthenticated?

Because the request is still only a claim.

Think:

```text
Client says:
"I am Aditya."

Spring Security says:
"Okay, I'll verify it."
```

Only after successful verification can Spring Security produce a trusted authentication result.

---

# 26. Creating an Unauthenticated Token

Conceptually:

```java
UsernamePasswordAuthenticationToken token =
    UsernamePasswordAuthenticationToken
        .unauthenticated(
            username,
            password
        );
```

This creates the **authentication request**.

---

# 27. What Should the Authentication Filter Do?

The filter should **not** contain:

```text
Database logic
Password verification logic
Role lookup logic
```

Its main responsibilities are:

```text
1. Extract credentials
2. Create Authentication request
3. Send it to the authentication engine
```

This separation is very important.

---

# 28. AuthenticationManager

Now we reach:

```text
AuthenticationManager
```

Its central method is conceptually:

```java
public interface AuthenticationManager {

    Authentication authenticate(
        Authentication authentication
    ) throws AuthenticationException;
}
```

Think of it as:

> **The main entry point into Spring Security's authentication engine.**

---

# 29. AuthenticationManager Input and Output

Input:

```text
Unauthenticated Authentication
```

Then:

```java
authenticationManager.authenticate(
    authentication
);
```

Possible outcomes:

```text
Success
  ↓
Authenticated Authentication

Failure
  ↓
AuthenticationException
```

Mental model:

```text
Authentication request
        ↓
AuthenticationManager
   ┌────┴────┐
Success    Failure
   ↓          ↓
Trusted     Exception
Authentication
```

---

# 30. AuthenticationManager Is an Interface

The common implementation is:

```text
ProviderManager
```

So remember:

```text
AuthenticationManager
        ↑
   interface
        │
ProviderManager
        │
common implementation
```

---

# 31. Why ProviderManager?

A Spring Security application can support multiple authentication mechanisms.

For example:

```text
Database username/password
JWT
OTP
LDAP
SAML
Remember-me
Custom authentication
```

Instead of one giant authentication class handling every mechanism, Spring Security uses:

```text
AuthenticationProvider
```

objects.

`ProviderManager` coordinates them.

---

# 32. ProviderManager Architecture

Conceptually:

```text
ProviderManager
    ├── DaoAuthenticationProvider
    ├── JwtAuthenticationProvider
    ├── RememberMeAuthenticationProvider
    └── CustomOtpAuthenticationProvider
```

Each provider understands a particular authentication mechanism/token type.

---

# 33. AuthenticationProvider

An `AuthenticationProvider` is responsible for authenticating a specific kind of authentication request.

It conceptually exposes:

```java
boolean supports(
    Class<?> authenticationType
);
```

This answers:

> "Can this provider handle this type of Authentication?"

---

# 34. Example: DaoAuthenticationProvider

```text
UsernamePasswordAuthenticationToken
        ↓
DaoAuthenticationProvider
```

So conceptually:

```text
DaoAuthenticationProvider
→ supports UsernamePasswordAuthenticationToken
```

Another:

```text
JwtAuthenticationProvider
→ supports JWT-related authentication
```

---

# 35. ProviderManager Decision

When the request reaches:

```text
ProviderManager
```

it asks:

```text
Which provider supports
this authentication type?
```

For a username/password token:

```text
UsernamePasswordAuthenticationToken
          ↓
DaoAuthenticationProvider
          ↓
authenticate
```

---

# 36. The Bridge to Our Database

From the previous lecture, we already know:

```text
User
UserDetails
UserDetailsService
UserRepository
PasswordEncoder
```

Now we can connect them with the authentication engine:

```text
AuthenticationManager
       ↓
ProviderManager
       ↓
DaoAuthenticationProvider
       ↓
UserDetailsService
       ↓
UserRepository
       ↓
Database
```

And password verification:

```text
DaoAuthenticationProvider
       ↓
PasswordEncoder
       ↓
matches(...)
```

---

# 37. Two Most Important Contracts

Spring Security does not need to understand your complete domain model.

It needs:

```text
UserDetails
```

and:

```text
UserDetailsService
```

Think:

```text
UserDetails
→ Spring Security's view of ONE user

UserDetailsService
→ HOW to FIND that user
```

### Easy memory trick

```text
UserDetails
→ Result

UserDetailsService
→ Finder
```

---

# 38. Application User vs Spring Security User

Your project may have:

```java
com.example.entity.User
```

and Spring Security also has:

```java
org.springframework.security.core.userdetails.User
```

These are **not the same class**.

Your:

```text
User
```

is your domain/database entity.

Spring Security's:

```text
User
```

is a built-in `UserDetails` implementation.

---

# 39. CustomUserDetails

A very clear way to adapt your domain model is:

```java
public class CustomUserDetails
        implements UserDetails {

    private final User user;

    public CustomUserDetails(User user) {
        this.user = user;
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public Collection<? extends GrantedAuthority>
            getAuthorities() {

        return user.getRoles()
            .stream()
            .map(role ->
                new SimpleGrantedAuthority(
                    role.getName()
                )
            )
            .toList();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return user.isEnabled();
    }
}
```

The key idea is:

```text
Domain User
   ↓
CustomUserDetails
   ↓
UserDetails interface
```

---

# 40. Mapping Roles to Authorities

Suppose the database contains:

```text
ROLE_USER
ROLE_ADMIN
```

Custom mapping:

```java
return user.getRoles()
    .stream()
    .map(role ->
        new SimpleGrantedAuthority(
            role.getName()
        )
    )
    .toList();
```

So:

```text
Role entity
   ↓
role.getName()
   ↓
"ROLE_ADMIN"
   ↓
SimpleGrantedAuthority
   ↓
GrantedAuthority
```

---

# 41. UserDetailsService

The question answered by `UserDetailsService` is:

> **Given a login identifier, how should Spring Security find the user?**

Its central method:

```java
UserDetails loadUserByUsername(
    String username
);
```

Remember:

```text
UserDetails
→ one user

UserDetailsService
→ how to find one user
```

---

# 42. CustomUserDetailsService

Conceptually:

```java
@Service
public class CustomUserDetailsService
        implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(
            String username) {

        User user =
            userRepository
                .findByUsername(username)
                .orElseThrow(() ->
                    new UsernameNotFoundException(
                        "User not found"
                    )
                );

        return new CustomUserDetails(user);
    }
}
```

Flow:

```text
username
   ↓
UserRepository
   ↓
User
   ↓
CustomUserDetails
   ↓
UserDetails
```

---

# 43. If Email Is the Login Identifier

The method name can still be:

```java
loadUserByUsername(...)
```

but internally:

```java
userRepository.findByEmail(email);
```

Example:

```java
@Override
public UserDetails loadUserByUsername(
        String email) {

    User user =
        userRepository
            .findByEmail(email)
            .orElseThrow(() ->
                new UsernameNotFoundException(
                    "User not found"
                )
            );

    return new CustomUserDetails(user);
}
```

The interface name is historical; the actual identifier is an application-design choice.

---

# 44. Is CustomUserDetails Compulsory?

No.

You can use Spring Security's built-in:

```text
org.springframework.security.core.userdetails.User
```

implementation.

For example:

```java
return User
    .withUsername(user.getUsername())
    .password(user.getPassword())
    .authorities(
        user.getRoles()
            .stream()
            .map(role ->
                new SimpleGrantedAuthority(
                    role.getName()
                )
            )
            .toList()
    )
    .disabled(!user.isEnabled())
    .build();
```

Custom `UserDetails` is useful when:

```text
You want explicit mapping
You want access to additional domain-user properties
You want a clear separation between domain and security layers
```

---

# 45. DaoAuthenticationProvider — The Workhorse

Now the most important provider:

```text
DaoAuthenticationProvider
```

It performs the username/password authentication process using:

```text
UserDetailsService
+
PasswordEncoder
```

Conceptually:

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

---

# 46. What Does DaoAuthenticationProvider Actually Do?

It can be understood as:

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

This is the internal heart of database username/password authentication.

---

# 47. Why Is It Called DAO Authentication Provider?

DAO stands for:

```text
Data Access Object
```

The name reflects the fact that user information is obtained from a user-data source before credentials are verified.

In modern Spring applications, the data access may be:

```text
JPA repository
JDBC
Mongo repository
Custom data access
```

The important idea is:

```text
provider retrieves user data
then verifies credentials
```

---

# 48. Complete Internal Login Flow

Now combine everything.

```text
HTTP Request
       ↓
Authentication Filter
       ↓
Unauthenticated UsernamePasswordAuthenticationToken
       ↓
AuthenticationManager
       ↓
ProviderManager
       ↓
DaoAuthenticationProvider
       ├── UserDetailsService
       │      ↓
       │   UserRepository
       │      ↓
       │   Database
       │
       └── PasswordEncoder
              ↓
         Password verification
       ↓
Authenticated UsernamePasswordAuthenticationToken
       ↓
SecurityContext
       ↓
SecurityContextHolder
       ↓
Authorization
       ↓
Controller
```

This is the **master diagram for Lecture 3**.

---

# 49. Step 1 — Filter Extracts Credentials

For form login:

```text
UsernamePasswordAuthenticationFilter
```

reads:

```text
username
password
```

For HTTP Basic:

```text
BasicAuthenticationFilter
```

reads credentials from the:

```http
Authorization
```

header.

---

# 50. Step 2 — Create Unauthenticated Token

Example input:

```text
username = aditya
password = secret123
```

becomes:

```text
UsernamePasswordAuthenticationToken

principal     = "aditya"
credentials   = "secret123"
authorities   = []
authenticated = false
```

At this stage:

```text
NOT TRUSTED
```

---

# 51. Step 3 — Call AuthenticationManager

The filter calls:

```java
authenticationManager.authenticate(
    authenticationToken
);
```

This is the handoff:

```text
Filter
  ↓
Authentication Engine
```

---

# 52. Step 4 — ProviderManager Selects Provider

`ProviderManager` sees:

```text
UsernamePasswordAuthenticationToken
```

and looks for a provider that supports that authentication type.

It selects:

```text
DaoAuthenticationProvider
```

for database username/password authentication.

---

# 53. Step 5 — Extract the Username

`DaoAuthenticationProvider` gets the claimed identity:

```text
principal = "aditya"
```

Then:

```java
userDetailsService
    .loadUserByUsername("aditya");
```

---

# 54. Step 6 — Database Query

Your `UserDetailsService` calls:

```java
userRepository
    .findByUsername(username)
```

which reaches:

```text
Database
```

Suppose it returns:

```text
User
username = aditya
password = $2a$10$...
enabled = true
roles = [ROLE_USER]
```

Then the User is adapted into:

```text
UserDetails
```

---

# 55. Step 7 — Account Status Check

Spring Security can inspect:

```java
isEnabled()
isAccountNonLocked()
isAccountNonExpired()
isCredentialsNonExpired()
```

Example:

```text
enabled = false
```

Then:

```text
Authentication fails
```

even if the password is correct.

### Important

> **Database authentication is not only password comparison. Account state is part of authentication.**

---

# 56. Step 8 — Password Verification

Provider now has:

```text
Submitted password:
secret123

Stored encoded password:
$2a$10$...
```

It delegates to:

```java
passwordEncoder.matches(
    rawPassword,
    userDetails.getPassword()
);
```

Result:

```text
true
```

or:

```text
false
```

---

# 57. Password Is Not Encoded Again and Compared

Do not imagine:

```text
encode(rawPassword)
==
storedEncodedPassword
```

With BCrypt, that is the wrong mental model because a random salt is involved.

Instead:

```text
stored encoded BCrypt value
        ↓
salt + parameters available
        ↓
verify supplied raw password
        ↓
true / false
```

---

# 58. Step 9 — Successful Authentication

If:

```text
User exists
+
Account is valid
+
Password matches
```

the provider returns a trusted authentication object.

Conceptually:

```text
UsernamePasswordAuthenticationToken

principal     = CustomUserDetails
credentials   = null / erased
authorities   = [ROLE_USER]
authenticated = true
```

This is a major transformation:

```text
Unverified Claim
      ↓
Verification
      ↓
Trusted Identity
```

---

# 59. Why Are Credentials Erased?

After successful authentication, the raw password should not remain around longer than necessary.

So credentials can be:

```text
null
```

or otherwise erased/protected.

Conceptually:

```text
Before:
credentials = secret123

After:
credentials = null / erased
```

This follows the security principle:

> **Do not retain sensitive authentication material longer than necessary.**

---

# 60. Step 10 — Authentication Goes Into SecurityContext

After successful authentication:

```text
Authentication
      ↓
SecurityContext
      ↓
SecurityContextHolder
```

Now later parts of the request can know:

```text
Who is the user?
Which authorities do they have?
Is the user authenticated?
```

---

# 61. Step 11 — Authorization

Now authorization can finally happen.

Examples:

```java
.requestMatchers("/admin/**")
    .hasRole("ADMIN")
```

or:

```java
.anyRequest()
    .authenticated()
```

At this point Spring Security has enough information to evaluate:

```text
Who?
What authorities?
Which endpoint?
Which rule?
```

---

# 62. Step 12 — Controller Executes

If authorization succeeds:

```text
Security filters
     ↓
DispatcherServlet
     ↓
Controller
```

The controller does not need to:

```text
query user
verify password
rebuild Authentication
```

Spring Security already handled that.

---

# 63. Complete Flow With Real Values

Input:

```text
username = aditya
password = secret123
```

### Stage 1

```text
HTTP Request
```

### Stage 2

```text
UsernamePasswordAuthenticationToken
principal = "aditya"
credentials = "secret123"
authenticated = false
```

### Stage 3

```text
AuthenticationManager
```

### Stage 4

```text
ProviderManager
```

### Stage 5

```text
DaoAuthenticationProvider
```

### Stage 6

```text
UserDetailsService
```

### Stage 7

```text
UserRepository
```

### Stage 8

```text
Database
User = aditya
Password = $2a$10$...
Roles = ROLE_USER
```

### Stage 9

```text
PasswordEncoder.matches(
    "secret123",
    "$2a$10$..."
)
```

Result:

```text
true
```

### Stage 10

```text
Authentication
principal = UserDetails(aditya)
authorities = [ROLE_USER]
authenticated = true
```

### Stage 11

```text
SecurityContext
```

### Stage 12

```text
SecurityContextHolder
```

### Stage 13

```text
Authorization
```

### Stage 14

```text
Controller
```

---

# 64. What Happens When Authentication Fails?

There are several failure paths.

## User Not Found

```java
throw new UsernameNotFoundException(
    "User not found"
);
```

---

## Wrong Password

```java
passwordEncoder.matches(
    "wrongPassword",
    storedEncodedPassword
);
```

returns:

```text
false
```

Authentication commonly fails with:

```text
BadCredentialsException
```

---

## Invalid Account State

Authentication can also fail if the account is:

```text
Disabled
Locked
Expired
Using expired credentials
```

In failure cases:

```text
No trusted authenticated result
       ↓
SecurityContext is not populated
with successful authentication
```

---

# 65. Important Security Lesson About Failure

The internal reasons may be different:

```text
User not found
Wrong password
Account disabled
Account locked
```

but applications often avoid exposing overly specific external messages that help attackers enumerate accounts.

Internal diagnostic detail can be more specific than public error messages.

---

# 66. Form Login and HTTP Basic Share the Same Backend

This is a very important architecture concept.

## Form Login

```text
Form Login
   ↓
UsernamePasswordAuthenticationFilter
   ↓
UsernamePasswordAuthenticationToken
   ↓
AuthenticationManager
   ↓
DaoAuthenticationProvider
   ↓
Database authentication
```

## HTTP Basic

```text
HTTP Basic
   ↓
BasicAuthenticationFilter
   ↓
UsernamePasswordAuthenticationToken
   ↓
AuthenticationManager
   ↓
DaoAuthenticationProvider
   ↓
Database authentication
```

### Key insight

```text
Different Filter
        ↓
Same Authentication Engine
```

---

# 67. Registration and Login Are Different Flows

Do not mix these two.

## Registration

The user does not exist yet.

```text
Client
 ↓
username + raw password
 ↓
AuthService
 ↓
PasswordEncoder.encode(...)
 ↓
BCrypt encoded password
 ↓
UserRepository
 ↓
Database
```

---

## Login

The user already exists.

```text
Client
 ↓
username + raw password
 ↓
Authentication Filter
 ↓
AuthenticationManager
 ↓
DaoAuthenticationProvider
 ↓
Load existing user
 ↓
PasswordEncoder.matches(...)
 ↓
Authenticated Authentication
```

### Golden rule

```text
Registration
→ encode()

Login
→ matches()
```

---

# 68. Registration Should Not Allow Client-Selected Admin Role

This is an important security issue.

Unsafe request:

```json
{
  "username": "hacker",
  "password": "123",
  "role": "ROLE_ADMIN"
}
```

If the backend trusts this:

```text
Anyone can register
as ADMIN
```

That is a privilege-escalation vulnerability.

Instead the server should assign a safe default role.

Example:

```java
Role role = roleRepository
    .findByName("ROLE_USER")
    .orElseThrow();

user.getRoles().add(role);
```

### Golden rule

> **The client can request an account; the server decides the privileges.**

---

# 69. Registration Response Should Not Return the Entity Blindly

If your entity contains:

```text
username
password
roles
```

returning the entire entity from a registration endpoint can accidentally expose sensitive data.

Instead use a response DTO:

```java
public record UserResponse(
    Long id,
    String username
) {}
```

Then return only what the client should see.

---

# 70. Suggested Project Structure

A clean structure:

```text
src/main/java
│
├── entity/
│   ├── User.java
│   └── Role.java
│
├── repository/
│   ├── UserRepository.java
│   └── RoleRepository.java
│
├── dto/
│   ├── RegisterRequest.java
│   └── UserResponse.java
│
├── security/
│   ├── CustomUserDetails.java
│   └── CustomUserDetailsService.java
│
├── service/
│   └── AuthService.java
│
├── controller/
│   ├── AuthController.java
│   └── TestController.java
│
└── config/
    └── SecurityConfig.java
```

### Responsibility map

```text
Entity
→ database/domain model

Repository
→ database access

DTO
→ API request/response structure

Security
→ Spring Security adapters

Service
→ business logic

Controller
→ HTTP layer

Config
→ security configuration
```

---

# 71. Complete Implementation Pieces

## User Entity

```java
@Entity
@Table(name = "users")
@Getter
@Setter
@NoArgsConstructor
public class User {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        unique = true,
        nullable = false
    )
    private String username;

    @Column(nullable = false)
    private String password;

    private boolean enabled = true;

    @ManyToMany
    @JoinTable(
        name = "user_roles",
        joinColumns =
            @JoinColumn(name = "user_id"),
        inverseJoinColumns =
            @JoinColumn(name = "role_id")
    )
    private Set<Role> roles =
        new HashSet<>();
}
```

---

# 72. Role Entity

```java
@Entity
@Table(name = "roles")
@Getter
@Setter
@NoArgsConstructor
public class Role {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        unique = true,
        nullable = false
    )
    private String name;
}
```

Recommended values:

```text
ROLE_USER
ROLE_ADMIN
```

---

# 73. Role Prefix

If the database stores:

```text
ROLE_ADMIN
```

then:

```java
.hasRole("ADMIN")
```

typically checks for:

```text
ROLE_ADMIN
```

This is a very important convention.

So:

```java
hasRole("ADMIN")
```

is conceptually related to:

```text
hasAuthority("ROLE_ADMIN")
```

---

# 74. Repositories

```java
public interface UserRepository
        extends JpaRepository<User, Long> {

    Optional<User> findByUsername(
        String username
    );
}
```

```java
public interface RoleRepository
        extends JpaRepository<Role, Long> {

    Optional<Role> findByName(
        String name
    );
}
```

---

# 75. RegisterRequest

Simple DTO:

```java
public record RegisterRequest(
    String username,
    String password
) {}
```

In production, add appropriate validation such as:

```text
@NotBlank
@Size
```

according to your password policy.

---

# 76. UserResponse

Never return password through the registration response.

Example:

```java
public record UserResponse(
    Long id,
    String username
) {}
```

---

# 77. CustomUserDetails

```java
public class CustomUserDetails
        implements UserDetails {

    private final User user;

    public CustomUserDetails(User user) {
        this.user = user;
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public Collection<? extends GrantedAuthority>
            getAuthorities() {

        return user.getRoles()
            .stream()
            .map(role ->
                new SimpleGrantedAuthority(
                    role.getName()
                )
            )
            .toList();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return user.isEnabled();
    }
}
```

---

# 78. CustomUserDetailsService

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService
        implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(
            String username) {

        User user =
            userRepository
                .findByUsername(username)
                .orElseThrow(() ->
                    new UsernameNotFoundException(
                        "User not found"
                    )
                );

        return new CustomUserDetails(user);
    }
}
```

### One-line purpose

```text
Find domain User
+
adapt it
→
UserDetails
```

---

# 79. PasswordEncoder Bean

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

Use:

```java
PasswordEncoder
```

throughout the application rather than depending directly on the concrete class.

---

# 80. Explicit DaoAuthenticationProvider

For learning, making the provider visible helps us understand its dependencies:

```java
@Bean
public DaoAuthenticationProvider
authenticationProvider(
    CustomUserDetailsService userDetailsService,
    PasswordEncoder passwordEncoder
) {

    DaoAuthenticationProvider provider =
        new DaoAuthenticationProvider(
            userDetailsService
        );

    provider.setPasswordEncoder(
        passwordEncoder
    );

    return provider;
}
```

Architecture:

```text
DaoAuthenticationProvider
     ├── CustomUserDetailsService
     └── PasswordEncoder
```

Spring Security can often auto-configure much of this infrastructure when the appropriate beans exist, but explicitly defining it is useful for learning the internals.

---

# 81. Security Configuration

The lecture's classroom API configuration is conceptually:

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public DaoAuthenticationProvider
    authenticationProvider(
        CustomUserDetailsService userDetailsService,
        PasswordEncoder passwordEncoder
    ) {

        DaoAuthenticationProvider provider =
            new DaoAuthenticationProvider(
                userDetailsService
            );

        provider.setPasswordEncoder(
            passwordEncoder
        );

        return provider;
    }

    @Bean
    public SecurityFilterChain
    securityFilterChain(
        HttpSecurity http,
        DaoAuthenticationProvider provider
    ) throws Exception {

        http
            .csrf(csrf -> csrf.disable())

            .authenticationProvider(provider)

            .authorizeHttpRequests(auth -> auth

                .requestMatchers(
                    "/register"
                ).permitAll()

                .requestMatchers(
                    "/admin/**"
                ).hasRole("ADMIN")

                .anyRequest()
                .authenticated()
            )

            .formLogin(
                Customizer.withDefaults()
            )

            .httpBasic(
                Customizer.withDefaults()
            );

        return http.build();
    }
}
```

---

# 82. Important Warning About CSRF Disable

The course example uses:

```java
.csrf(csrf -> csrf.disable())
```

for the classroom API demonstration.

This is a **demonstration decision**, not a universal production recommendation.

Why?

CSRF risk depends heavily on how authentication credentials are transported and whether browsers automatically attach them to requests.

Understand:

```text
CSRF disabled for learning/demo
≠
CSRF should always be disabled
```

---

# 83. Registration Service

Conceptually:

```java
@Service
@RequiredArgsConstructor
public class AuthService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final PasswordEncoder passwordEncoder;

    public UserResponse register(
            RegisterRequest request) {

        User user = new User();

        user.setUsername(
            request.username()
        );

        user.setPassword(
            passwordEncoder.encode(
                request.password()
            )
        );

        user.setEnabled(true);

        Role role = roleRepository
            .findByName("ROLE_USER")
            .orElseThrow();

        user.getRoles().add(role);

        User saved =
            userRepository.save(user);

        return new UserResponse(
            saved.getId(),
            saved.getUsername()
        );
    }
}
```

The critical security decisions are:

```text
Encode password
Assign default role on server
Return safe DTO
```

---

# 84. Registration Controller

```java
@RestController
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/register")
    public UserResponse register(
        @RequestBody RegisterRequest request
    ) {

        return authService.register(
            request
        );
    }
}
```

---

# 85. Test Controller

Protected endpoint:

```java
@RestController
public class TestController {

    @GetMapping("/profile")
    public String profile() {
        return "Welcome to your profile";
    }
}
```

Because:

```java
.anyRequest().authenticated()
```

exists:

```text
/profile
```

requires authentication.

---

# 86. Inspecting the Current Authentication

You can make the authenticated object visible during learning:

```java
@GetMapping("/me")
public Map<String, Object> me(
        Authentication authentication) {

    return Map.of(
        "username",
        authentication.getName(),

        "authorities",
        authentication.getAuthorities(),

        "authenticated",
        authentication.isAuthenticated()
    );
}
```

Possible response:

```json
{
  "username": "aditya",
  "authorities": [
    {
      "authority": "ROLE_USER"
    }
  ],
  "authenticated": true
}
```

This is an excellent learning/debugging endpoint because it makes the authenticated security object visible.

Do not expose sensitive internal security information in production without considering the security implications.

---

# 87. Demo Sequence

A very good way to understand the complete system is to test it in this order.

## Step 1 — Ensure Default Role

Database contains:

```text
ROLE_USER
```

---

## Step 2 — Register User

```http
POST /register
Content-Type: application/json

{
    "username": "aditya",
    "password": "secret123"
}
```

Check the database:

```text
username = aditya
password = $2a$...
role = ROLE_USER
```

Notice:

```text
raw password is NOT stored
```

---

## Step 3 — Access Protected Endpoint Without Credentials

```http
GET /me
```

Expected:

```text
Authentication required
```

---

## Step 4 — Access With Correct HTTP Basic Credentials

```text
Username:
aditya

Password:
secret123
```

Now:

```text
UserDetails loaded
      ↓
Password verified
      ↓
Authentication created
      ↓
SecurityContext
      ↓
Authorization
      ↓
/me response
```

---

## Step 5 — Try Wrong Password

```text
aditya
wrongPassword
```

Authentication fails.

---

## Step 6 — Disable the User

Database:

```text
enabled = false
```

Try the correct password again.

Authentication still fails.

Why?

Because:

```text
Authentication
≠
only password comparison
```

Account state is part of the authentication decision.

---

# 88. The Excalidraw Notes — How to Read Them

The supplied Excalidraw diagrams are especially useful for building the mental model.

## Page 1

Shows the transformation:

```text
Unauthenticated Authentication
        ↓
Authenticated Authentication
```

and the login/database relationship:

```text
Login
  ↓
Application
  ↓
Database
```

The important idea is:

```text
Claim
→ verify
→ trusted identity
```

---

## Page 2

Shows:

```text
UserDetails
```

on the Spring Security side and:

```text
User
```

on the application side.

It also visually shows:

```text
User
 ↕
Role
```

as a many-to-many relationship.

---

## Page 3

Shows the physical database design:

```text
User
Role
User_role
```

The join table contains:

```text
user_id
role_id
```

This is the database representation of:

```text
User ↔ Role
```

---

## Page 4

Shows the password-security picture:

```text
Raw password
+
Salt
 ↓
BCrypt
 ↓
Encoded representation
 ↓
Database
```

It also contrasts secure password storage with the idea of precomputed password/hash tables.

---

## Page 5

Shows the adapter architecture:

```text
Application User
        ↓
CustomUserDetails
        ↓
UserDetails
```

and:

```text
CustomUserDetailsService
        ↓
UserDetails
```

This is the key reason `UserDetails` and your own `User` entity should not be treated as identical concepts.

---

## Page 6

Shows the work done by `DaoAuthenticationProvider`:

```text
Receive authentication request
        ↓
Ask UserDetailsService
        ↓
Get stored password
        ↓
Ask PasswordEncoder to match
        ↓
Check account state
        ↓
Return Authentication
```

This is one of the best diagrams for understanding the internal authentication engine.

---

## Page 7

Shows the request path:

```text
Client
 ↓
Tomcat
 ↓
Filters
 ↓
DispatcherServlet
 ↓
Controller
```

and the conceptual security stages:

```text
Exploit Protection
 ↓
Authentication
 ↓
Exception Handling
 ↓
Authorization
```

---

## Page 8

Shows the architecture behind:

```text
Tomcat
 ↓
DelegatingFilterProxy
 ↓
FilterChainProxy
 ↓
SecurityFilterChain
```

and multiple chains such as:

```text
/api/**
/admin/**
/**
```

---

## Page 9

Shows that:

```text
HttpSecurity
    ↓
SecurityFilterChain
```

and the default chain can contain filters such as:

```text
SecurityContextHolderFilter
CsrfFilter
LogoutFilter
UsernamePasswordAuthenticationFilter
BasicAuthenticationFilter
AnonymousAuthenticationFilter
ExceptionTranslationFilter
AuthorizationFilter
```

The exact filters depend on enabled features.

---

## Page 10

Shows the authentication filters:

```text
Form Login
   ↓
UsernamePasswordAuthenticationFilter

HTTP Basic
   ↓
BasicAuthenticationFilter
```

This makes the key distinction clear:

```text
Different credential transport
→ different filter
```

---

## Page 11

Shows the most important login transformation:

```text
POST /login
       ↓
Authentication Filter
       ↓
UsernamePasswordAuthenticationToken
       ↓
AuthenticationManager
       ↓
Authentication
```

and the transformation:

```text
authenticated = false
        ↓
verified
        ↓
authenticated = true
```

---

## Page 12

Shows the provider architecture:

```text
AuthenticationManager
        ↓
ProviderManager
   ┌────┼─────────────┐
   ↓    ↓             ↓
DAO   JWT           Custom OTP
Provider Provider     Provider
```

with each provider being selected according to the authentication type it supports.

This diagram is the bridge between today's database authentication and future topics such as JWT and custom OTP authentication.

---

# 89. The Most Important Internal Architecture

Memorize this:

```text
Tomcat
  ↓
DelegatingFilterProxy
  ↓
FilterChainProxy
  ↓
SecurityFilterChain
  ↓
Authentication Filter
  ↓
UsernamePasswordAuthenticationToken
  ↓
AuthenticationManager
  ↓
ProviderManager
  ↓
DaoAuthenticationProvider
  ├── UserDetailsService
  │      ↓
  │   UserRepository
  │      ↓
  │   Database
  │
  └── PasswordEncoder
         ↓
       BCrypt
  ↓
Authenticated Authentication
  ↓
SecurityContext
  ↓
SecurityContextHolder
  ↓
Authorization
  ↓
Controller
```

---

# 90. The Four Layers of the Entire Login Process

A very powerful way to revise this lecture is through **four layers**.

## Layer 1 — Receive Credentials

```text
UsernamePasswordAuthenticationFilter
```

or:

```text
BasicAuthenticationFilter
```

---

## Layer 2 — Represent the Request

```text
UsernamePasswordAuthenticationToken
```

---

## Layer 3 — Authenticate the Request

```text
AuthenticationManager
        ↓
ProviderManager
        ↓
DaoAuthenticationProvider
```

---

## Layer 4 — Load and Verify User

```text
UserDetailsService
        ↓
UserRepository
        ↓
Database

PasswordEncoder
        ↓
Password verification
```

After success:

```text
Authenticated Authentication
        ↓
SecurityContext
        ↓
SecurityContextHolder
        ↓
Authorization
        ↓
Controller
```

---

# 91. Interview Comparisons

## Authentication Filter vs AuthenticationManager

```text
Authentication Filter
→ extracts credentials
→ creates Authentication request

AuthenticationManager
→ authenticates that request
```

---

## AuthenticationManager vs ProviderManager

```text
AuthenticationManager
→ interface / authentication entry point

ProviderManager
→ common implementation that delegates
  to AuthenticationProvider objects
```

---

## ProviderManager vs DaoAuthenticationProvider

```text
ProviderManager
→ chooses/delegates to a provider

DaoAuthenticationProvider
→ performs database username/password authentication
```

---

## UserDetails vs UserDetailsService

```text
UserDetails
→ one user's security representation

UserDetailsService
→ how that user is found
```

---

## UserDetailsService vs UserRepository

```text
UserDetailsService
→ Spring Security adapter

UserRepository
→ database/domain data-access component
```

---

## PasswordEncoder vs BCryptPasswordEncoder

```text
PasswordEncoder
→ abstraction

BCryptPasswordEncoder
→ concrete implementation
```

---

## Authentication vs Authorization

```text
Authentication
→ who is the user?

Authorization
→ what can the user do?
```

---

## Form Login vs HTTP Basic

```text
Form Login
→ UsernamePasswordAuthenticationFilter

HTTP Basic
→ BasicAuthenticationFilter
```

But both can use:

```text
AuthenticationManager
→ ProviderManager
→ DaoAuthenticationProvider
→ Database
```

---

# 92. Common Beginner Mistakes

### Mistake 1

> "`SecurityFilterChain` is the same as `FilterChainProxy`."

Wrong.

```text
FilterChainProxy
→ manages/selects a SecurityFilterChain

SecurityFilterChain
→ matcher + ordered security filters
```

---

### Mistake 2

> "Every authentication filter authenticates every request."

Wrong.

Filters generally look for requests that contain the authentication mechanism they handle.

---

### Mistake 3

> "UsernamePasswordAuthenticationToken always means authenticated user."

Wrong.

It can represent:

```text
Unauthenticated authentication request
```

and later:

```text
Authenticated result
```

The state matters.

---

### Mistake 4

> "Authentication filter checks the database."

Not its primary responsibility.

The filter extracts credentials and delegates to the authentication engine.

---

### Mistake 5

> "AuthenticationManager directly queries the database."

Not necessarily.

The common flow is:

```text
AuthenticationManager
→ ProviderManager
→ DaoAuthenticationProvider
→ UserDetailsService
→ Repository
→ Database
```

---

### Mistake 6

> "DaoAuthenticationProvider stores users."

No.

It authenticates using:

```text
UserDetailsService
+
PasswordEncoder
```

---

### Mistake 7

> "UserDetailsService is the database."

No.

It is the service contract for **finding/loading the user**.

---

### Mistake 8

> "CustomUserDetails is mandatory."

No.

You can use Spring Security's built-in `User` implementation.

---

### Mistake 9

> "If password is correct, login always succeeds."

Not necessarily.

Account state can reject authentication:

```text
disabled
locked
expired
credentials expired
```

---

### Mistake 10

> "Registration can trust a role sent by the client."

Dangerous.

The server should assign privileges.

---

### Mistake 11

> "Form login and Basic authentication are completely different systems."

Their credential transport/filter differs, but they can share the same authentication engine.

---

# 93. 🎤 Interview Questions

## Q1. How does a request enter Spring Security?

A servlet request passes through the servlet filter mechanism. In Spring Security, `DelegatingFilterProxy` bridges the servlet container to Spring-managed security infrastructure, which leads to `FilterChainProxy` and the selected `SecurityFilterChain`.

---

## Q2. What is DelegatingFilterProxy?

A servlet filter that delegates request processing to a Spring-managed filter bean.

---

## Q3. What is FilterChainProxy?

The central Spring Security servlet filter that selects the first matching `SecurityFilterChain` and executes its filters.

---

## Q4. What is a SecurityFilterChain?

A request matcher plus an ordered list of security filters.

---

## Q5. What happens if multiple SecurityFilterChains match?

The first matching chain wins. Spring Security does not combine the matching chains.

---

## Q6. Why should specific chains come before `/**`?

Because the first matching chain wins, and `/**` is very broad.

---

## Q7. What is HttpSecurity?

A builder used to configure and construct a `SecurityFilterChain`.

---

## Q8. What is an authentication filter's job?

Extract credentials, create an `Authentication` request object, and hand it to the authentication engine.

---

## Q9. What is UsernamePasswordAuthenticationToken?

An `Authentication` implementation commonly used for username/password authentication. It can represent the initial unverified request and, after success, the authenticated result.

---

## Q10. What is AuthenticationManager?

The entry point into the authentication engine.

---

## Q11. What is ProviderManager?

A common `AuthenticationManager` implementation that delegates authentication to configured `AuthenticationProvider` instances.

---

## Q12. What is AuthenticationProvider?

A component responsible for authenticating a particular kind/type of authentication request.

---

## Q13. What is DaoAuthenticationProvider?

An `AuthenticationProvider` designed for username/password authentication using `UserDetailsService` and `PasswordEncoder`.

---

## Q14. What does `supports()` do?

It tells Spring Security whether a provider can handle a particular `Authentication` type.

---

## Q15. What is the role of UserDetailsService?

Load the user's security information from the application's user store.

---

## Q16. What is the role of UserDetails?

Represent one user's security information in a form Spring Security understands.

---

## Q17. Where does password verification happen?

Conceptually through the configured `PasswordEncoder`, commonly inside `DaoAuthenticationProvider` during username/password authentication.

---

## Q18. What happens after successful authentication?

An authenticated `Authentication` is placed into the `SecurityContext`, accessible through `SecurityContextHolder`, and authorization rules can then be evaluated.

---

## Q19. Why are credentials erased?

To avoid keeping sensitive authentication material, such as a raw password, longer than necessary.

---

## Q20. Do form login and HTTP Basic need different database authentication logic?

No. They use different filters for different credential transport formats but can share the same `AuthenticationManager`/provider/database authentication pipeline.

---

# 94. ⭐ If You Remember Only These 25 Things

```text
1. Spring Security runs before the controller.

2. Servlet filters are the foundation of the servlet security architecture.

3. DelegatingFilterProxy bridges Tomcat and Spring-managed security beans.

4. FilterChainProxy manages SecurityFilterChain selection.

5. SecurityFilterChain = matcher + ordered filters.

6. First matching SecurityFilterChain wins.

7. Specific chains should normally be ordered before broad chains.

8. HttpSecurity builds the SecurityFilterChain.

9. Authentication filter extracts credentials.

10. Authentication filter does not perform the entire database authentication itself.

11. UsernamePasswordAuthenticationToken represents username/password
    authentication.

12. Before verification it is unauthenticated.

13. AuthenticationManager is the authentication entry point.

14. ProviderManager is a common AuthenticationManager implementation.

15. ProviderManager delegates to AuthenticationProvider objects.

16. DaoAuthenticationProvider handles username/password authentication.

17. UserDetailsService finds the user.

18. UserDetails represents one security user.

19. UserRepository loads the domain User from the database.

20. PasswordEncoder verifies the raw password against the stored representation.

21. Account status is part of authentication.

22. After success, Authentication goes into SecurityContext.

23. SecurityContextHolder provides access to that current context.

24. Authorization happens after authentication.

25. Different filters can share the same authentication backend.
```

---

# 95. 🔥 One-Minute Revision

> **Spring Security is built on servlet filters and executes before the controller. Tomcat reaches `DelegatingFilterProxy`, which connects the servlet container to Spring Security's `FilterChainProxy`. `FilterChainProxy` selects the first matching `SecurityFilterChain`, where the chain contains an ordered set of security filters. `HttpSecurity` is the builder used to configure that chain. For username/password authentication, an authentication filter extracts the credentials and creates an unauthenticated `UsernamePasswordAuthenticationToken`; it does not contain the full database-verification logic. The token is passed to `AuthenticationManager`, commonly implemented by `ProviderManager`, which selects a compatible `AuthenticationProvider`. For database username/password authentication that provider is `DaoAuthenticationProvider`, which loads a `UserDetails` through `UserDetailsService` and the repository, checks account state, and verifies the password through `PasswordEncoder`. On success, it returns a trusted authenticated `Authentication`, which is stored in `SecurityContext` and accessed through `SecurityContextHolder`. Authorization rules are then evaluated, and only after the request is allowed does it continue to the controller. Form login and HTTP Basic use different authentication filters but can share this same authentication backend.**

---

# 96. Complete Master Diagram

```text
                              CLIENT
                                 │
                                 ↓
                           HTTP REQUEST
                                 │
                                 ↓
                               TOMCAT
                                 │
                                 ↓
                      DelegatingFilterProxy
                                 │
                                 ↓
                         FilterChainProxy
                                 │
                    selects first matching
                                 │
                                 ↓
                      SecurityFilterChain
                                 │
                ┌────────────────┴───────────────┐
                │                                │
       Request / Security                    Filters
          matching                             │
                                                ↓
                                 ┌────────────────────────┐
                                 │ Authentication Filter  │
                                 │                        │
                                 │ Form → Username...    │
                                 │ Basic → Basic...      │
                                 └───────────┬────────────┘
                                             ↓
                              UsernamePasswordAuthenticationToken
                                             │
                                   authenticated = false
                                             ↓
                                  AuthenticationManager
                                             ↓
                                       ProviderManager
                                             ↓
                                  DaoAuthenticationProvider
                                     ┌───────┴────────┐
                                     │                │
                                     ↓                ↓
                           UserDetailsService   PasswordEncoder
                                     │                │
                                     ↓                ↓
                              UserRepository        BCrypt
                                     │
                                     ↓
                                  Database
                                     │
                                     ↓
                                  User Entity
                                     │
                                     ↓
                                UserDetails
                                     │
                                     └────────┐
                                              ↓
                                      Password Verification
                                              │
                                              ↓
                                  Authenticated Authentication
                                              │
                                              ↓
                                       SecurityContext
                                              │
                                              ↓
                                      SecurityContextHolder
                                              │
                                              ↓
                                         Authorization
                                              │
                                       ┌──────┴──────┐
                                       │             │
                                     DENY          ALLOW
                                       │             │
                                       ↓             ↓
                                    401/403      DispatcherServlet
                                                     │
                                                     ↓
                                                  Controller
                                                     │
                                                     ↓
                                                   Service
                                                     │
                                                     ↓
                                                 Database
```

---

# 97. The Single Most Important Flow to Draw Yourself

Before an interview, try drawing this from memory:

```text
POST /login
     ↓
UsernamePasswordAuthenticationFilter
     ↓
UsernamePasswordAuthenticationToken
     ↓
AuthenticationManager
     ↓
ProviderManager
     ↓
DaoAuthenticationProvider
     ├── UserDetailsService
     │      ↓
     │   UserRepository
     │      ↓
     │   Database
     │
     └── PasswordEncoder
            ↓
          BCrypt
     ↓
Authenticated Authentication
     ↓
SecurityContext
     ↓
SecurityContextHolder
     ↓
Authorization
     ↓
Controller
```

If you can explain every box in this diagram in your own words, you understand the core of Lecture 3.

---

# 98. What Lecture 3 Adds to Lecture 2

Lecture 2 taught:

```text
Database
 ↓
UserRepository
 ↓
UserDetailsService
 ↓
UserDetails
 ↓
PasswordEncoder
```

Lecture 3 adds the **authentication engine and request pipeline**:

```text
HTTP Request
 ↓
Security Filters
 ↓
Authentication Token
 ↓
AuthenticationManager
 ↓
ProviderManager
 ↓
DaoAuthenticationProvider
 ↓
UserDetailsService
 ↓
Repository / Database
 ↓
PasswordEncoder
 ↓
Authenticated Authentication
 ↓
SecurityContext
 ↓
Authorization
```

So the progression is:

```text
Lecture 1
→ Security concepts

Lecture 2
→ Database user + password security

Lecture 3
→ Internal authentication engine
```

That progression is important because now the previously learned components have a place in the larger architecture.

---

# Lecture 3 Core Principle

> **Spring Security database authentication is a chain of responsibilities, not one magical login method. The filter extracts credentials, creates an authentication request, and delegates to the authentication engine. `ProviderManager` selects the appropriate `AuthenticationProvider`; `DaoAuthenticationProvider` loads the user through `UserDetailsService`, obtains the stored password and authorities, checks account state, and verifies the submitted password through `PasswordEncoder`. On success, Spring Security creates a trusted `Authentication`, stores it in the `SecurityContext`, and only then performs authorization before allowing the request to reach the controller.**
