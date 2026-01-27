

# The One-Sentence Mental Model of Spring

> **Spring is a container that creates, wires, manages, and orchestrates objects for you, and exposes them over HTTP (or other protocols) when requested.**

Everything in Spring flows from this.

---

# The Core Idea: Inversion of Control (IoC)

Normally, in plain Java:

```java
UserService service = new UserService(new UserRepository());
```

 **You create objects and decide their dependencies**

Spring flips this.

> You describe *what you want*, and **Spring decides *when* and *how* to create it.**

This is **Inversion of Control**.

---

# Spring = 3 Big Layers (lock this in)

```
┌────────────────────────────┐
│   Spring MVC (Web Layer)   │
│ Controllers, Requests     │
└────────────▲──────────────┘
             │
┌────────────┴──────────────┐
│   Application Layer       │
│ Services, Business Logic  │
└────────────▲──────────────┘
             │
┌────────────┴──────────────┐
│   Infrastructure Layer    │
│ Repositories, DB, APIs    │
└───────────────────────────┘
```

Spring **connects these layers automatically**.

---

# The Spring Container (the heart)

## What is the Spring Container?

> A runtime environment that:

* Creates objects
* Injects dependencies
* Manages lifecycle
* Applies cross-cutting logic (transactions, security, etc.)

These objects are called **Beans**.

---

## Bean

> A **Bean** is just a Java object whose lifecycle is managed by Spring.

Nothing magical:

```java
class UserService { }
```

Becomes a Bean if Spring knows about it.

---

# How Spring Knows About Beans

Spring scans your code.

Annotations tell Spring:

* “Create this”
* “Inject this”
* “Expose this”

### Common stereotypes

| Annotation                        | Meaning        |
| --------------------------------- | -------------- |
| `@Component`                      | Generic bean   |
| `@Service`                        | Business logic |
| `@Repository`                     | Data access    |
| `@Controller` / `@RestController` | Web endpoints  |

All of them mean:

> “Spring, create this class as a Bean”

---

# Dependency Injection (DI)

Instead of this:

```java
UserService service = new UserService();
```

You do:

```java
@Service
public class UserService {
    private final UserRepository repo;

    public UserService(UserRepository repo) {
        this.repo = repo;
    }
}
```

Spring:

1. Sees `UserService`
2. Sees it needs `UserRepository`
3. Finds a Bean of that type
4. Injects it

**You never call `new` for core app objects.**

---

# Application Startup (VERY important)

When Spring Boot starts:

1. JVM starts
2. Spring Boot bootstrap
3. Component scanning
4. Bean definitions created
5. Dependency graph built
6. Beans instantiated
7. Application ready

Think:

> “Spring builds a giant object graph at startup.”

---

# Spring MVC: Request Flow Mental Model

This is where backend devs live daily.

---

## High-level flow

```
HTTP Request
   ↓
DispatcherServlet
   ↓
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
   ↑
Response
```

---

## DispatcherServlet (the traffic cop)

> **Every HTTP request hits ONE entry point.**

That entry point is:

```
DispatcherServlet
```

Its job:

* Match URL + HTTP method
* Find correct controller method
* Handle serialization/deserialization
* Apply filters & interceptors

---

# Controller

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        ...
    }
}
```

Controller responsibilities:

* Accept HTTP input
* Convert input → Java
* Call service
* Return result

Controllers should NOT:

* Contain business logic
* Talk to database

---

# Service (business brain)

```java
@Service
public class UserService {
    public User getUser(Long id) {
        // rules, validation, orchestration
    }
}
```

Service:

* Implements business rules
* Coordinates repositories
* Is reusable
* Transaction boundary (usually)

---

# Repository (data access)

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

This is wild but important:

> Spring generates the implementation at runtime.

Repositories:

* Talk to DB
* Return domain objects
* Hide SQL/JPA details

---

# Spring Data JPA Mental Model

You are NOT calling SQL directly.

You are saying:

> “Here’s the data shape, Spring—handle persistence.”

Spring:

* Builds queries
* Manages connections
* Handles mapping

---

# How Spring “magically” adds behavior (AOP)

This is where beginners get lost.

## Aspect-Oriented Programming (AOP)

> Spring wraps your beans in **proxies**.

Example:

```java
@Transactional
public void createUser() { }
```

Spring actually does:

```
Proxy → YourService
```

The proxy:

1. Starts transaction
2. Calls your method
3. Commits / rolls back

Same idea for:

* Security
* Logging
* Caching

---

# Filters vs Interceptors (request lifecycle)

Order matters.

```
Request
 ↓
Filters (Servlet level)
 ↓
Interceptors (Spring level)
 ↓
Controller
```

* Filters: authentication, CORS
* Interceptors: logging, timing

---

# Serialization & Deserialization

When you return:

```java
return user;
```

Spring:

1. Converts Java → JSON
2. Sets headers
3. Writes response

Powered by:

* Jackson
* HttpMessageConverters

---

# Exception Handling Model

Exceptions bubble up.

You define:

```java
@ControllerAdvice
public class GlobalExceptionHandler {
}
```

Spring:

* Catches exception
* Maps it to HTTP response
* Keeps controllers clean

---

# Spring Boot vs Spring (important clarity)

| Spring        | Spring Boot        |
| ------------- | ------------------ |
| Framework     | Opinionated setup  |
| Manual config | Auto configuration |
| Flexible      | Fast to start      |

Boot:

* Configures defaults
* Reduces boilerplate
* Still **pure Spring inside**

---

# Mental Model Summary (burn this in)

> **Spring builds an object graph at startup, intercepts requests through a central dispatcher, routes them to controller methods, injects dependencies automatically, and enhances behavior using proxies.**

understand this:

* IoC
* DI
* DispatcherServlet
* Layered architecture
* Proxies



---

