# Spring Security — Lecture 4
# JWT Authentication: From Stateful Sessions to Stateless Authentication

> **Main goal of this lecture:** Understand why JWT authentication is needed, how a JWT works internally, how Spring Security issues and validates JWTs, and how a username/password login can be converted into a stateless bearer-token authentication system.

This lecture connects directly to the previous database-authentication lectures.

Before JWT:

```text
Client
  ↓
Username + Password
  ↓
Authentication Filter
  ↓
AuthenticationManager
  ↓
DaoAuthenticationProvider
  ↓
UserDetailsService
  ↓
Database
  ↓
Authenticated Authentication
  ↓
SecurityContext
```

With JWT:

```text
LOGIN REQUEST
Client
  ↓
Username + Password
  ↓
AuthenticationManager
  ↓
DaoAuthenticationProvider
  ↓
Database + PasswordEncoder
  ↓
Authenticated Authentication
  ↓
JwtService
  ↓
JwtEncoder
  ↓
Signed JWT
  ↓
Client


FUTURE PROTECTED REQUEST
Client
  ↓
Authorization: Bearer <JWT>
  ↓
JWT validation
  ↓
Authentication
  ↓
SecurityContext
  ↓
Authorization
  ↓
Controller
```

### The biggest idea

> **JWT does not replace username/password authentication during login. It changes how authentication is carried across future requests.**

---

# 1. The Problem: Authentication on Every HTTP Request

Suppose we have:

```java
@GetMapping("/profile")
public String profile(Authentication authentication) {
    return "Logged in as: "
        + authentication.getName();
}
```

Now imagine:

```http
GET /profile
Authorization: Basic ...
```

The request can be authenticated successfully.

But the next request:

```http
GET /profile
```

contains no authentication credentials.

Why doesn't the server simply remember the previous request?

Because HTTP requests are independent.

```text
Request 1
GET /profile
Authorization: Basic ...
        ↓
Authentication succeeds


Request 2
GET /profile
        ↓
No credentials
        ↓
Authentication cannot be reconstructed automatically
```

### Core principle

> **Authenticating one HTTP request does not automatically authenticate future requests.**

The application needs some authentication state or credential to travel with future requests.

---

# 2. Two Ways to Remember Authentication

There are two broad approaches introduced here:

```text
Stateful Authentication
        VS
Stateless Authentication
```

## Stateful

The server remembers the authenticated user.

Typical mechanism:

```text
HTTP Session
+
JSESSIONID cookie
```

## Stateless

The request itself carries enough authentication material to reconstruct the user's authentication.

Typical mechanism:

```text
Bearer JWT
```

---

# 3. Session-Based Authentication

After successful username/password authentication, Spring Security has an authenticated:

```text
Authentication
```

Conceptually:

```text
Authentication
├── Principal
├── Authorities
└── authenticated = true
```

Instead of discarding this information, the server can associate it with an HTTP session.

Example:

```text
SERVER

Session ABC123
      ↓
SecurityContext
      ↓
Authentication
      ↓
aditya
ROLE_USER
```

The browser receives only the identifier:

```text
ABC123
```

and commonly sends it using:

```http
Cookie: JSESSIONID=ABC123
```

---

# 4. Stateful Session Login Flow

The complete flow:

```text
CLIENT                             SERVER

username + password
       ─────────────────────────────→
                                  Authentication
                                  succeeds
                                      ↓
                                  HttpSession
                                      ↓
                                  SecurityContext
                                      ↓
                                  Authentication
       ←─────────────────────────────
       Set-Cookie:
       JSESSIONID=ABC123
```

Future request:

```http
GET /profile
Cookie: JSESSIONID=ABC123
```

The server does:

```text
ABC123
   ↓
Find HttpSession
   ↓
SecurityContext
   ↓
Authentication
   ↓
User is authenticated
```

---

# 5. What Is Actually Stored in the Session?

The session ID itself is normally only an identifier.

Conceptually:

```text
Browser:

JSESSIONID = ABC123


Server:

ABC123
  ↓
Session
  ↓
SecurityContext
  ↓
Authentication
  ↓
Aditya
ROLE_USER
```

The browser does not normally contain:

```text
username
password
role
```

inside the `JSESSIONID` itself.

---

# 6. Why Session Authentication Is Stateful

Suppose the server stores:

```text
ABC123 → Aditya
XYZ456 → Rohit
PQR789 → Priya
```

The important authentication state exists on the server.

Therefore:

> **Session-based authentication is stateful because the server maintains authentication state between requests.**

---

# 7. SessionCreationPolicy

Spring Security provides:

```java
SessionCreationPolicy
```

to control how the application uses HTTP sessions.

The important policies are:

| Policy | Creates Session? | Uses Existing Session? | Mental Model |
|---|---:|---:|---|
| `ALWAYS` | Yes | Yes | Always session-based |
| `IF_REQUIRED` | When needed | Yes | Create when security requires it |
| `NEVER` | No | Yes | Do not create, but use an existing session |
| `STATELESS` | No for restoring security context | No for restoring it | Authenticate each request independently |

---

# 8. `IF_REQUIRED`

For session-based applications:

```java
.sessionManagement(session ->
    session.sessionCreationPolicy(
        SessionCreationPolicy.IF_REQUIRED
    )
)
```

means:

> Create an HTTP session when Spring Security needs one.

This is a common session-based policy.

---

# 9. `STATELESS`

For JWT authentication:

```java
.sessionManagement(session ->
    session.sessionCreationPolicy(
        SessionCreationPolicy.STATELESS
    )
)
```

means Spring Security should not use the HTTP session to remember authentication between requests.

Instead:

```text
Every protected request
        ↓
Carries authentication material
        ↓
JWT is validated
        ↓
Authentication reconstructed
```

### Very important

```text
STATELESS
≠
"No data is ever stored by the application."
```

It specifically means the application does not rely on the HTTP session to restore the authentication for the next request.

---

# 10. The Problem JWT Tries to Solve

Suppose every request sends:

```http
Cookie: JSESSIONID=ABC123
```

The server needs:

```text
ABC123 → session → SecurityContext → Authentication
```

That means server-side authentication state exists for every active session.

JWT asks:

> Can the client carry enough signed information for the server to reconstruct authentication without maintaining a separate HTTP session entry?

That is the central motivation.

---

# 11. JWT

JWT stands for:

```text
JSON Web Token
```

A JWT can carry claims such as:

```json
{
  "sub": "aditya",
  "roles": ["ROLE_USER"],
  "iat": 1786570000,
  "exp": 1786573600
}
```

The server can validate the token and use trusted claims to reconstruct authentication.

Conceptually:

```text
JWT
 ↓
Claims
 ↓
Authentication
 ↓
SecurityContext
```

The server does not need to restore the user from an HTTP session.

---

# 12. Important Clarification: JWT Is a Token Format

Do not memorize:

```text
JWT = stateless
```

as if they were the same concept.

Correct:

```text
JWT
→ token format

Stateless
→ authentication/session design
```

JWT can be used in different architectures.

The lecture focuses on:

```text
JWT + STATELESS authentication
```

---

# 13. What Problem Exists With Client-Carried Data?

Suppose the JWT payload contains:

```json
{
  "sub": "aditya",
  "roles": ["ROLE_USER"]
}
```

What stops a client from changing it to:

```json
{
  "sub": "aditya",
  "roles": ["ROLE_ADMIN"]
}
```

The server needs a way to determine:

```text
Was this information issued by a trusted party?

Was it modified after issuance?
```

The answer is:

```text
Cryptographic signature
```

---

# 14. Integrity vs Authenticity

Two important security properties:

## Integrity

> The protected data was not modified after it was signed.

## Authenticity

> The data was created by someone possessing the trusted signing key.

JWT signatures help provide these properties.

### Think:

```text
Payload says:
ROLE_USER

Attacker changes:
ROLE_ADMIN

Signature:
does not match anymore
```

So the modified token can be detected.

---

# 15. Why Not Send Plain JSON as the Token?

A naive token could be:

```json
{
  "sub": "aditya",
  "role": "ROLE_USER"
}
```

Raw JSON is not ideal as a compact transport representation because it can contain:

```text
Spaces
Braces
Quotes
Line breaks
Different formatting
```

JWT uses a compact representation based on:

```text
Base64URL
```

This gives a consistent textual representation suitable for transport and signing.

---

# 16. Base64URL Is NOT Encryption

This is one of the most important JWT concepts.

Example:

```json
{"sub":"42"}
```

can become something like:

```text
eyJzdWIiOiI0MiJ9
```

This is:

```text
Encoding
```

not:

```text
Encryption
```

Anyone who has the token can generally decode:

```text
Header
Payload
```

without the signing key.

### Therefore

> **JWT payload data should not be treated as secret simply because it is inside a JWT.**

---

# 17. Base64URL Does Not Hide Sensitive Information

Do not put:

```text
Passwords
Secret API keys
Credit card numbers
Private personal information
```

inside a normal signed JWT payload simply because the payload is Base64URL-encoded.

Remember:

```text
Encoded ≠ encrypted
```

---

# 18. Why Encoding Alone Is Insecure

Imagine:

```text
Token =
Base64URL(payload)
```

Payload:

```json
{
  "sub": "user-42",
  "role": "STUDENT"
}
```

Attacker can:

```text
1. Decode payload
2. Change STUDENT → ADMIN
3. Encode it again
4. Send modified token
```

The server would have no cryptographic evidence that:

```text
the trusted server originally created this modified data.
```

So we need:

```text
Integrity
+
Authenticity
```

---

# 19. Why a Plain Hash Is Not Enough

A beginner might propose:

```text
token = payload + SHA256(payload)
```

or:

```text
Base64URL(payload).SHA256(payload)
```

But this fails.

Why?

Because the attacker can:

```text
1. Modify payload
2. Calculate SHA-256(modified payload)
3. Send new payload + new hash
```

The attacker can calculate the same public hash function.

A plain hash answers:

```text
"Does this hash match this data?"
```

It does not prove:

```text
"Was this data created by my trusted server?"
```

---

# 20. What a Signature Adds

A signature depends on a secret/private key that the attacker does not possess.

So:

```text
Data
+
Trusted signing key
    ↓
Signature
```

Now an attacker can modify the data:

```text
Data → Modified Data
```

but cannot create a valid signature unless they possess the necessary signing key.

Therefore the server can detect unauthorized modifications.

---

# 21. JWT Structure

A signed JWT commonly has:

```text
HEADER.PAYLOAD.SIGNATURE
```

Example shape:

```text
xxxxx.yyyyy.zzzzz
```

These are the three logical parts:

```text
1. Header
2. Payload
3. Signature
```

---

# 22. JWT Header

Example:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

## `alg`

Specifies the cryptographic algorithm used for signing.

Example:

```text
HS256
```

## `typ`

Describes the token type:

```text
JWT
```

---

# 23. JWT Payload

Example:

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

The payload contains **claims**.

---

# 24. Important JWT Claims

| Claim | Meaning |
|---|---|
| `iss` | Issuer — who issued the token |
| `sub` | Subject — who/what the token represents |
| `aud` | Audience — which application/API should accept it |
| `iat` | Issued At — when the token was created |
| `exp` | Expiration Time — when the token expires |
| `jti` | Unique identifier for the token |

Not every claim is mandatory for every application.

The application/security protocol determines which claims should exist and which should be validated.

---

# 25. `iss` — Issuer

Example:

```json
"iss": "coderarmy-api"
```

Means:

```text
Who issued this token?
```

Later the decoder can require:

```text
Expected issuer = coderarmy-api
```

---

# 26. `sub` — Subject

Example:

```json
"sub": "aditya"
```

Means:

```text
Who does this token represent?
```

It is commonly used to identify the principal.

---

# 27. `aud` — Audience

Example:

```json
"aud": "course-api"
```

Answers:

```text
Which service/application is this token intended for?
```

This is useful in systems where multiple services exist.

---

# 28. `iat` — Issued At

Example:

```json
"iat": 1786570000
```

Represents:

```text
When the JWT was issued.
```

---

# 29. `exp` — Expiration

Example:

```json
"exp": 1786573600
```

Represents:

```text
When the JWT should stop being accepted.
```

Expiration limits how long a stolen token can remain valid.

---

# 30. `jti` — JWT ID

`jti` can provide:

```text
A unique identifier for the token.
```

It can be useful in systems that need token tracking/revocation strategies.

---

# 31. Custom Claims

Applications can add custom claims.

Example:

```json
{
  "sub": "aditya",
  "roles": ["ROLE_USER"],
  "department": "engineering"
}
```

But remember:

> **A JWT payload is normally readable by whoever possesses the token.**

So do not put unnecessary secrets inside it.

---

# 32. JWT Signature

The signature protects:

```text
Encoded Header
+
Encoded Payload
```

Conceptually:

```text
Base64URL(header)
        +
"."
        +
Base64URL(payload)
        ↓
Signing Input
        ↓
Cryptographic Signing
        ↓
Signature
```

Final token:

```text
Base64URL(header)
.
Base64URL(payload)
.
Base64URL(signature)
```

### Key idea

> **The signature covers the header and payload together.**

---

# 33. HS256 — Symmetric JWT Signing

HS256 means:

```text
HMAC using SHA-256
```

Conceptually:

```text
Header.Payload
     +
Secret Key
     ↓
HMAC-SHA256
     ↓
Signature
```

The same secret is used for:

```text
Signing
Verification
```

---

# 34. Symmetric Key Model

With HS256:

```text
Issuer
  ↓
HMAC(message, secret)
  ↓
Signature
```

Verifier:

```text
HMAC(message, same secret)
  ↓
Expected signature
```

If it matches:

```text
Token integrity/authenticity check succeeds
```

---

# 35. Advantage of HS256

It is:

```text
Simple
Fast
Convenient
```

especially when:

```text
One trusted application
both issues and validates tokens
```

---

# 36. Limitation of HS256

The same secret is needed for verification.

Imagine:

```text
Authentication Server
Course Service
Payment Service
Admin Service
```

All must know:

```text
SAME SECRET
```

Anyone who has that shared secret can:

```text
Verify tokens
+
Create valid tokens
```

Therefore a compromised verifier could potentially become capable of token issuance.

---

# 37. Asymmetric JWT Signing

Asymmetric cryptography uses:

```text
Private Key
+
Public Key
```

Responsibilities:

```text
Private Key
→ Signs token

Public Key
→ Verifies token
```

Common JWT algorithms mentioned:

```text
RS256
ES256
```

---

# 38. Asymmetric Architecture

Typical design:

```text
Authentication Server
   ↓
Private Key + Public Key

Course Service
   ↓
Public Key only

Payment Service
   ↓
Public Key only

Admin Service
   ↓
Public Key only
```

The services can:

```text
VERIFY tokens
```

but cannot create new valid signatures if they possess only the public key.

---

# 39. Why Asymmetric Signing Is Useful

Suppose:

```text
One authorization/authentication server
+
Many independent resource APIs
```

Then:

```text
Issuer
→ keeps private key

APIs
→ receive public key
```

This means compromising one API does not automatically reveal the signing private key.

---

# 40. Symmetric vs Asymmetric

| Property | HS256 | RS256 / ES256 |
|---|---|---|
| Key model | Shared secret | Private/public pair |
| Signing | Secret key | Private key |
| Verification | Same secret | Public key |
| Can verifier create token? | Yes, if it has secret | No, if it only has public key |
| Simplicity | High | More setup |
| Useful for multiple independent APIs | Less ideal | Very useful |

### Memory trick

```text
Symmetric
→ SAME key for sign + verify

Asymmetric
→ PRIVATE signs
→ PUBLIC verifies
```

---

# 41. JWT Tampering Experiment

One of the best ways to understand JWT security is to try modifying one.

Original:

```text
A.B.C
```

where:

```text
A = Header
B = Payload
C = Signature
```

Suppose original payload:

```json
{
  "sub": "aditya",
  "role": "STUDENT"
}
```

Attacker changes:

```text
STUDENT → ADMIN
```

Now the encoded payload becomes:

```text
B2
```

---

# 42. What Happens to the Signature?

Original:

```text
A.B.C
```

Correct modified token would have to be:

```text
A.B2.C2
```

because the signing input changed.

But the attacker does not know the signing key.

So they may produce:

```text
A.B2.C
```

That is:

```text
Original Header
+
Modified Payload
+
Old Signature
```

---

# 43. Server Detects the Tampering

The server verifies:

```text
Expected Signature
        ≠
Token Signature
```

Therefore:

```text
Invalid JWT
```

### Important conceptual point

> **The signature does not physically prevent a person from editing the JWT. It makes unauthorized modifications detectable.**

This is one of the best ways to understand JWT security.

---

# 44. Login Flow With JWT

JWT does not replace the initial username/password authentication.

The login flow is:

```text
POST /auth/login
{
  "username": "aditya",
  "password": "123456"
}
        ↓
AuthenticationManager
        ↓
DaoAuthenticationProvider
        ↓
CustomUserDetailsService
        ↓
Database
        ↓
PasswordEncoder
        ↓
Authentication successful
        ↓
Generate JWT
        ↓
Return token
```

### Critical idea

> **First authenticate the user normally. Then generate a JWT from the successful Authentication.**

---

# 45. Future Protected Request

After login, the client receives something like:

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9..."
}
```

The client later sends:

```http
GET /profile
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
```

Server flow:

```text
JWT
 ↓
Verify signature
 ↓
Validate claims
 ↓
Identify user
 ↓
Create Authentication
 ↓
SecurityContext
 ↓
Authorization
 ↓
Controller
```

---

# 46. Two Different Authentication Moments

This distinction is extremely important.

## Login

```text
Username + Password
       ↓
Database authentication
       ↓
Authentication
       ↓
JWT generation
```

## Future Request

```text
JWT
       ↓
JWT validation
       ↓
Authentication
       ↓
Authorization
```

So:

```text
Username/password
→ used to obtain the JWT

JWT
→ used for subsequent authentication
```

---

# 47. Spring Security Resource Server Support

The lecture uses:

```xml
<dependency>
    <groupId>
        org.springframework.boot
    </groupId>

    <artifactId>
        spring-boot-starter-oauth2-resource-server
    </artifactId>
</dependency>
```

This support allows Spring Security to authenticate bearer JWTs.

Important:

> Resource Server support can protect an API using JWTs even when the same application is not implementing a full OAuth2 Authorization Server.

---

# 48. JwtEncoder vs JwtDecoder

Spring Security separates JWT creation and validation.

## `JwtEncoder`

```text
CREATE / SIGN JWT
```

## `JwtDecoder`

```text
READ / VERIFY / VALIDATE JWT
```

Mental model:

```text
Login success
     ↓
JwtEncoder
     ↓
JWT
```

Future request:

```text
JWT
     ↓
JwtDecoder
     ↓
Trusted JWT claims
```

---

# 49. JWT Secret Configuration

For HS256, the application needs a strong secret.

The lecture demonstrates generating a Base64-encoded secret using:

```bash
openssl rand -base64 32
```

Configuration example:

```properties
jwt.secret=${JWT_SECRET}
jwt.issuer=coderarmy-api
jwt.expiry=900
```

Where:

```text
900 seconds = 15 minutes
```

---

# 50. Do Not Commit JWT Secrets to Git

A signing secret is sensitive.

Bad:

```properties
jwt.secret=myVerySecretProductionKey
```

committed into:

```text
GitHub
```

Better:

```properties
jwt.secret=${JWT_SECRET}
```

and supply the real secret through:

```text
Environment variable
Secret manager
Deployment configuration
```

### Rule

> **Signing secrets should be treated like passwords/private keys.**

---

# 51. Creating the `SecretKey`

Conceptually:

```java
@Bean
public SecretKey jwtSecretKey(
        @Value("${jwt.secret}") String secret) {

    byte[] decodedKey =
        Base64.getDecoder().decode(secret);

    return new SecretKeySpec(
        decodedKey,
        "HmacSHA256"
    );
}
```

Flow:

```text
Base64 configuration value
        ↓
Base64 decode
        ↓
Raw key bytes
        ↓
SecretKey
```

The same key is then used for:

```text
JwtEncoder
+
JwtDecoder
```

when using HS256 in this example.

---

# 52. JwtEncoder

Example:

```java
@Bean
public JwtEncoder jwtEncoder(
        SecretKey secretKey) {

    return NimbusJwtEncoder
        .withSecretKey(secretKey)
        .algorithm(MacAlgorithm.HS256)
        .build();
}
```

Architecture:

```text
Application
    ↓
JwtEncoder
    ↓
NimbusJwtEncoder
```

The rest of the application depends on the abstraction:

```java
JwtEncoder
```

rather than directly depending on the concrete implementation.

---

# 53. Login DTOs

Request:

```java
public class LoginRequest {
    String username;
    String password;
}
```

Response:

```java
public record LoginResponse(
    String accessToken
) {}
```

Client:

```json
{
  "username": "aditya",
  "password": "123456"
}
```

Server:

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9..."
}
```

---

# 54. JwtService

The lecture creates a service responsible for producing tokens.

Conceptually:

```java
@Service
public class JwtService {

    private final JwtEncoder jwtEncoder;

    private final String issuer;
    private final long expiry;

    public String generateToken(
            Authentication authentication) {

        Instant now =
            Instant.now();

        List<String> authorities =
            authentication
                .getAuthorities()
                .stream()
                .map(
                    GrantedAuthority::getAuthority
                )
                .toList();

        JwtClaimsSet claims =
            JwtClaimsSet.builder()
                .issuer(issuer)
                .issuedAt(now)
                .expiresAt(
                    now.plusSeconds(expiry)
                )
                .subject(
                    authentication.getName()
                )
                .claim(
                    "authorities",
                    authorities
                )
                .build();

        Jwt jwt =
            jwtEncoder.encode(
                JwtEncoderParameters.from(
                    claims
                )
            );

        return jwt.getTokenValue();
    }
}
```

---

# 55. Understanding JwtService

The service receives an already authenticated:

```java
Authentication
```

Conceptually:

```text
Principal
→ aditya

Authorities
→ ROLE_USER

Authenticated
→ true
```

It converts this trusted result into JWT claims.

This is important:

> **JwtService should generate the token from an already authenticated user, not from raw client claims.**

---

# 56. `issuer`

```java
.issuer(issuer)
```

produces:

```json
"iss": "coderarmy-api"
```

It identifies the issuer.

The decoder can later validate:

```text
Expected issuer
=
Token issuer
```

---

# 57. `subject`

```java
.subject(authentication.getName())
```

produces:

```json
"sub": "aditya"
```

This identifies who the token represents.

---

# 58. `issuedAt`

```java
.issuedAt(now)
```

produces:

```text
iat
```

It records when the token was created.

---

# 59. `expiresAt`

```java
.expiresAt(
    now.plusSeconds(expiry)
)
```

produces:

```text
exp
```

After the expiration time:

```text
Token should no longer be accepted.
```

---

# 60. Why Put Authorities in the JWT?

The lecture stores:

```java
.claim(
    "authorities",
    authorities
)
```

Example:

```json
{
  "authorities": [
    "ROLE_USER"
  ]
}
```

Now a future protected request can derive:

```text
Who is this?
        ↓
sub

What can they do?
        ↓
authorities
```

without necessarily doing:

```text
JWT
 ↓
username
 ↓
database
 ↓
UserDetails
```

on every request.

That is one of the key ideas behind this stateless design.

---

# 61. JWT Can Reconstruct Authentication

If the trusted token contains:

```json
{
  "sub": "aditya",
  "authorities": [
    "ROLE_USER"
  ]
}
```

the application can conceptually reconstruct:

```text
Authentication
├── principal = aditya
├── authorities = ROLE_USER
└── authenticated = true
```

This is what makes the example stateless.

---

# 62. Login Controller

Example:

```java
@RestController
@RequestMapping("/auth")
public class AuthController {

    private final AuthenticationManager
        authenticationManager;

    private final JwtService jwtService;

    @PostMapping("/login")
    public LoginResponse login(
            @RequestBody LoginRequest request) {

        Authentication authenticationRequest =
            UsernamePasswordAuthenticationToken
                .unauthenticated(
                    request.username(),
                    request.password()
                );

        Authentication authentication =
            authenticationManager.authenticate(
                authenticationRequest
            );

        String token =
            jwtService.generateToken(
                authentication
            );

        return new LoginResponse(token);
    }
}
```

---

# 63. Login Controller Mental Model

```text
Request DTO
   ↓
Create unauthenticated Authentication
   ↓
AuthenticationManager
   ↓
Database authentication
   ↓
Authenticated Authentication
   ↓
JwtService
   ↓
JWT
   ↓
LoginResponse
```

The username/password authentication mechanism is therefore still the same mechanism learned earlier.

JWT generation happens **after** authentication succeeds.

---

# 64. JWT Creation Flow

```text
username + password
        ↓
UsernamePasswordAuthenticationToken
        ↓
AuthenticationManager
        ↓
DaoAuthenticationProvider
        ↓
CustomUserDetailsService
        ↓
Database
        ↓
PasswordEncoder
        ↓
Authentication
authenticated = true
        ↓
JwtService
        ↓
JwtEncoder
        ↓
JWT
```

---

# 65. JWT Validation Is the Other Half

Creating a JWT is only half of the system.

When the client sends:

```http
Authorization: Bearer <JWT>
```

the server must not simply trust it.

It must:

```text
1. Read token
2. Verify signature
3. Validate claims
4. Treat result as trusted only after validation
5. Create Authentication
6. Continue with authorization
```

---

# 66. JwtDecoder

Example:

```java
@Bean
public JwtDecoder jwtDecoder(
        SecretKey secretKey,
        @Value("${jwt.issuer}")
        String issuer) {

    NimbusJwtDecoder decoder =
        NimbusJwtDecoder
            .withSecretKey(secretKey)
            .macAlgorithm(
                MacAlgorithm.HS256
            )
            .build();

    decoder.setJwtValidator(
        JwtValidators
            .createDefaultWithIssuer(
                issuer
            )
    );

    return decoder;
}
```

---

# 67. What Does the Decoder Check?

Conceptually:

```text
Is this structurally a valid JWT?
        ↓
Can the signature be verified?
        ↓
Was the correct trusted key used?
        ↓
Are claims valid?
        ↓
Is issuer correct?
        ↓
Is token currently valid?
        ↓
Only then → trusted JWT
```

The key idea:

> **Do not treat claims as trusted before cryptographic verification and claim validation succeed.**

---

# 68. Why We Need `JwtAuthenticationConverter`

JWT claims are just token data.

Spring Security needs:

```text
GrantedAuthority
```

objects.

The lecture's token stores:

```json
{
  "authorities": [
    "ROLE_USER"
  ]
}
```

But Spring Security Resource Server has standard authority mapping conventions, commonly related to OAuth scopes.

Therefore the application explicitly tells Spring:

```text
Read the "authorities" claim.
Do not add another prefix.
```

---

# 69. JWT Authority Mapping

Example:

```java
@Bean
public JwtAuthenticationConverter
jwtAuthenticationConverter() {

    JwtGrantedAuthoritiesConverter
        authoritiesConverter =
            new JwtGrantedAuthoritiesConverter();

    authoritiesConverter
        .setAuthoritiesClaimName(
            "authorities"
        );

    authoritiesConverter
        .setAuthorityPrefix("");

    JwtAuthenticationConverter
        authenticationConverter =
            new JwtAuthenticationConverter();

    authenticationConverter
        .setJwtGrantedAuthoritiesConverter(
            authoritiesConverter
        );

    return authenticationConverter;
}
```

---

# 70. Why Empty Authority Prefix?

Token contains:

```text
ROLE_USER
```

If Spring automatically added another prefix:

```text
SCOPE_ROLE_USER
```

or another configured prefix, authorization rules would not match as intended.

We want:

```text
JWT
  ↓
ROLE_USER
  ↓
GrantedAuthority("ROLE_USER")
```

Then existing rules work:

```java
.hasRole("USER")
.hasRole("ADMIN")
```

---

# 71. Final JWT SecurityFilterChain

The lecture's JWT configuration is conceptually:

```java
@Bean
public SecurityFilterChain securityFilterChain(
        HttpSecurity http,
        JwtAuthenticationConverter
            jwtAuthenticationConverter
) throws Exception {

    http
        .csrf(csrf ->
            csrf.disable()
        )

        .sessionManagement(session ->
            session.sessionCreationPolicy(
                SessionCreationPolicy.STATELESS
            )
        )

        .authorizeHttpRequests(auth ->
            auth
                .requestMatchers(
                    "/auth/login",
                    "/register"
                )
                .permitAll()

                .requestMatchers(
                    "/admin/**"
                )
                .hasRole("ADMIN")

                .anyRequest()
                .authenticated()
        )

        .oauth2ResourceServer(oauth2 ->
            oauth2.jwt(jwt ->
                jwt.jwtAuthenticationConverter(
                    jwtAuthenticationConverter
                )
            )
        );

    return http.build();
}
```

---

# 72. Understanding Each Security Configuration

## CSRF

```java
.csrf(csrf -> csrf.disable())
```

The classroom API demo disables CSRF.

This is a demo choice and should not be treated as a universal production rule.

---

## STATELESS

```java
.sessionManagement(session ->
    session.sessionCreationPolicy(
        SessionCreationPolicy.STATELESS
    )
)
```

Means:

```text
Do not rely on HTTP session
to restore authentication.
```

---

## Public endpoints

```java
.requestMatchers(
    "/auth/login",
    "/register"
).permitAll()
```

Users need to reach these endpoints before they have a JWT.

---

## Admin endpoint

```java
.requestMatchers(
    "/admin/**"
).hasRole("ADMIN")
```

Requires:

```text
ROLE_ADMIN
```

---

## Other endpoints

```java
.anyRequest()
    .authenticated()
```

Requires an authenticated principal.

---

## Resource Server JWT Support

```java
.oauth2ResourceServer(...)
```

tells Spring Security to use bearer-token JWT authentication.

---

# 73. Why `oauth2ResourceServer().jwt()` for a Custom JWT?

This can initially look confusing.

You may think:

```text
"I am not implementing a complete OAuth2 Authorization Server.
Why am I using Resource Server support?"
```

Because Spring Security packages:

```text
Bearer token authentication
+
JWT validation
```

inside Resource Server support.

The application can issue its own custom JWT while using Spring Security's Resource Server machinery to validate bearer JWTs on protected requests.

### Mental model

```text
My application issues JWT
        +
Spring Security Resource Server validates JWT
```

---

# 74. Two Authentication Providers, Two Purposes

After adding JWT, the application has two authentication mechanisms.

## Username/Password

```text
DaoAuthenticationProvider
```

Understands:

```text
Username + Password
```

Its purpose:

```text
Authenticate login credentials
```

---

## JWT

```text
JwtAuthenticationProvider
```

Understands:

```text
JWT
```

Its purpose:

```text
Authenticate future requests carrying bearer JWTs
```

---

# 75. Both Ultimately Produce Authentication

This is a beautiful Spring Security design principle.

Whether the user proves identity using:

```text
Username + Password
```

or:

```text
JWT
```

the result is still:

```text
Authentication
```

This keeps the rest of Spring Security consistent.

Conceptually:

```text
Username/Password
        ↓
DaoAuthenticationProvider
        ↓
Authentication


JWT
 ↓
JwtAuthenticationProvider
 ↓
Authentication
```

Then:

```text
Authentication
 ↓
SecurityContext
 ↓
Authorization
```

---

# 76. Complete End-to-End Flow

## First Request — Login

```text
POST /auth/login
username + password
        ↓
AuthenticationManager
        ↓
DaoAuthenticationProvider
        ↓
CustomUserDetailsService
        ↓
Database
        ↓
PasswordEncoder
        ↓
Authentication
        ↓
JwtService
        ↓
JwtEncoder
        ↓
Signed JWT
        ↓
Return token to client
```

---

## Future Request — Protected API

```text
GET /profile
Authorization: Bearer <JWT>
        ↓
Bearer JWT authentication
        ↓
JwtDecoder
        ↓
Verify signature
        ↓
Validate claims
        ↓
JwtAuthenticationConverter
        ↓
Create Authentication
        ↓
SecurityContext
        ↓
Authorization
        ↓
Controller
```

This is the **complete JWT mental model**.

---

# 77. Session Authentication vs JWT Authentication

| Session-Based | JWT-Based |
|---|---|
| Server stores authentication state | Authentication is reconstructed from token |
| Client commonly sends `JSESSIONID` | Client sends bearer JWT |
| Session ID points to server-side state | Token carries signed claims |
| Commonly stateful | Can be stateless |
| Server restores SecurityContext from session | Server validates token and creates Authentication per request |
| Session invalidation can directly remove server-side state | Token lifetime/validation determine acceptance |

### Easy memory trick

```text
Session:
"Here is an ID. Server, remember me."

JWT:
"Here is my signed authentication material.
Verify it and reconstruct me."
```

---

# 78. JWT Is Readable but Protected Against Unauthorized Modification

This distinction is worth repeating:

```text
Header + Payload
→ readable

Signature
→ protects integrity/authenticity
```

Therefore:

```text
JWT is NOT:
encrypted user data
```

It is primarily:

```text
signed claims
```

unless an additional encryption mechanism is explicitly used.

---

# 79. JWT vs Encryption

Very important:

```text
JWT
→ commonly signed

Encryption
→ hides/enciphers content
```

A normal signed JWT does not hide:

```text
sub
roles
iss
aud
iat
exp
```

Anyone holding the token can normally decode them.

---

# 80. JWT Security Properties

A signed JWT provides useful guarantees such as:

```text
Integrity
Authenticity
Expiration enforcement
Issuer validation
Audience validation
```

provided the application actually performs the relevant validation checks.

It does NOT automatically provide:

```text
Confidentiality
```

because the payload is normally readable.

---

# 81. What If Someone Steals a Valid JWT?

A bearer JWT is a credential.

If someone possesses a valid token:

```text
Bearer
   ↓
Possession may be sufficient to present it
```

Therefore JWTs should be protected carefully.

Important controls include:

```text
HTTPS
Short expiration
Secure storage strategy
Avoid logging tokens
Strong signing keys
Proper issuer/audience validation
Appropriate authorities/scopes
```

---

# 82. Why Expiration Matters

Suppose:

```json
{
  "sub": "aditya",
  "exp": 1786573600
}
```

If a token is stolen, it should not remain valid forever.

Therefore:

```text
Short-lived access token
        ↓
Reduced lifetime of stolen credential
```

The lecture example uses:

```text
900 seconds
=
15 minutes
```

---

# 83. Why Authorities in JWT Are Powerful

Suppose JWT says:

```json
{
  "sub": "aditya",
  "authorities": [
    "ROLE_ADMIN"
  ]
}
```

Then authorization may allow:

```java
.hasRole("ADMIN")
```

without loading the user's roles from the database during each protected request.

### Trade-off

This also means:

> **Changes to the user's roles may not take effect for already-issued tokens until the token expires or another revocation strategy is used.**

This is an architectural consequence of carrying authorization data inside the token.

---

# 84. Statelessness vs Immediate Revocation

Session:

```text
Session exists on server
        ↓
Delete session
        ↓
Authentication immediately invalidated
```

JWT:

```text
Token issued
        ↓
Token remains valid until
validation says it is no longer valid
```

So stateless tokens often trade some easy server-side session invalidation for reduced dependence on server-side authentication session state.

---

# 85. Important Mental Model: JWT Does Not Eliminate Authentication

It changes the authentication evidence used after login.

### Session model

```text
Login
  ↓
Authentication
  ↓
Server session
  ↓
Future request uses JSESSIONID
```

### JWT model

```text
Login
  ↓
Authentication
  ↓
JWT generated
  ↓
Future request uses Bearer JWT
  ↓
JWT validated
  ↓
Authentication reconstructed
```

---

# 86. Important Mental Model: Token Creation and Token Validation Are Different Jobs

## Token Creation

```text
Authentication
     ↓
JwtService
     ↓
JwtEncoder
     ↓
Signed JWT
```

## Token Validation

```text
Incoming JWT
     ↓
JwtDecoder
     ↓
Signature verification
     ↓
Claims validation
     ↓
Trusted JWT
     ↓
Authentication
```

Do not mix these responsibilities.

---

# 87. Common Beginner Mistakes

### Mistake 1

> "JWT encrypts the payload."

Wrong.

Normal signed JWT payloads are encoded, not encrypted.

---

### Mistake 2

> "Base64URL is security."

No.

It is encoding.

---

### Mistake 3

> "If I change the role inside a JWT, the server will see the new role."

Only if the server fails to validate the signature.

A correctly signed JWT will reject unauthorized modifications.

---

### Mistake 4

> "SHA-256 is enough as a JWT signature."

A plain public hash is not a signature because an attacker can recompute it.

JWT signing requires a secret/private-key-based cryptographic mechanism.

---

### Mistake 5

> "JWT replaces username/password login."

Not in this architecture.

Username/password still authenticate the login request.

JWT is then issued for future requests.

---

### Mistake 6

> "JwtEncoder authenticates the user."

No.

`JwtEncoder` creates/signs a token from already trusted information.

---

### Mistake 7

> "JwtDecoder creates the JWT."

No.

```text
JwtEncoder
→ create/sign

JwtDecoder
→ decode/verify/validate
```

---

### Mistake 8

> "Every JWT verifier should have the private key."

Not with asymmetric signing.

```text
Private key
→ sign

Public key
→ verify
```

---

### Mistake 9

> "HS256 is always better because it is simpler."

It depends on architecture.

HS256 is convenient for a trusted single issuer/verifier setup.

Asymmetric signing is useful when many independent APIs need verification without receiving the private signing key.

---

### Mistake 10

> "STATELESS means my application cannot store anything."

Wrong.

It specifically means authentication is not restored from the HTTP session between requests.

---

### Mistake 11

> "JWT means no authentication processing happens on future requests."

Wrong.

Every future request still needs token validation and authentication reconstruction.

---

### Mistake 12

> "JWT roles are automatically understood by Spring Security."

Not necessarily.

If you use a custom claim such as:

```json
"authorities": ["ROLE_USER"]
```

you may need a `JwtAuthenticationConverter`.

---

### Mistake 13

> "The client can choose its own role when requesting a JWT."

Dangerous.

The JWT must be generated from trusted server-side authentication/authority data.

---

# 88. Interview Questions

## Q1. Why do we need JWT if session authentication already works?

JWT can carry signed authentication information with each request, allowing the application to authenticate requests without relying on server-side HTTP session state.

---

## Q2. What is JWT?

JSON Web Token, a compact token format commonly used to carry signed claims.

---

## Q3. What are the three parts of a signed JWT?

```text
Header
Payload
Signature
```

represented as:

```text
HEADER.PAYLOAD.SIGNATURE
```

---

## Q4. Is JWT encrypted?

A normal signed JWT is not encrypted. Its header and payload are Base64URL-encoded and readable.

---

## Q5. What does the JWT signature provide?

It helps establish integrity and authenticity of the signed header/payload.

---

## Q6. Why is a plain SHA-256 hash not enough?

Because an attacker can modify the payload and calculate a new SHA-256 hash. A signature depends on a secret/private key the attacker should not possess.

---

## Q7. What is HS256?

HMAC using SHA-256 with a shared secret.

---

## Q8. Difference between symmetric and asymmetric signing?

```text
Symmetric:
same shared secret signs and verifies.

Asymmetric:
private key signs,
public key verifies.
```

---

## Q9. Why use asymmetric signing in microservices?

Multiple services can verify tokens with the public key without receiving the private signing key.

---

## Q10. What is `JwtEncoder`?

Spring Security abstraction used to create/sign JWTs.

---

## Q11. What is `JwtDecoder`?

Spring Security abstraction used to decode, verify, and validate incoming JWTs.

---

## Q12. Why use `oauth2ResourceServer().jwt()`?

Spring Security's Resource Server support provides bearer-token JWT authentication/validation for protected APIs.

---

## Q13. What is `SessionCreationPolicy.STATELESS`?

It tells Spring Security not to rely on HTTP session state for restoring authentication between requests.

---

## Q14. Why put authorities in a JWT?

To allow future requests to reconstruct the user's authorities from the trusted token without necessarily querying the database for roles on every request.

---

## Q15. What happens if a JWT payload is modified?

The payload changes, so the signing input changes. Unless the attacker can create a new valid signature with the trusted signing key, signature verification fails.

---

## Q16. Does JWT prevent editing?

No.

People can edit the encoded payload.

The signature makes unauthorized editing detectable.

---

## Q17. Is JWT completely stateless?

The lecture's design uses JWT with:

```text
SessionCreationPolicy.STATELESS
```

So it does not rely on HTTP-session authentication state. The overall application can still store users, roles, logs, business data, etc.

---

## Q18. Why validate `iss` and `exp`?

Because the application needs to ensure:

```text
Correct issuer
+
Token not expired
```

before trusting the token's claims.

---

## Q19. Why use a short JWT lifetime?

To reduce the usable lifetime of a stolen access token.

---

## Q20. Why is `JwtAuthenticationConverter` needed in this example?

Because the application stores authorities in a custom `authorities` claim and wants Spring Security to convert those values into `GrantedAuthority` without adding another prefix.

---

# 89. ⭐ If You Remember Only 30 Things

```text
1. HTTP does not remember authentication automatically.

2. Session authentication stores authentication state on the server.

3. JSESSIONID usually identifies server-side session state.

4. JWT can carry signed claims to represent authentication.

5. JWT is a token format.

6. Stateless means authentication is not restored from HTTP session state.

7. Base64URL is encoding, not encryption.

8. JWT payloads are normally readable.

9. Do not put passwords/secrets into normal JWT payloads.

10. JWT commonly has:
    HEADER.PAYLOAD.SIGNATURE

11. Header contains metadata such as the signing algorithm.

12. Payload contains claims.

13. Signature protects the encoded header + payload.

14. Signature gives integrity/authenticity properties.

15. Plain SHA-256 is not a digital signature.

16. HS256 uses one shared secret for sign + verify.

17. RS256/ES256 use private/public keys.

18. Private key signs.

19. Public key verifies.

20. JwtEncoder creates/signs tokens.

21. JwtDecoder verifies/validates incoming tokens.

22. Username/password still authenticate the login request.

23. JWT is generated only after login authentication succeeds.

24. Future requests send:
    Authorization: Bearer <JWT>

25. Future requests must validate the JWT before trusting its claims.

26. `sub` commonly identifies the token subject.

27. `iss` identifies issuer.

28. `aud` identifies intended audience.

29. `iat` identifies issue time.

30. `exp` identifies expiration time.

31. Authorities can be stored in custom JWT claims.

32. `JwtAuthenticationConverter` maps JWT claims to GrantedAuthority.

33. `STATELESS` means no HTTP-session authentication restoration.

34. Access-token expiration limits the lifetime of a stolen token.

35. JWT can reduce dependence on per-user server-side HTTP sessions,
    but it does not eliminate all server state.

36. Different authentication mechanisms can still produce the same
    Spring Security Authentication abstraction.
```

---

# 90. One-Minute Revision

> **JWT is a compact signed-token format commonly used to carry authentication information between client and server. Session authentication stores the authenticated SecurityContext on the server and sends a session ID such as JSESSIONID to the client. JWT-based stateless authentication instead sends a bearer token with each protected request, allowing the server to validate the token and reconstruct Authentication without relying on HTTP session state. A signed JWT normally has three parts—header, payload, and signature. The header and payload are Base64URL-encoded, not encrypted, so their claims are readable. The signature provides integrity/authenticity by proving that the protected header and payload were signed using the trusted signing key. HS256 uses one shared secret for signing and verification, while asymmetric algorithms such as RS256/ES256 use a private key for signing and a public key for verification. During login, Spring Security still performs normal username/password authentication using AuthenticationManager → DaoAuthenticationProvider → UserDetailsService → database → PasswordEncoder. After successful authentication, JwtService uses JwtEncoder to create the JWT. On future requests, a bearer JWT is processed by Resource Server support; JwtDecoder verifies the signature and validates claims such as issuer and expiration, then Spring Security reconstructs Authentication and performs authorization. Thus JWT does not remove authentication—it changes the evidence used to authenticate subsequent requests.**

---

# 91. Final Concept Map

```text
                         JWT AUTHENTICATION
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
             LOGIN FLOW                 FUTURE REQUEST
                 │                             │
      username + password               Bearer JWT
                 │                             │
                 ↓                             ↓
      AuthenticationManager              JwtDecoder
                 │                             │
                 ↓                        Verify signature
       DaoAuthenticationProvider               │
            ┌────┴────┐                   Validate claims
            │         │                         │
            ↓         ↓                         ↓
 UserDetailsService  PasswordEncoder      Trusted JWT
            │         │                         │
            ↓         ↓                         ↓
        Database     BCrypt              JwtAuthenticationConverter
            │                                      │
            ↓                                      ↓
        UserDetails                          Authentication
            │                                      │
            └──────────────┐                       ↓
                           ↓                SecurityContext
                      Authentication               │
                           │                      ↓
                           ↓                Authorization
                       JwtService                   │
                           │                      ↓
                       JwtEncoder              Controller
                           │
                           ↓
                         JWT
                           │
                           ↓
                     Return to Client
```

---

# 92. Stateful vs JWT — Draw This From Memory

## STATEFUL

```text
Client
  ↓
Login
  ↓
Authentication
  ↓
HTTP Session
  ↓
SecurityContext
  ↓
Authentication

Later:
Client
  ↓
JSESSIONID
  ↓
Session
  ↓
Authentication
```

## STATELESS JWT

```text
Client
  ↓
Login
  ↓
Authentication
  ↓
JwtEncoder
  ↓
JWT
  ↓
Client

Later:
Client
  ↓
Bearer JWT
  ↓
JwtDecoder
  ↓
Validate
  ↓
Authentication
  ↓
SecurityContext
  ↓
Authorization
```

### One-line difference

```text
Session:
"Server, remember me using this session ID."

JWT:
"Here is my signed authentication material;
verify it for this request."
```

---

# 93. Most Important JWT Diagram

```text
                 JWT
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
      HEADER    PAYLOAD   SIGNATURE
        │         │           │
        │         │           │
      alg/typ   claims     protects
                  │         header+payload
                  │
        ┌─────────┼─────────────┐
        ↓         ↓             ↓
       iss       sub            aud
       iat       exp            jti
                  │
                  ↓
             authorities
```

And:

```text
HEADER + "." + PAYLOAD
          ↓
    Signing Input
          ↓
   Cryptographic Sign
          ↓
      Signature
```

Final:

```text
Base64URL(header)
.
Base64URL(payload)
.
Base64URL(signature)
```

---

# 94. Security Checklist for JWT

When building JWT authentication, mentally check:

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

---

# 95. What This Lecture Adds to Earlier Lectures

### Lecture 1

You learned:

```text
Authentication
Authorization
SecurityContext
Stateful vs Stateless
JWT basics
```

### Lecture 2

You learned:

```text
Database User
UserDetails
UserDetailsService
PasswordEncoder
BCrypt
User + Role
```

### Lecture 3

You learned:

```text
Filters
AuthenticationManager
ProviderManager
DaoAuthenticationProvider
Complete username/password authentication flow
```

### Lecture 4 — This Lecture

Now everything connects:

```text
Username + Password
        ↓
Database Authentication
        ↓
Authenticated Authentication
        ↓
JWT Generation
        ↓
Bearer JWT
        ↓
JWT Validation
        ↓
Authentication Reconstruction
        ↓
Authorization
```

So the progression is:

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

# 96. What You Should Be Able to Explain Before Moving On

You should be able to answer these in your own words:

### 1. Why do we need JWT?

Because the server needs a way to recognize and authenticate later requests without relying on a separate HTTP session entry for every client.

### 2. What is inside a JWT?

```text
Header
Payload
Signature
```

### 3. Is the payload encrypted?

No. Normally it is Base64URL-encoded and readable.

### 4. What prevents role tampering?

The cryptographic signature.

### 5. How is the JWT generated?

```text
Authenticated Authentication
        ↓
JwtService
        ↓
JwtEncoder
        ↓
Signed JWT
```

### 6. How is a future request authenticated?

```text
Bearer JWT
        ↓
JwtDecoder
        ↓
Signature + claim validation
        ↓
Authentication
```

### 7. Why use STATELESS?

To prevent Spring Security from relying on HTTP session authentication state between requests.

### 8. What is the difference between HS256 and RS256?

```text
HS256
→ shared secret

RS256
→ private key signs
→ public key verifies
```

### 9. Why is `JwtAuthenticationConverter` needed here?

Because the application uses a custom `authorities` claim and wants those values converted into Spring Security `GrantedAuthority` objects without changing the role strings.

### 10. What is the relationship between JWT and Spring Security?

JWT carries the authentication evidence; Spring Security validates it, converts the trusted claims into `Authentication`, stores it in the security context, and applies authorization rules.

---

# Lecture 4 Core Principle

> **JWT authentication is not a replacement for authentication itself. It is a way to carry trusted authentication information between requests without relying on an HTTP session. The initial username/password login still uses the database authentication pipeline. After successful authentication, a `JwtEncoder` creates a signed token containing trusted claims such as subject, issuer, expiration, and authorities. Future requests send that token as a bearer credential; Spring Security validates it with a `JwtDecoder`, converts trusted claims into an `Authentication`, places it in the `SecurityContext`, and then performs authorization. The central security property is not that JWT data cannot be edited—it is that unauthorized edits make the signature invalid and therefore detectable.**
