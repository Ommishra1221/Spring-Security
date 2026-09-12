# Spring Security — Lecture 2 Notes
## Database Authentication & Password Security

This lecture moves from Spring Security's development-time default user to a real **database-backed authentication system**.

The central problem is:

```text
Client sends username + password
              ↓
How does Spring Security know
whether they are correct?
              ↓
It needs trusted user information
              ↓
Database
              ↓
UserRepository
              ↓
UserDetailsService
              ↓
UserDetails
              ↓
PasswordEncoder
              ↓
Authenticated Authentication
```

The lecture also introduces the database model for users and roles, `UserDetails`, `UserDetailsService`, `DaoAuthenticationProvider`, password hashing, BCrypt, salting, registration, and secure password verification.

---

# 1. From Default Authentication to Database Authentication

When Spring Security is added to a Spring Boot application, Boot can automatically provide development-time security.

A default setup can provide:

```text
username = user
password = generated password
authority = ROLE_USER
```

and protect endpoints with authentication.

This is useful for:

```text
Learning
Testing
Prototypes
```

But it is not a real user-management system.

A real application needs:

```text
Many users
Persistent accounts
Passwords
Roles
Account status
Data that survives restarts
Data that survives deployments
Data that works when the application scales
```

So we move from:

```text
Default User
     ↓
Database Authentication
```

---

# 2. Authentication Needs a Source of Truth

Suppose a client sends:

```text
username = aditya
password = secret123
```

Spring Security cannot simply trust these values.

It needs trusted information against which the claim can be validated.

Think:

```text
USER-SUBMITTED DATA
-------------------
username = aditya
password = secret123


TRUSTED DATA
-------------------
username = aditya
passwordHash = ...
roles = ...
accountStatus = ...
```

So authentication becomes:

```text
Claimed identity + Credential
            ↓
        Validation
            ↓
    Trusted user information
            ↓
     Authentication result
```

---

# 3. Where Can the Source of Truth Live?

Trusted user information can be stored in:

```text
Application memory
Relational database
MongoDB
LDAP
External Identity Provider
Separate authentication service
```

This lecture uses:

```text
MySQL
```

So the high-level architecture is:

```text
Spring Boot Application
        ↓
      MySQL
```

---

# 4. Why the Default User Is Not Enough

Imagine a real learning platform with:

```text
10 users
100 users
10,000 users
```

Every account may need:

```text
Identity
Password
Roles
Account status
```

That information must survive:

```text
Application restart
Server restart
Deployment
Scaling
```

A generated in-memory user cannot provide that persistent user-management model.

Therefore:

> **Real users normally need persistent storage.**

---

# 5. Why Spring Security Does Not Directly Understand Our Database

Every application has a different database design.

Example A:

```text
users
 ├── id
 ├── email
 ├── password
 ├── phone
 ├── created_at
 └── enabled
```

Example B:

```text
employees
 ├── employee_id
 ├── corporate_email
 ├── credential_hash
 ├── department
 └── active
```

Another system may use:

```text
customers
members
accounts
```

Spring Security cannot force every application to use the same:

```text
table names
column names
database technology
schema design
```

So it provides common abstractions.

The most important one in this lecture:

```text
UserDetails
```

---

# 6. UserDetails — The Security Representation of a User

The basic architecture is:

```text
Application-specific User
          ↓
       Adapter
          ↓
Spring Security UserDetails
```

Your application's `User` entity represents what **your application stores**.

`UserDetails` represents what **Spring Security needs for authentication**.

Therefore:

> **Domain User and Spring Security UserDetails are related, but they are not automatically the same object or concept.**

---

# 7. Identity and Credential

Authentication must answer two questions:

```text
Who are you?
      ↓
Identity

Can you prove it?
      ↓
Credential
```

For username/password authentication:

```text
Username / Email
        ↓
Identity

Password
        ↓
Credential
```

---

# 8. Username vs Email

Spring Security methods use historical names such as:

```java
loadUserByUsername(...)
getUsername()
```

But your application's login identifier does not have to literally be a username.

It can be:

```text
Username
Email
Employee ID
Phone number
Customer ID
Any unique account identifier
```

For example:

```java
findByEmail(email);
```

can be used instead of:

```java
findByUsername(username);
```

### Important rule

The login identifier must uniquely identify exactly one account.

---

# 9. Why Login Identifier Must Be Unique

Suppose the database contains:

```text
ID | EMAIL
-----------------------
1  | aditya@gmail.com
2  | aditya@gmail.com
```

Now this request:

```text
login = aditya@gmail.com
```

is ambiguous.

Which account should be authenticated?

Therefore:

> **The identifier used for login should normally have a unique database constraint.**

---

# 10. Minimum Database Information

For basic username/password authentication, the minimum information is roughly:

```text
User
 ├── username
 └── password
```

A minimal table could be:

```text
users
--------------------------------
id        BIGINT PRIMARY KEY
username  VARCHAR UNIQUE
password  VARCHAR
```

A more realistic application may also contain:

```text
enabled
roles
account state
timestamps
profile data
```

depending on requirements.

---

# 11. UserDetails Interface

A simplified view:

```java
public interface UserDetails {

    String getUsername();

    String getPassword();

    Collection<? extends GrantedAuthority>
        getAuthorities();

    boolean isAccountNonExpired();

    boolean isAccountNonLocked();

    boolean isCredentialsNonExpired();

    boolean isEnabled();
}
```

Think:

```text
UserDetails
   ├── Identity
   ├── Credential
   ├── Authorities
   └── Account Status
```

---

# 12. `getUsername()`

Returns the login identifier.

Example:

```text
aditya
```

Even if the application uses:

```text
aditya@gmail.com
```

the method name can still be:

```java
getUsername()
```

because that is the Spring Security interface terminology.

---

# 13. `getPassword()`

Represents the stored encoded password.

Important:

```text
NOT raw password
```

For a secure application, the value returned here should represent the encoded password stored by the application.

---

# 14. `getAuthorities()`

Returns the user's granted authorities.

Examples:

```text
ROLE_USER
ROLE_ADMIN
COURSE_READ
COURSE_WRITE
USER_DELETE
```

These authorities later participate in authorization decisions.

---

# 15. Account Status Methods

`UserDetails` provides:

```java
isAccountNonExpired()
isAccountNonLocked()
isCredentialsNonExpired()
isEnabled()
```

These let Spring Security decide whether the account is eligible for authentication.

---

# 16. Enabled vs Locked

These are different.

## Disabled

```text
The account exists,
but it is not enabled for authentication.
```

## Locked

```text
The account exists,
but authentication is temporarily blocked.
```

Example:

```text
Repeated failed login attempts
        ↓
Account locked
```

---

# 17. Account Expiration

`isAccountNonExpired()` relates to the **account itself**.

Example:

```text
Temporary employee
      ↓
Contract expires
      ↓
Account expires
```

The database row can still exist, but the account is no longer valid for authentication.

---

# 18. Credential Expiration

`isCredentialsNonExpired()` relates to the **credential**.

Example:

```text
Company policy:
Change password every 90 days
        ↓
Password expires
```

So:

```text
Account expired
→ account itself is no longer active

Credential expired
→ account can exist, but the credential is no longer valid
```

---

# 19. You Don't Need Every Status in the Database

Spring Security provides the hooks, but your application does not need a separate database column for every possible state.

For example, if your application does not support account expiration:

```java
isAccountNonExpired()
```

can simply return:

```java
true
```

### Design principle

> **Spring Security provides the abstraction; your application decides which account states it actually needs.**

---

# 20. Initial User Entity

A minimal JPA entity can look like:

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy =
        GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false,
            unique = true)
    private String username;

    @Column(nullable = false)
    private String password;

    private boolean enabled = true;
}
```

The important constraint is:

```java
unique = true
```

because one login identifier should map to one account.

---

# 21. Domain User vs UserDetails

A clean separation is:

```text
Application / Domain Layer

User
 ├── email
 ├── profile
 ├── createdAt
 ├── password
 └── roles
```

while Spring Security works with:

```text
Security Layer

UserDetails
 ├── username
 ├── password
 ├── authorities
 └── account status
```

Architecture:

```text
Domain User
    ↓
Adapter
    ↓
UserDetails
```

Do not assume:

```text
User == UserDetails
```

---

# 22. Introducing Roles

Authentication establishes:

```text
Who is the user?
```

Authorization determines:

```text
What is the user allowed to do?
```

Example:

```text
STUDENT
→ View courses

TEACHER
→ Create courses

ADMIN
→ Delete users
```

---

# 23. Users Can Have Multiple Roles

Example:

```text
Aditya
 ├── ROLE_USER
 └── ROLE_ADMIN

Rohit
 └── ROLE_USER
```

This naturally creates:

```text
User ↔ Role
```

as a many-to-many relationship when both sides can participate in multiple associations.

---

# 24. Database Design for Roles

A common design:

```text
users
--------------------------------
id | username | password | enabled


roles
--------------------------------
id | name


user_roles
--------------------------------
user_id | role_id
```

Example:

```text
users
1 | aditya | ... | true
2 | rohit  | ... | true

roles
1 | ROLE_USER
2 | ROLE_ADMIN

user_roles
1 | 1
1 | 2
2 | 1
```

Meaning:

```text
Aditya → ROLE_USER
Aditya → ROLE_ADMIN

Rohit → ROLE_USER
```

---

# 25. Role Diagram

The supplied Excalidraw diagram shows the relationship visually as:

```text
User
  ↘
   ↘
 User_role
   ↗
  ↗
Role
```

The join table contains:

```text
user_id
role_id
```

This is the bridge between the two entities.

---

# 26. Role Entity

A minimal role entity:

```java
@Entity
@Table(name = "roles")
public class Role {

    @Id
    @GeneratedValue(strategy =
        GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false,
            unique = true)
    private String name;
}
```

The unique role name prevents duplicate role records.

---

# 27. Mapping User → Roles

The lecture uses:

```java
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
```

Conceptually:

```text
User
 ↓
Many Roles
```

through:

```text
user_roles
```

---

# 28. Why Unidirectional Mapping?

The lecture uses:

```text
User → roles
```

and does not currently need:

```text
Role → users
```

because authentication currently needs:

```java
user.getRoles()
```

but does not require:

```java
role.getUsers()
```

### Design principle

> **Add a bidirectional relationship only when the application actually needs navigation in both directions.**

---

# 29. Repository Layer

The application needs to find a user by the login identifier.

Example:

```java
public interface UserRepository
        extends JpaRepository<User, Long> {

    Optional<User> findByUsername(
        String username
    );

    boolean existsByUsername(
        String username
    );
}
```

The two methods serve different purposes:

```text
findByUsername()
→ load user during authentication

existsByUsername()
→ prevent duplicate registration
```

---

# 30. The Missing Bridge

At this stage:

```text
Spring Security
      ↓
     ??????
      ↓
UserRepository
      ↓
Database
```

Spring Security still does not directly know how your repository works.

The missing bridge is:

```text
UserDetailsService
```

---

# 31. UserDetailsService

The job of `UserDetailsService` is:

> **Load the user information required by Spring Security during username/password authentication.**

Architecture:

```text
Spring Security
      ↓
UserDetailsService
      ↓
UserRepository
      ↓
Database
      ↓
User
      ↓
UserDetails
```

---

# 32. Why UserDetailsService Exists

Every application has a different user model.

Spring Security wants one common abstraction.

So:

```text
Application-specific User
        ↓
UserDetailsService
        ↓
Spring Security UserDetails
```

This prevents Spring Security from being tightly coupled to your database structure.

---

# 33. Conceptual Custom UserDetailsService

A typical implementation:

```java
@Service
public class CustomUserDetailsService
        implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(
            UserRepository userRepository) {

        this.userRepository =
            userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(
            String username)
            throws UsernameNotFoundException {

        User user =
            userRepository
                .findByUsername(username)
                .orElseThrow(() ->
                    new UsernameNotFoundException(
                        "User not found"
                    )
                );

        return ...;
    }
}
```

The core responsibility is:

```text
Login identifier
      ↓
Find domain user
      ↓
Adapt domain user
      ↓
Return UserDetails
```

---

# 34. `DaoAuthenticationProvider`

This lecture also introduces:

```text
DaoAuthenticationProvider
```

Its conceptual role is to connect the username/password authentication process with:

```text
UserDetailsService
+
PasswordEncoder
```

Flow:

```text
Login Request
      ↓
AuthenticationManager
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
Authentication Success / Failure
```

### Easy mental model

```text
UserDetailsService
→ "Find the user."

PasswordEncoder
→ "Verify the password."

DaoAuthenticationProvider
→ Coordinates the authentication process.
```

---

# 35. Where Password Verification Happens

Suppose the client submits:

```text
username = aditya
password = secret123
```

Database:

```text
username = aditya
password = $2a$...
```

The application does not do:

```text
secret123 == $2a$...
```

Instead:

```text
Raw password
     +
Stored encoded password
     ↓
PasswordEncoder.matches()
     ↓
true / false
```

---

# 36. Raw Passwords Must Never Be Stored

Bad:

```java
user.setPassword(
    request.getPassword()
);
```

Database:

```text
id | username | password
----------------------------
1  | aditya   | secret123
```

If the database is compromised:

```text
Database breach
      ↓
Raw passwords exposed
      ↓
Users may have reused
the same password elsewhere
      ↓
Damage may spread to other services
```

### Golden rule

> **Never intentionally store a user's raw password.**

---

# 37. Why Password Encryption Is Not the Right Solution

Encryption is reversible:

```text
Plaintext
   ↓
Encryption + Key
   ↓
Ciphertext
   ↓
Decryption + Key
   ↓
Plaintext
```

For password storage, this creates an unnecessary ability to recover the original password.

If attackers obtain both:

```text
Database
+
Decryption key
```

they may recover all passwords.

But login does not require recovering the original password.

It only requires:

```text
"Does the password supplied now match
the password registered earlier?"
```

Therefore:

> **Passwords should use one-way password hashing rather than reversible encryption.**

---

# 38. Password Hashing

Conceptually:

```text
secret123
    ↓
One-way password function
    ↓
Encoded representation
    ↓
Database
```

There should not be a normal operation:

```text
Encoded password
      ↓
Original password
```

---

# 39. Password Verification

At login:

```text
Entered raw password
        +
Stored encoded password
        ↓
Password verification
        ↓
true / false
```

Spring Security exposes this through:

```java
passwordEncoder.matches(
    rawPassword,
    encodedPassword
);
```

---

# 40. Why SHA-256 Alone Is Not Enough

A general-purpose hash such as SHA-256 is intentionally fast.

For password storage, fast hashing is not ideal because an attacker who steals the hash database can perform offline guessing:

```text
guess password
      ↓
calculate hash
      ↓
compare
      ↓
repeat...
```

A fast algorithm allows many guesses quickly.

---

# 41. Adaptive Password Hashing

Password-storage algorithms are deliberately designed to make each guess computationally expensive.

Think:

```text
Fast general-purpose hash
→ cheap guess
→ attacker can test many guesses quickly

Adaptive password hash
→ expensive guess
→ slows large-scale offline guessing
```

The lecture introduces:

```text
BCrypt
```

for this purpose.

---

# 42. BCrypt and PasswordEncoder

Spring Security provides:

```java
BCryptPasswordEncoder
```

through:

```java
PasswordEncoder
```

Example bean:

```java
@Bean
public PasswordEncoder passwordEncoder() {

    return new BCryptPasswordEncoder();
}
```

Architecture:

```text
PasswordEncoder
      ↑
implements
      |
BCryptPasswordEncoder
```

---

# 43. Why Depend on `PasswordEncoder`?

Application code can depend on:

```java
PasswordEncoder
```

instead of hard-coding:

```java
BCryptPasswordEncoder
```

Example:

```java
private final PasswordEncoder passwordEncoder;
```

The concrete implementation is configured separately.

This creates a cleaner abstraction:

```text
Business code
     ↓
PasswordEncoder
     ↓
BCryptPasswordEncoder
```

---

# 44. `encode()`

During registration:

```java
String encoded =
    passwordEncoder.encode(
        "secret123"
    );
```

The database receives an encoded representation similar to:

```text
$2a$10$...
```

The exact value can vary between executions.

---

# 45. `matches()`

During login:

```java
boolean valid =
    passwordEncoder.matches(
        "secret123",
        storedEncodedPassword
    );
```

Correct password:

```text
true
```

Incorrect password:

```java
passwordEncoder.matches(
    "wrongPassword",
    storedEncodedPassword
);
```

Result:

```text
false
```

---

# 46. `encode()` vs `matches()`

Memorize this:

```text
REGISTRATION
    ↓
encode(rawPassword)
    ↓
store encoded value
```

```text
LOGIN
    ↓
matches(rawPassword, storedEncodedValue)
    ↓
true / false
```

### Never do this:

```java
passwordEncoder.encode(rawPassword)
    .equals(storedEncodedPassword);
```

for BCrypt-style salted password storage.

---

# 47. Salt

BCrypt uses a random salt.

Conceptually:

```text
Aditya
secret123 + randomSaltA
       ↓
    BCrypt
       ↓
encodedValueA
```

Another user:

```text
Rohit
secret123 + randomSaltB
       ↓
    BCrypt
       ↓
encodedValueB
```

Therefore:

```text
Same password
      ↓
Different salts
      ↓
Different encoded values
```

---

# 48. Why Salt Matters

Without unique salts, attackers can benefit from precomputed password-hash dictionaries/rainbow-table approaches.

With salts:

```text
same password
     ↓
different salt
     ↓
different output
```

So one precomputed list of password → hash pairs becomes much less useful.

---

# 49. Does BCrypt Need a Separate Salt Column?

With standard BCrypt encoded values:

```text
algorithm information
+
cost parameters
+
salt
+
derived password data
```

are represented inside the encoded value.

Therefore a separate salt column is not required when using the standard BCrypt representation.

---

# 50. How Does `matches()` Work If Every Hash Is Different?

Important interview question.

Suppose:

```java
encode("secret123")
```

produces different values every time.

So how can:

```java
matches("secret123", storedHash)
```

work?

It does **not** do:

```java
encode("secret123")
    .equals(storedHash);
```

Instead:

```text
Stored BCrypt value
       ↓
Read its salt + parameters
       ↓
Use them to verify the raw password
       ↓
true / false
```

So `matches()` uses information contained in the stored BCrypt representation.

---

# 51. Registration Flow

A registration request:

```http
POST /register
Content-Type: application/json

{
  "username": "aditya",
  "password": "secret123"
}
```

The flow:

```text
RegisterRequest
      ↓
Validate input
      ↓
Check duplicate username
      ↓
PasswordEncoder
      ↓
BCrypt
      ↓
Encoded password
      ↓
User Entity
      ↓
UserRepository
      ↓
Database
```

---

# 52. Raw Password Lifetime

The raw password should exist only for the short time needed to process registration.

It should:

```text
NOT be logged
NOT be returned in response
NOT be stored in database
NOT be unnecessarily copied elsewhere
```

Safe flow:

```text
raw password
     ↓
encode immediately
     ↓
encoded password
     ↓
persist
```

---

# 53. Registration DTO

Example:

```java
public class RegisterRequest {

    @NotBlank
    private String username;

    @NotBlank
    @Size(min = 8)
    private String password;
}
```

This demonstrates:

```text
Input validation
```

which is separate from:

```text
Password encoding
```

---

# 54. Validation vs Password Encoding

These solve different problems.

## Validation

```text
Is the submitted password acceptable?
```

Examples:

```text
Not blank
Minimum length
Maximum length
Password policy
```

## Encoding

```text
How do we safely store the password representation?
```

Flow:

```text
Raw password
     ↓
BCrypt
     ↓
Encoded representation
```

### Remember

```text
Validation
≠
Password Encoding
```

---

# 55. Registration Service

A conceptual service:

```java
@Service
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public AuthService(
            UserRepository userRepository,
            PasswordEncoder passwordEncoder) {

        this.userRepository =
            userRepository;

        this.passwordEncoder =
            passwordEncoder;
    }

    public User register(
            RegisterRequest request) {

        if (userRepository.existsByUsername(
                request.getUsername())) {

            throw new IllegalArgumentException(
                "Username is already registered"
            );
        }

        User user = new User();

        user.setUsername(
            request.getUsername()
        );

        user.setPassword(
            passwordEncoder.encode(
                request.getPassword()
            )
        );

        user.setEnabled(true);

        return userRepository.save(user);
    }
}
```

Responsibilities:

```text
1. Validate incoming data
2. Check duplicate identity
3. Encode password
4. Create user entity
5. Save user
```

---

# 56. The Most Important Registration Line

```java
user.setPassword(
    passwordEncoder.encode(
        request.getPassword()
    )
);
```

Input:

```text
secret123
```

Database:

```text
$2a$10$...
```

not:

```text
secret123
```

---

# 57. Complete Authentication Architecture

This is the main diagram to remember:

```text
                       CLIENT
                          │
                  username + password
                          ↓
                 AuthenticationManager
                          ↓
               DaoAuthenticationProvider
                    ┌─────┴─────┐
                    │           │
                    ↓           ↓
           UserDetailsService  PasswordEncoder
                    │           │
                    ↓           ↓
             UserRepository    BCrypt
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
                    └───── password verification
                              ↓
                       Authentication
                              ↓
                       SecurityContext
```

Read it from top to bottom:

```text
Who is the user?
       ↓
Find the user
       ↓
Get security representation
       ↓
Verify password
       ↓
Create authenticated identity
       ↓
Store in SecurityContext
```

---

# 58. Registration vs Login — Critical Difference

## Registration

```text
Raw Password
      ↓
PasswordEncoder.encode()
      ↓
BCrypt
      ↓
Encoded Password
      ↓
Database
```

## Login

```text
Raw Password
      +
Stored Encoded Password
      ↓
PasswordEncoder.matches()
      ↓
true / false
```

### Easy memory trick

```text
New password
→ encode()

Checking password
→ matches()
```

---

# 59. Role Data Flow Into Spring Security

The roles stored in the database eventually become Spring Security authorities.

Conceptually:

```text
User
 ↓
user_roles
 ↓
Role
 ↓
ROLE_USER / ROLE_ADMIN
 ↓
UserDetails
 ↓
GrantedAuthority
 ↓
Authentication
 ↓
Authorization
```

So the database relationship is not isolated from security—it eventually feeds authorization decisions.

---

# 60. Layer Separation

A clean architecture:

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

Spring Security integrates through:

```text
Spring Security
       ↓
UserDetailsService
       ↓
Repository
```

The domain layer has:

```text
User
Role
```

while Spring Security works with:

```text
UserDetails
GrantedAuthority
Authentication
```

This keeps the security framework from dictating your whole database design.

---

# 61. Practical Code Map

## User Entity

```java
@Entity
@Table(name = "users")
class User {

    @Id
    @GeneratedValue(strategy =
        GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false,
            unique = true)
    private String username;

    @Column(nullable = false)
    private String password;

    private boolean enabled = true;
}
```

---

## Role Entity

```java
@Entity
@Table(name = "roles")
class Role {

    @Id
    @GeneratedValue(strategy =
        GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false,
            unique = true)
    private String name;
}
```

---

## User → Roles

```java
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
```

---

## Repository

```java
public interface UserRepository
        extends JpaRepository<User, Long> {

    Optional<User> findByUsername(
        String username
    );

    boolean existsByUsername(
        String username
    );
}
```

---

## PasswordEncoder

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

---

## Encode

```java
String encoded =
    passwordEncoder.encode(
        rawPassword
    );
```

---

## Verify

```java
boolean valid =
    passwordEncoder.matches(
        rawPassword,
        storedEncodedPassword
    );
```

---

# 62. Most Important Comparisons

## User vs UserDetails

```text
User
→ application/domain entity

UserDetails
→ Spring Security security representation
```

## UserRepository vs UserDetailsService

```text
UserRepository
→ retrieves domain data

UserDetailsService
→ adapts/loads user information for Spring Security
```

## UserDetails vs UserDetailsService

```text
UserDetails
→ WHAT the security user looks like

UserDetailsService
→ HOW Spring Security obtains it
```

## PasswordEncoder vs BCryptPasswordEncoder

```text
PasswordEncoder
→ abstraction

BCryptPasswordEncoder
→ concrete implementation
```

## `encode()` vs `matches()`

```text
encode()
→ create stored password representation

matches()
→ verify raw password against stored representation
```

## Password validation vs Password encoding

```text
Validation
→ Is input acceptable?

Encoding
→ How should password representation be stored?
```

## Password hashing vs encryption

```text
Hashing
→ one-way

Encryption
→ reversible with key
```

---

# 63. Common Beginner Mistakes

### Mistake 1

> "I can store the password directly in MySQL."

Wrong.

Never intentionally store raw passwords.

---

### Mistake 2

> "I should encrypt passwords so they are secure."

Not the correct password-storage model.

Authentication normally does not require recovering the original password.

Use password hashing.

---

### Mistake 3

> "SHA-256 is enough for password storage."

Not ideal.

It is a fast general-purpose hash, which helps attackers perform offline guessing quickly.

---

### Mistake 4

> "BCrypt produces the same hash every time for the same password."

Wrong.

BCrypt uses random salting, so encoded values can differ.

---

### Mistake 5

> "During login I should call encode() and compare strings."

Wrong.

Use:

```java
matches(rawPassword, storedEncodedPassword)
```

---

### Mistake 6

> "`User` and `UserDetails` are the same thing."

Not necessarily.

They belong to different layers.

---

### Mistake 7

> "`UserDetailsService` is the repository."

Wrong.

It can use a repository, but it is the Spring Security abstraction that loads security user information.

---

### Mistake 8

> "Username must literally be a username."

No.

The login identifier can be an email, employee ID, phone, etc.

---

### Mistake 9

> "Every possible account status needs a database column."

Not necessarily.

Only model the states your application needs.

---

### Mistake 10

> "Password validation and password encoding are the same."

Wrong.

They solve different problems.

---

### Mistake 11

> "The client can send its own ROLE_ADMIN."

Never trust a client-provided authority.

Authorities should come from a trusted authentication source.

---

# 64. Interview Questions

## Q1. Why is database authentication needed?

Because the default generated user is mainly for development/testing and cannot represent persistent real-world users.

---

## Q2. Why doesn't Spring Security directly use our User entity?

Because applications have different schemas and domain models. Spring Security uses standard abstractions such as `UserDetails`.

---

## Q3. What is UserDetails?

It represents the username, password, authorities, and account-status information Spring Security needs for authentication.

---

## Q4. What is UserDetailsService?

A service abstraction used to load the user's security information by the login identifier.

---

## Q5. What is the role of DaoAuthenticationProvider?

It coordinates database-backed username/password authentication using `UserDetailsService` and password verification.

---

## Q6. UserRepository vs UserDetailsService?

```text
UserRepository
→ data access for domain entity

UserDetailsService
→ security authentication bridge
```

---

## Q7. Why should passwords not be encrypted?

Because encryption is reversible. If the database and decryption key are compromised, the original passwords may be recovered.

---

## Q8. Why use hashing?

Because login only needs to verify whether the presented password matches the registered password.

---

## Q9. Why isn't SHA-256 ideal for passwords?

Because it is fast, allowing attackers to test password guesses rapidly offline.

---

## Q10. What is BCrypt?

An adaptive password-hashing algorithm supported through Spring Security's `BCryptPasswordEncoder`.

---

## Q11. Why are two BCrypt hashes of the same password different?

Because BCrypt uses random salt.

---

## Q12. How does `matches()` work if hashes differ?

It uses the salt and parameters represented in the stored BCrypt value to verify the raw password.

---

## Q13. `encode()` vs `matches()`?

```text
encode()
→ registration/password creation

matches()
→ login/password verification
```

---

## Q14. Why should username be unique?

Because the login identifier must map to exactly one account.

---

## Q15. Why use many-to-many User ↔ Role?

Because:

```text
One user → many roles
One role → many users
```

---

## Q16. Why can the User → Role mapping remain unidirectional?

If the application only needs:

```java
user.getRoles()
```

and does not need:

```java
role.getUsers()
```

there is no need to create reverse navigation.

---

# 65. ⭐ If You Remember Only 20 Things

```text
1. Default generated user is mainly for development/testing.

2. Real applications need persistent users.

3. Authentication needs trusted user information.

4. UserDetails is Spring Security's user-information abstraction.

5. UserDetailsService is the bridge that loads it.

6. UserRepository belongs to the domain/data layer.

7. DaoAuthenticationProvider coordinates username/password authentication.

8. Username/email/login identifier must uniquely identify one account.

9. UserDetails contains identity, credential, authorities, and account state.

10. User entity and UserDetails are separate concepts.

11. Users and roles are commonly many-to-many.

12. Raw passwords must NEVER be stored.

13. Password encryption is not the right storage strategy.

14. Password hashing is one-way.

15. Fast general-purpose hashes such as SHA-256 are not suitable as the primary
    password-storage algorithm.

16. BCrypt is an adaptive password-hashing algorithm.

17. encode() is used when creating/storing a password representation.

18. matches() is used to verify a login password.

19. BCrypt uses a random salt, so the same password can produce different values.

20. Registration:
    validate → duplicate check → encode → save

21. Login:
    load user → verify password → authenticated Authentication

22. Never trust authorities sent directly by the client.
```

---

# 66. 🔥 One-Minute Revision

> **Database authentication connects Spring Security to persistent application users. The client sends an identity claim and credential, but Spring Security needs trusted user information to validate them. `UserDetails` is the security abstraction containing the username, encoded password, authorities, and account-status information. `UserDetailsService` loads that information, usually through `UserRepository`, while `DaoAuthenticationProvider` coordinates the username/password authentication process and `PasswordEncoder` verifies the password. Real applications commonly store users and roles in a database, often with a many-to-many relationship through a join table. Raw passwords must never be stored. Encryption is reversible and therefore is not the correct password-storage model; instead, use an adaptive one-way password-hashing algorithm such as BCrypt. Registration uses `PasswordEncoder.encode()`, while login uses `PasswordEncoder.matches()`. BCrypt uses a random salt, so encoding the same password twice can produce different values. `matches()` verifies the raw password using the salt and parameters represented by the stored encoded value.**

---

# 67. Complete Mental Model

## Registration

```text
Client
  ↓
POST /register
  ↓
RegisterRequest
  ↓
Validate
  ↓
Check duplicate
  ↓
PasswordEncoder.encode()
  ↓
BCrypt
  ↓
Encoded password
  ↓
User entity
  ↓
UserRepository
  ↓
MySQL
```

## Login

```text
Client
  ↓
Username + Password
  ↓
AuthenticationManager
  ↓
DaoAuthenticationProvider
  ↓
UserDetailsService
  ↓
UserRepository
  ↓
MySQL
  ↓
User
  ↓
UserDetails
  ↓
PasswordEncoder.matches()
  ↓
Success / Failure
  ↓
Authenticated Authentication
  ↓
SecurityContext
```

---

# 68. Final Architecture Diagram

```text
                              CLIENT
                                 │
                         username + password
                                 ↓
                     ┌───────────────────────┐
                     │   Spring Security     │
                     └───────────┬───────────┘
                                 ↓
                      AuthenticationManager
                                 ↓
                    DaoAuthenticationProvider
                           ┌─────┴─────┐
                           │           │
                           ↓           ↓
                  UserDetailsService  PasswordEncoder
                           │           │
                           ↓           ↓
                    UserRepository    BCrypt
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
                           └──── password verification
                                      ↓
                               Authentication
                                      ↓
                               SecurityContext
                                      ↓
                                 Authorization
```

---

# 69. What You Should Be Able to Explain Now

Before moving to the next lecture, you should be able to explain these questions **without memorizing the code**:

### Question 1

**How does Spring Security authenticate a user stored in MySQL?**

Expected mental answer:

```text
AuthenticationManager
→ DaoAuthenticationProvider
→ UserDetailsService
→ UserRepository
→ Database
→ UserDetails
→ PasswordEncoder
→ Authentication
```

### Question 2

**How is a password stored securely?**

```text
Raw password
→ PasswordEncoder
→ BCrypt
→ encoded value
→ database
```

### Question 3

**How is a password checked during login?**

```text
Raw password
+
stored encoded password
→ PasswordEncoder.matches()
→ true / false
```

### Question 4

**Why can't Spring Security directly use my User entity?**

Because every application has a different domain/database structure, so Spring Security uses `UserDetails` as a common security abstraction.

---

# Lecture 2 Core Principle

> **The most important architecture to remember is: Spring Security does not need to understand your entire database. Your application provides a bridge through `UserDetailsService`, which loads and adapts trusted database information into `UserDetails`. `DaoAuthenticationProvider` coordinates that information with `PasswordEncoder` to verify credentials and produce an authenticated `Authentication`. For new users, raw passwords are validated, encoded with an adaptive one-way password-hashing algorithm such as BCrypt, and only the encoded representation is stored.**
