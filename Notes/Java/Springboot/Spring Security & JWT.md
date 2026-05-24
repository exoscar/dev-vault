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