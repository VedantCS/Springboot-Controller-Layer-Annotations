# Spring Security Architecture 

Spring Security is a powerful and highly customizable authentication and authorization framework for Java applications, particularly those built using Spring Boot. Its architecture is designed around a chain of filters, a central security context, and pluggable authentication and authorization mechanisms.

---

# 1. High-Level Architecture Overview

<img width="530" height="146" alt="image" src="https://github.com/user-attachments/assets/f8633549-d8b4-45e2-96e2-eeb2064e2ce2" />
<img width="1514" height="500" alt="image" src="https://github.com/user-attachments/assets/0d1ba266-bb84-4253-96e8-fb20db4416c4" />

<img width="800" height="300" alt="image" src="https://github.com/user-attachments/assets/7434c708-9c62-4a68-b564-db453458f1bd" />

At its core, Spring Security operates as a **Servlet Filter-based system** that intercepts incoming HTTP requests before they reach your application logic.

## Core Flow

```
Incoming HTTP Request
        ↓
Security Filter Chain
        ↓
Authentication (Who are you?)
        ↓
Authorization (What can you do?)
        ↓
Controller / Business Logic
        ↓
Response
```

---

# 2. Key Architectural Components

## 2.1 SecurityContext & SecurityContextHolder

### What is SecurityContext?

* Holds **authentication information** of the current user.
* Accessible throughout the request lifecycle.

```java
SecurityContext context = SecurityContextHolder.getContext();
Authentication auth = context.getAuthentication();
```

### SecurityContextHolder Strategies

| Strategy                    | Description                   |
| --------------------------- | ----------------------------- |
| MODE_THREADLOCAL            | Default, per-thread storage   |
| MODE_INHERITABLETHREADLOCAL | Child threads inherit         |
| MODE_GLOBAL                 | Shared globally (rarely used) |

---

## 2.2 Authentication Object

Represents the currently authenticated user.

### Structure:

```
Authentication
 ├── Principal (UserDetails)
 ├── Credentials (password/token)
 ├── Authorities (roles/permissions)
 └── Authenticated (boolean)
```

---

## 2.3 UserDetails & UserDetailsService

### UserDetails

Represents user-specific data:

```java
public interface UserDetails {
    String getUsername();
    String getPassword();
    Collection<? extends GrantedAuthority> getAuthorities();
}
```

### UserDetailsService

Responsible for loading users:

```java
UserDetails loadUserByUsername(String username);
```

---

## 2.4 GrantedAuthority

Represents roles or permissions:

```
ROLE_USER
ROLE_ADMIN
READ_PRIVILEGE
WRITE_PRIVILEGE
```

---

# 3. The Security Filter Chain (Core of Architecture)

Spring Security is fundamentally **filter-based**.

## Filter Chain Diagram
 <img width="600" height="500" alt="image" src="https://github.com/user-attachments/assets/cf0633e2-05d6-4eb0-adf7-e52a02667e92" />

```
Client Request
     ↓
[DelegatingFilterProxy]
     ↓
[FilterChainProxy]
     ↓
 ┌─────────────────────────────┐
 │ Security Filters Chain      │
 │                             │
 │ 1. SecurityContextPersistenceFilter
 │ 2. UsernamePasswordAuthenticationFilter
 │ 3. BasicAuthenticationFilter
 │ 4. ExceptionTranslationFilter
 │ 5. FilterSecurityInterceptor
 │                             │
 └─────────────────────────────┘
     ↓
Application Controller
```

---

## 3.1 DelegatingFilterProxy

* Entry point from the Servlet container.
* Delegates to Spring-managed beans.

```
web.xml / Servlet Container
        ↓
DelegatingFilterProxy
        ↓
Spring Bean (FilterChainProxy)
```

---

## 3.2 FilterChainProxy

* Central dispatcher of filters.
* Selects appropriate filter chain based on URL.

---

# 4. Important Filters Explained

## 4.1 SecurityContextPersistenceFilter

* Loads SecurityContext at request start.
* Saves it at request end.

```
Before Request → Load Context
After Request  → Save Context
```

---

## 4.2 UsernamePasswordAuthenticationFilter

* Handles login form submissions.
* Extracts username/password.
* Delegates to AuthenticationManager.

---

## 4.3 BasicAuthenticationFilter

* Processes HTTP Basic Auth headers.

```
Authorization: Basic base64(username:password)
```

---

## 4.4 ExceptionTranslationFilter

* Handles security exceptions:

  * AuthenticationException
  * AccessDeniedException

---

## 4.5 FilterSecurityInterceptor

* Final authorization check.
* Decides if request is allowed.

---

# 5. Authentication Architecture

## 5.1 AuthenticationManager

Main entry point for authentication.

```java
Authentication authenticate(Authentication authentication);
```

---

## 5.2 ProviderManager

* Default implementation.
* Delegates to multiple AuthenticationProviders.

```
AuthenticationManager
        ↓
ProviderManager
        ↓
[AuthProvider1, AuthProvider2, ...]
```

---

## 5.3 AuthenticationProvider

Each provider handles a specific authentication type:

| Provider                   | Purpose           |
| -------------------------- | ----------------- |
| DaoAuthenticationProvider  | Username/password |
| JwtAuthenticationProvider  | JWT tokens        |
| LdapAuthenticationProvider | LDAP              |

---

## Authentication Flow
<img width="800" height="350" alt="image" src="https://github.com/user-attachments/assets/b3dd6359-8169-43dd-807b-d90bb96f86f8" />

```
Login Request
     ↓
UsernamePasswordAuthenticationFilter
     ↓
AuthenticationManager
     ↓
AuthenticationProvider
     ↓
UserDetailsService
     ↓
User Loaded + Password Checked
     ↓
Authentication Success
```

---

# 6. Authorization Architecture

## 6.1 AccessDecisionManager

Decides whether a user can access a resource.

---

## 6.2 AccessDecisionVoters

Vote on access decisions:

| Voter              | Role                  |
| ------------------ | --------------------- |
| RoleVoter          | Checks roles          |
| AuthenticatedVoter | Checks authentication |
| CustomVoter        | Custom logic          |

---

## Voting Process

```
Request → Voters → Decision
            ↓
     GRANT / DENY / ABSTAIN
```

---

## 6.3 Method-Level Security

Enabled via:

```java
@EnableMethodSecurity
```

### Annotations:

```java
@PreAuthorize("hasRole('ADMIN')")
@PostAuthorize("returnObject.owner == authentication.name")
```

---

# 7. Session Management

Spring Security supports:

* Stateful sessions (default)
* Stateless APIs (JWT)

## Stateful

```
Session ID stored in cookie
SecurityContext stored in session
```

## Stateless (JWT)

```
Each request carries token
No session stored
```

---

# 8. CSRF Protection

Cross-Site Request Forgery protection is enabled by default.

## Flow:

```
Server generates CSRF token
        ↓
Client sends token with request
        ↓
Server validates token
```

---

# 9. Password Encoding

Passwords are never stored in plain text.

## PasswordEncoder

```java
PasswordEncoder encoder = new BCryptPasswordEncoder();
```

### Supported Encodings:

* BCrypt (recommended)
* PBKDF2
* SCrypt

---

# 10. JWT Authentication Architecture
<img width="800" height="320" alt="image" src="https://github.com/user-attachments/assets/557fc5dd-072e-4b41-a7b6-2773d76d8516" />

## Flow

```
Client Login
     ↓
Server Generates JWT
     ↓
Client Stores Token
     ↓
Client Sends Token in Header
     ↓
JWT Filter Validates Token
     ↓
User Authenticated
```

---

# 11. Customization Points

You can customize:

* Filters
* AuthenticationProviders
* UserDetailsService
* Security Config

---

## Example Configuration

```java
@Bean
SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf().disable()
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        )
        .formLogin();

    return http.build();
}
```

---

# 12. Complete Request Lifecycle

```
1. Request enters Filter Chain
2. SecurityContext loaded
3. Authentication attempted
4. Authentication stored
5. Authorization checked
6. Controller executed
7. Response returned
8. SecurityContext saved
```

---

# 13. Advanced Concepts

## 13.1 Remember-Me Authentication

* Stores token in cookie
* Auto-login without credentials

---

## 13.2 OAuth2 / OpenID Connect

* Delegates authentication to external providers
* Example: Google login

---

## 13.3 Reactive Security (WebFlux)

Uses:

* `SecurityWebFilterChain`
* Non-blocking authentication

---

# 14. Visual Summary Diagram

```
                +----------------------+
                |   Client Request     |
                +----------+-----------+
                           ↓
                +----------------------+
                | DelegatingFilterProxy|
                +----------+-----------+
                           ↓
                +----------------------+
                |  FilterChainProxy    |
                +----------+-----------+
                           ↓
     +---------------------------------------------+
     | Security Filters Chain                      |
     |---------------------------------------------|
     | SecurityContextPersistenceFilter            |
     | Authentication Filters                      |
     | ExceptionTranslationFilter                  |
     | FilterSecurityInterceptor                   |
     +---------------------------------------------+
                           ↓
                +----------------------+
                | AuthenticationManager|
                +----------+-----------+
                           ↓
                +----------------------+
                | AuthenticationProvider|
                +----------+-----------+
                           ↓
                +----------------------+
                |   UserDetailsService |
                +----------------------+
```

---

# 15. Key Takeaways

* Spring Security is **filter-chain based**
* Authentication is handled via **AuthenticationManager**
* Authorization is handled via **AccessDecisionManager**
* Fully **pluggable and customizable**
* Supports **stateful and stateless architectures**

---

# 16. Final Thoughts

Spring Security’s architecture is built with clear separation of concerns:

* Filters → Request interception
* Managers → Decision making
* Providers → Authentication logic
* Context → State management

Understanding these layers deeply allows you to build secure, scalable, and flexible applications.

---


