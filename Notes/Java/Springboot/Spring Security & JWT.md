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

#### Signature
`It prevents tampering`
Generated using:
- header
- payload
- secret key

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
```
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