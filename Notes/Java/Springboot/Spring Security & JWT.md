## Spring Security
### 1. Core Security Concepts
#### Authentication
`Who are you?` 
Example:
- Login with email/password
- JWT validation
#### Authorization
`what you are allowed to do?`
Example:
- admin-only endpoint
- workspace permissions

### 2. Why Spring Security uses Filters
Security is a `cross-cutting concern` -> it affects the entire application.
cross cutting concerns should be centralized.
Instead of putting check inside the controller. Spring security centralizes the security using filters.

Filters execute `BEFORE request reaches application logic` meaning unauthorized requests are blocked early
This improves: 
- security
- performance
- maintainability

### 3. Request flow
Without Security
```
Request
   ↓
DispatcherServlet
   ↓
Controller
   ↓
Response
```

With Spring Security
```
Request
   ↓
Security Filters
   ↓
DispatcherServlet
   ↓
Controller
```

### 4. What is a Filter?
A filter:
- intercepts requests
- can inspect headers/body
- can block requests
- can modify request/response
- runs before controller

### 5. Servlet Filter Chain
Spring security is built on `Java Servlet Filters`
Request flow internally
```
Client
   ↓
Tomcat
   ↓
Filter Chain
   ↓
DispatcherServlet
   ↓
Controller
```

Spring Security adds multiple filters:
- authentication filters
- authorization filters
- CSRF filters
- session filters
- JWT filters

### 6. Why Filters Are Better Than Controller Checks
- centralized security
- avoids duplicate code
- prevents forgotten checks
- easier maintenance
- consistent behavior
- blocks unauthorized requests early

### 7. Stateful and Stateless Authentication
Traditional session based auth(Stateful):
```
Login
   ↓
Server creates session
   ↓
Client sends session ID
```
server stores the authentication state.

Stateless Authentication
JWT-based auth:
```
Login
   ↓
Server creates JWT
   ↓
Client stores token
   ↓
Client sends token every request
```
Server stores No session state.
Authentication data lives inside the token

### 8. Why JWT Systems Are Stateless
Server does not remember clients between requests.
Each request contains all authentication information inside `JWT`
Benefits: 
- scalable
- no shared session store
- works well in distributed systems
## JWT
JWT -> `JSON web token`
It is signed structured data
It has 3 parts `xxxxxx.yyyyy.zzzzzz`
#### Header contains 
- algorithm
- token type
example: 
```
{
  "alg": "HS256",
  "typ": "JWT"
}
```

#### Payload
It contains claims/data
```
{
  "sub": "user@email.com",
  "role": "ADMIN",
  "exp": 123456789
}
```

Registered Claims -- Standard claims 
```
sub -- subject
iss -- issuer
iat -- issued at
exp -- expiry
aud -- audience
```
#### Signature
`It prevents tampering`
Generated using:
- header
- payload
- secret key

#### Access Token 
short lived and sent on every request.

Refresh Token 
long lived and gets new access token without force login 
highly protected.

```Flow
Request
   ↓
JWT Filter
   ↓
Extract Token
   ↓
Validate Signature
   ↓
Check Expiration
   ↓
Extract Subject
   ↓
Load UserDetails
   ↓
Create Authentication
   ↓
SecurityContext
   ↓
Controller
```


### Important JWT Security concepts
JWT payload is not encrypted
payload is only `Base64 encoded`. Anyone can decode it
Never store:
- passwords
- secrets
- sensitive information
inside payload
#### JWT Signature Guarantees
JWT signature guarantees:
- token integrity
- token authenticity
Meaning:
- payload not modified
- token issued by trusted server

signature can be generated using HMAC
signature = HMAC(secret key, header + payload)

HMAC can be implemented using symmetric and asymmetric cryptographic ways
1. HS256
	1. its one of symmetric cryptographic  
	2. We use a  shared secret key 
	3. Both JWT generation(signing) and JWT validator uses the same key
	4. Pros
		1. Fast
		2. simple
	5. Cons
		1. secret shared everywhere
2. RSA (RS256)
	1. It is asymmetric cryptographic method
	2. here we have two keys 
		1. public key  -- used for validation
		2. private key -- used for generation(signing)
	3. Pros
		1. Safer distribution
		2. more scalable
	4. Cons
		1. Complex

[[JWT RISKS - LOGOUT]]
# JWT Token Theft Defenses — Quick Notes

## Core Reality

JWT is a **Bearer Token**:

```
Whoever possesses the token can use it.
```

Security goal:

```
Reduce risk, not eliminate risk.
```

---

## 1. Short-Lived Access Tokens
Purpose:
```
Reduce attack window.
```
Example:
```
Access Token → 15-30 minutes
```
If stolen:
```
Attacker gets limited access time.
```
---
## 2. Refresh Tokens

Purpose:
```
Issue new access tokens without forcing login.
```
Example:
```
Access Token  → 15 minRefresh Token → 7 days
```
Benefit:
```
Access token stays short-lived.
```
---
## 3. HTTPS Everywhere
Purpose:
```
Prevent token interception in transit.
```
Without HTTPS:
```
Token can be sniffed on network.
```
Production Rule:
```
Always use HTTPS.
```
---
## 4. Safe Token Storage
Avoid:
```
localStorage (XSS risk)
```
Prefer:
```
HttpOnly Cookies
```
Benefit:
```
JavaScript cannot read token.
```
---
## 5. Refresh Token Rotation
Flow:
```
Old Refresh Token
        ↓
Invalidate
        ↓
Issue New Refresh Token
```
Benefit:
```
Detect stolen refresh tokens.
```
---

## 6. Token Revocation

Invalidate sessions when:

```
Password changed
Account disabled
Critical role change
```

Benefit:

```
Force re-authentication.
```

---

## 7. Device / Session Tracking

Store active sessions:

```
Chrome - Laptop
Android - Mobile
```

Benefit:

```
User can revoke individual sessions.
```

---

## 8. Risk-Based Security Checks

Monitor:

```
IP changes
Device changes
Geographic anomalies
```

Example:

```
India login
2 min later Brazil request
```

Action:

```
Require re-authentication.
```

---

## Recommended DevSync Setup

```
Access Token  → 15-30 min
Refresh Token → 7 days
HTTPS         → Enabled in deployment
Password Change → Revoke sessions
```

---

# Security Principle

```
Assume token theft is possible.  
Design to:  
- minimize impact  
- detect abuse  
- revoke access quickly
```

This is how real-world authentication systems are designed.


## SecurityContext & Authentication Flow
`SecurityContext = request-scoped security container
Spring Security stores authentication information here.
security context exists per each request
It enables:
- authorization
- role checks
- `@PreAuthorize`
- authenticated endpoint protection
- principal injection
- Spring Security ecosystem integration
```
Request
   ↓
JWT Filter validates token
   ↓
Authentication object created
   ↓
Stored inside SecurityContext
   ↓
Controllers/services access current user
```

### What Is Authentication Object?
`Authentication` represents: 
`trusted authenticated identity INSIDE application`

It contains:
- principal (user) -`identity of authenticated user` like email,username and etc
- authorities/roles - permissions/roles like  ROLE_ADMIN,ROLE_MEMBER
- authentication status
- ```
  Authentication
    ├── principal 
    ├── authorities
    ├── authenticated
  ```
  
```
principal:
   skilledsapien@gmail.com
authorities:
   ROLE_ADMIN
authenticated:
   true
```

> Authentication object is NOT the JWT itself. JWT is just `credential/token`
> Authentication object is `trusted authenticated identity inside application`

### SecurityContextHolder
Spring stores SecurityContext using: SecurityContextHolder
`global access point for current authentication`
```
token valid
    ↓
create Authentication
    ↓
SecurityContextHolder.setContext(...)
```
can be used later
`SecurityContextHolder.getContext()`

### Important Note
> Spring usaually stores securityContext using `ThreadLocal`

What is threadLocal - > each request runs on a separate thread.
ThreadLocal allows : `data isolated per request thread`



```
Client Request
      ↓
JWT Filter
      ↓
Validate JWT
      ↓
Load user details
      ↓
Create Authentication object
      ↓
Store in SecurityContext
      ↓
Controller executes
      ↓
Current user accessible anywhere
```


### `@Configuration`
This makes Spring Bean Configuration Class
methods annotated with `@Bean` become Spring-managed beans.

### `@Bean`
This tells Spring that
> Create and manage this object inside IoC container.
Example:
```
@Bean
PasswordEncoder passwordEncoder()
```
Spring will creates and manages the object for password encoder inside IoC containers
#### What Spring Does Internally
Spring creates: `new BCryptPasswordEncoder()` stores in  `ApplicationContext`Then injects it wherever needed.

```
@Bean PasswordEncoder
        ↓
Spring Container
        ↓
Injected into AuthService
```

### @EnableWebSecurity 
this tell spring that
> `@EnableWebSecurity` enables Spring Security's web layer and allows configuration of request-level security through `SecurityFilterChain`.
- Activates Spring Security for HTTP requests.
- Allows configuration of:
    - Authentication
    - Authorization
    - Login/Logout
    - JWT filters
    - CORS
    - CSRF
```
@Configuration
@EnableWebSecurity
public class SecurityConfig {
}
```

```
@Bean
SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    return http
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/public/**").permitAll()
                    .anyRequest().authenticated()
            )
            .build();
}
```

### @EnableMethodSecurity
```
@Configuration
@EnableMethodSecurity
public class SecurityConfig {
}
```

> Enables **method-level security annotations**.
Enables
- `@PreAuthorize`
- `@PostAuthorize`
- `@PreFilter`
- `@PostFilter`
- `@Secured`
- `@RolesAllowed`
@PreAuthorize -> performs an authority check before the method executes
Examples:
```
@PreAuthorize("isAuthenticated()")
```
Authenticated users only.
```
@PreAuthorize("hasRole('USER')")
```
Users with role USER.
```
@PreAuthorize("hasAnyRole('ADMIN','MANAGER')")
```
ADMIN or MANAGER.
```
@PreAuthorize("hasAuthority('DELETE_USER')")
```
Specific authority/permission.
```
@PreAuthorize("#id == authentication.principal.id")
```

User can access only their own resource.

## Important Points

> Why is password hashing done during registration,
but password verification done during login using `matches()` instead of encoding again and comparing strings?

BCrypt generates random salt for every password hash. that means even if we hash the same string we get two different hashes
this prevents
- rainbow tables attacks
- identical password detection
so that's why we use `matches()` instead of hashing the password again.

What does `matches()` do:
```
Raw Password
       ↓
Extract salt from stored hash
       ↓
Hash raw password USING SAME salt + same algorithm
       ↓
Compare resulting hash
```
> The salt is actually embedded INSIDE the BCrypt hash string. 

Hash contains:
- algorithm info
- cost factor
- salt
- hash
BCrypt knows how to parse it automatically.


## UserDetails & UserDetailsService

Why user details?
Suppose our application has:
```
class User {
    UUID id;
    String email;
    String password;
    String firstName;
}
```

But Spring Security internally needs standardized information like:
```
username
password
roles
isAccountLocked
isEnabled
```
Spring cannot assume:
- your entity is named `User`
- your login field is `email`
- your password field name
- your role structure
>“I don’t care about your entity structure.  
Just give me user information in MY expected format.”

Thats why expected format is UserDetails

What is UserDetails?
A standard security-user format understood by Spring Security.
It contains 
```
username
password
roles
isAccountLocked
isEnabled
```

#### UserDetailsService
The Key responsibility of User Details Service is to load users from the DB during authentication

Flow
```
Login Request
      ↓
UserDetailsService
      ↓
load User from DB
      ↓
convert to UserDetails
      ↓
Spring verifies password
      ↓
Authentication created
      ↓
SecurityContext populated
```

Login Flow
1. User logins 
```
{
  "email": "user@mail.com",
  "password": "Password123!"
}
```

2. Spring need user info like `stored password + roles.`
	 so it calls UserDetailsService.loadUserbyUsername(email);

3. We convert user into UserDetails   `User` --> `CustomUserDetails` (This is adapter step)
4. Spring receives UserDetails
5. Spring has:
	- hashed password
	- username
	- authorities
6. Password Verification - Spring internally does the password verification
7.  Authentication object created and security context is populated.


### AuthenticationManager & AuthenticationProvider

#### AuthenticationManager
Its job is to navigate the incoming authentication request to appropriate auth mechanisms.

```
receive authentication request
        ↓
find appropriate authentication mechanism
        ↓
delegate authentication
        ↓
return authenticated Authentication object
```

Why AuthenticationManager?
Application may support multi auth methods like
```
username/password
JWT
OAuth2
LDAP
API keys
SAML
```
Each auth type requires:
- different credential parsing
- different verification logic
So Spring avoids giant monolithic authentication logic.

`This pattern is called Strategy Pattern Architecture`
### AuthenticationProvider
This is where the actual verification happens
It know how to authenticate a specific credential method

Responsibilites
```
1. validate credentials
2. load principal
3. verify identity
4. create authenticated Authentication object
```

#### 1. DaoAuthenticationProvider
It means `database-backed authentication`

```
Request
   ↓
AuthenticationManager
   ↓
AuthenticationProvider
   ↓
UserDetailsService
   ↓
Repository
   ↓
Password Verification
   ↓
Authenticated Authentication object
   ↓
SecurityContext
```


## SecurityFilterChain & Request Processing

SecurityFilterChain --> Ordered pipeline of security filters

```
Security is composed of MANY responsibilities:

- authentication
- authorization
- CSRF protection
- session handling
- exception handling
- logout
- JWT validation
  
  Spring splits responsibilities into filters.
```
Spring Security internally uses:
```
DelegatingFilterProxy
        ↓
FilterChainProxy
        ↓
SecurityFilterChain
        ↓
Individual Filters
```

`Authority -> simply a granted permission`
authority represents - atomic permission
example
```
ISSUE_CREATE
ISSUE_DELETE
PROJECT_EDIT
USER_INVITE
```

`Role -> collection of authorities`
represents business grouping of permissions
like
```
PROJECT_CREATE
PROJECT_DELETE
ISSUE_DELETE
USER_MANAGE
```



# Why Spring Uses ROLE_ Prefix
Convention.
When you write:
```
hasRole("ADMIN")
```
Spring internally checks:
```
ROLE_ADMIN
```
# Example
This:
```
hasRole("ADMIN")
```
becomes:
```
hasAuthority("ROLE_ADMIN")
```
internally.
Huge thing to remember.


There are 2 types of authorization styles

A. URL-Based Authorization
```
.requestMatchers("/admin/**")
.hasRole("ADMIN")
```
meaning Any /admin request requires ROLE_ADMIN

B. Method-Level Authorization
`@PreAuthorize("hasRole('ADMIN')")`
We will use preAuthorize on service methods
Example:
```
@PreAuthorize("hasRole('ADMIN')")
public void deleteWorkspace(...)
```


JWT is implemented using the following dependencies
```
jjwt-jackson
jjwt-impl
jjwt-api
```

Important Methods in JWT Service
1. generateAccessToken

```
public String generateAccessToken(UUID userId){  
    long now = System.currentTimeMillis();  
    return Jwts.builder()  
            .subject(userId.toString())  
            .issuedAt(new Date(now))  
            .expiration(new Date(now + expirationMs))  
            .signWith(secretKey)  
            .compact();
```
2. extractClaims
```
public Claims extractClaims(String token){  
    return Jwts.parser().verifyWith(secretKey).build().parseSignedClaims(token).getPayload();  
}
```

3. isTokenValid
```
   public boolean isTokenValid(String token) {  
    try {  
        Jwts.parser()  
                .verifyWith(secretKey)  
                .build()  
                .parseSignedClaims(token);  
  
        return true;  
  
    } catch (ExpiredJwtException ex) {  
        log.warn("JWT token has expired: {}", ex.getMessage());  
  
    } catch (MalformedJwtException ex) {  
        log.warn("Invalid JWT token format: {}", ex.getMessage());  
  
    } catch (SecurityException ex) {  
        log.warn("JWT signature validation failed: {}", ex.getMessage());  
  
    } catch (UnsupportedJwtException ex) {  
        log.warn("Unsupported JWT token: {}", ex.getMessage());  
  
    } catch (IllegalArgumentException ex) {  
        log.warn("JWT token is null or empty");  
  
    } catch (WeakKeyException ex) {  
        log.error("JWT secret key is too weak: {}", ex.getMessage());  
  
    } catch (Exception ex) {  
        log.error("Unexpected error while validating JWT", ex);  
    }  
  
    return false;  
}
```

valid token and extract token can be merged and used for token validation and extracting claims

once we build the [JWT service](https://github.com/exoscar/dev-sync/blob/master/src/main/java/org/devsync/spring/common/security/JwtFilter.java) class 

then we have to build the [customUserDetails](https://github.com/exoscar/dev-sync/blob/master/src/main/java/org/devsync/spring/common/security/CustomUserDetails.java) and [CustomUserDetailsService](https://github.com/exoscar/dev-sync/blob/master/src/main/java/org/devsync/spring/common/security/CustomUserDetailsService.java).

CustomUserDetails implements UserDetails.
CustomUserDetailsService implement UserDetailsService.


After implementing above both. we need to build the [JWT Filter](https://github.com/exoscar/dev-sync/blob/master/src/main/java/org/devsync/spring/common/security/JwtFilter.java).
the filter should be placed before the UsernamePasswordAuthenticationFilter.

1. Get the token from header
2. if no token continue
3. if token is invalid or expired --> throw an exception
4. get sub/claims from the token
5. get userdetails from DB
	`CustomUserDetails userDetails = userDetailsService.loadUserById(userId);`
6. populate the authentication container with userdetails and authorities
	```
	UsernamePasswordAuthenticationToken authentication =  
        new UsernamePasswordAuthenticationToken(  
                userDetails,  
                null,  
                userDetails.getAuthorities()  
        );  
  
	SecurityContextHolder.getContext()  
        .setAuthentication(authentication);
	```

7. populate the securityContextHolder

