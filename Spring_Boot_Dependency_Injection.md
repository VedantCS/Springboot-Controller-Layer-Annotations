# Spring Boot Dependency Injection - Complete Guide

> **Comprehensive notes on Dependency Injection in Spring Boot**  


---

## Table of Contents
- [1. Problem DI Tries to Solve](#1-problem-di-tries-to-solve)
- [2. Dependency Inversion Principle](#2-dependency-inversion-principle)
- [3. What is Dependency Injection?](#3-what-is-dependency-injection)
- [4. DI in Spring: Components and Autowiring](#4-di-in-spring-components-and-autowiring)
- [5. Three Ways of Dependency Injection](#5-three-ways-of-dependency-injection)
- [6. Field Injection](#6-field-injection)
- [7. Setter Injection](#7-setter-injection)
- [8. Constructor Injection (Recommended)](#8-constructor-injection-recommended)
- [9. Common DI Issues](#9-common-di-issues)
- [10. Circular Dependency](#10-circular-dependency)
- [11. Unsatisfied Dependency](#11-unsatisfied-dependency)
- [12. Summary and Best Practices](#12-summary-and-best-practices)

---

## 1. Problem DI Tries to Solve

### Example Setup: Tightly Coupled Classes

```java
class User {
    private Order order;

    public User() {
        this.order = new Order();  // Direct instantiation
    }
}

class Order {
    public Order() {
        // Constructor logic
    }
}
```

### Problems with This Approach

1. **Tight Coupling**
   - `User` class is tightly coupled to `Order` class
   - Any change in `Order` structure directly impacts `User`

2. **Maintainability Issues**
   - Hard to change implementations
   - Difficult to test in isolation
   - Violates SOLID principles

3. **Inflexibility**
   - Cannot easily swap implementations
   - Cannot use interfaces effectively

---

## 2. Dependency Inversion Principle

> **D in SOLID**: High-level modules should not depend on low-level modules. Both should depend on abstractions.

### Scenario: Moving to Interfaces

```java
interface Order { 
    void process();
}

@Component
class OnlineOrder implements Order {
    @Override
    public void process() {
        // Online order logic
    }
}

@Component
class OfflineOrder implements Order {
    @Override
    public void process() {
        // Offline order logic
    }
}
```

###  Wrong Approach (Violates Dependency Inversion)

```java
class User {
    private Order order;

    public User() {
        this.order = new OnlineOrder();  // Depends on concrete implementation
    }
}
```

**Problem**: Now `User` depends on concrete `OnlineOrder` implementation, not abstraction.

###  Correct Approach (Follows Dependency Inversion)

```java
class User {
    private Order order;

    // Dependency comes from outside
    public User(Order order) {
        this.order = order;
    }
}
```

**Benefits**:
- `User` depends only on `Order` interface (abstraction)
- External code decides which implementation to inject
- Easy to switch between `OnlineOrder` and `OfflineOrder`

---

## 3. What is Dependency Injection?

### Definition

**Dependency Injection (DI)** is a design pattern where an object receives its dependencies from external sources rather than creating them itself.

### Goals

-  Make classes **independent** of concrete dependencies
-  Achieve **loose coupling**
-  Support **Dependency Inversion Principle**
-  Improve **testability**
-  Enable **flexible configuration**

### In Spring Framework

Spring acts as the **external source** that:
1. Creates objects (beans)
2. Manages their lifecycle
3. Resolves dependencies
4. Injects the right implementations

---

## 4. DI in Spring: Components and Autowiring

### Basic Spring DI Example

```java
@Component
class User {
    @Autowired
    private Order order;

    public void processUser() {
        order.process();
    }
}

@Component
class Order {
    public void process() {
        System.out.println("Processing order...");
    }
}
```

### Key Annotations

#### `@Component`
- Marks a class as a Spring-managed bean
- Tells Spring: "You manage the lifecycle of this class"
- Spring includes it in the IoC container

#### `@Autowired`
- Tells Spring to inject a dependency
- Spring's process:
  1. Look for an existing bean of the required type
  2. If found → inject it
  3. If not found → create it, then inject

### How Spring Resolves Dependencies

1. **Application Startup**: IoC container initializes
2. **Bean Discovery**: Scans for `@Component` classes
3. **Bean Creation**: Instantiates beans
4. **Dependency Resolution**: Uses reflection to find `@Autowired` fields/methods
5. **Dependency Injection**: Injects required beans

---

## 5. Three Ways of Dependency Injection

Spring supports three main injection types:

| Type | Annotation Location | Use Case | Recommendation |
|------|---------------------|----------|----------------|
| **Field Injection** | On the field | Quick prototyping | ❌ Avoid |
| **Setter Injection** | On setter method | Optional dependencies | ⚠️ Use sparingly |
| **Constructor Injection** | On constructor | Mandatory dependencies | ✅ **Recommended** |

---

## 6. Field Injection

### Syntax

```java
@Component
class User {
    @Autowired
    private Order order;

    public User() {
        System.out.println("User initialized");
    }

    public void process() {
        order.process();
    }
}

@Component
@Lazy
class Order {
    public Order() {
        System.out.println("Order initialized");
    }

    public void process() {
        System.out.println("Order processing");
    }
}
```

### How It Works

1. Spring creates `User` bean
2. Uses **reflection** to scan fields
3. Finds `@Autowired` on `order` field
4. Checks if `Order` bean exists
5. Creates `Order` bean if needed (respecting `@Lazy`)
6. Injects `Order` into `User.order` field

###  Advantages

1. **Simple and concise**
   - Easy to write
   - Minimal boilerplate

2. **Easy to read**
   - Dependencies visible directly on fields
   - Quick understanding of class dependencies

###  Disadvantages

1. **Cannot Make Fields Immutable**

```java
@Autowired
private final Order order = null;  // Doesn't work as expected
```

- Spring uses reflection and ignores `final`
- Breaks immutability principle
- Field value changes after initialization

2. **NullPointerException Risk**

```java
User user = new User();  // Created manually, not by Spring
user.process();          //  NullPointerException: order is null
```

- When you create objects with `new`, Spring doesn't inject dependencies
- Easy to accidentally bypass Spring's DI

3. **Testing Complexity**

```java
@Test
void testUser() {
    User user = new User();  // How to inject mock Order?
    // Need @Mock and @InjectMocks (reflection-based)
}
```

- Requires test frameworks like Mockito
- Uses `@Mock` and `@InjectMocks` (reflection-based)
- Cannot simply pass mock dependencies

### When to Use

-  Quick prototypes only
-  **Not recommended** for production code

---

## 7. Setter Injection

### Syntax

```java
@Component
class User {
    private Order order;

    @Autowired
    public void setOrder(Order order) {
        this.order = order;
    }

    public void process() {
        order.process();
    }
}
```

### How It Works

1. Spring creates `User` bean (calls constructor)
2. Uses reflection to find methods with `@Autowired`
3. Calls `setOrder()` with resolved `Order` bean
4. Dependency injected after construction

###  Advantages

1. **Mutable Dependencies**

```java
user.setOrder(onlineOrder);
// Later...
user.setOrder(offlineOrder);  // Can change implementation
```

- Dependencies can be changed after object creation
- Useful for optional or reconfigurable dependencies

2. **Easier Testing Than Field Injection**

```java
@Test
void testUser() {
    User user = new User();
    user.setOrder(mockOrder);  //  Easy to inject mock
    user.process();
}
```

- Can inject mocks via setter methods
- No need for reflection-based test frameworks

###  Disadvantages

1. **Cannot Make Fields Immutable**

```java
private final Order order;  //  Cannot use final with setter injection
```

- Field must be mutable to be set via setter
- No `final` keyword allowed

2. **Harder to Read and Maintain**

```java
@Component
class User {
    private Order order;           // Is this managed by Spring?
    private Invoice invoice;       // Which ones are injected?
    private Payment payment;

    @Autowired
    public void setOrder(Order order) { ... }

    // Hard to track which fields are dependencies
}
```

- Not obvious which fields are Spring-managed
- Must scan for setter methods with `@Autowired`
- Easy to miss if someone forgets annotation

3. **Object in Incomplete State**

```java
User user = new User();  // Object created
// order is null here!
user.process();          // Potential NullPointerException
// Later Spring calls setOrder()
```

- Object exists in incomplete state between construction and setter call

### When to Use

- Optional dependencies (with proper null checks)
- Dependencies that may change at runtime
- Use sparingly in modern Spring applications

---

## 8. Constructor Injection (Recommended)

### Syntax

```java
@Component
class User {
    private final Order order;

    @Autowired  // Optional from Spring 4.3+ if only one constructor
    public User(Order order) {
        this.order = order;
    }

    public void process() {
        order.process();
    }
}

@Component
class Order {
    public void process() {
        System.out.println("Processing order");
    }
}
```

### Spring 4.3+ Feature: Implicit Autowiring

```java
@Component
class User {
    private final Order order;

    // @Autowired is optional if there's only one constructor
    public User(Order order) {
        this.order = order;
    }
}
```

**Rule**: If a class has **only one constructor**, `@Autowired` is optional.

### Multiple Constructors

```java
@Component
class User {
    private final Order order;
    private final Invoice invoice;

    // Constructor 1
    public User(Order order) {
        this.order = order;
        this.invoice = null;
    }

    // Constructor 2
    @Autowired  // Must specify which one to use
    public User(Invoice invoice) {
        this.invoice = invoice;
        this.order = null;
    }
}
```

**Rule**: If **multiple constructors** exist, you **must** annotate one with `@Autowired`.

**What happens without `@Autowired`?**
-  Spring throws **BeanInitializationException**
- Error: "Could not determine which constructor to use"

###  Advantages

#### 1. All Mandatory Dependencies Set at Initialization

```java
//  Object is fully initialized from the start
User user = new User(order);
// No intermediate state where order is null
```

- Object cannot exist without required dependencies
- Always in a valid state
- No partial initialization

#### 2. Immutable Fields (Final)

```java
@Component
class User {
    private final Order order;        // ✅ Immutable
    private final Invoice invoice;    // ✅ Immutable

    public User(Order order, Invoice invoice) {
        this.order = order;
        this.invoice = invoice;
    }
}
```

- Dependencies marked as `final`
- Set once in constructor
- Cannot be changed later
- Thread-safe by design

#### 3. Fail-Fast Behavior

```java
@Component
class User {
    private final Order order;

    public User(Order order) {  // If Order bean missing
        this.order = order;      // → Fails at STARTUP
    }
}
```

- Missing dependencies cause **startup failure**
- Errors discovered immediately, not at runtime
- No `NullPointerException` surprises later

**Comparison**:

| Injection Type | Error Discovery |
|----------------|----------------|
| Field | Runtime (when field accessed) |
| Setter | Runtime (when setter should be called) |
| **Constructor** | **Startup (fail-fast)**  |

#### 4. Simple Unit Testing

```java
@Test
void testUser() {
    Order mockOrder = mock(Order.class);
    User user = new User(mockOrder);  //  Easy and clean

    user.process();

    verify(mockOrder).process();
}
```

- No special annotations needed
- No reflection required
- Pure Java instantiation
- Clean and readable tests

#### 5. Better IDE Support

- IDEs can easily:
  - Show required dependencies
  - Navigate to dependency usages
  - Refactor safely
  - Detect missing dependencies at compile time

### ⚠️ "Disadvantage" (Actually a Design Smell Detector)

```java
@Component
class User {
    // ⚠️ Too many dependencies!
    public User(Order order, Invoice invoice, Payment payment,
                Notification notification, Logger logger,
                Cache cache, Validator validator,
                EmailService email, SmsService sms,
                Analytics analytics) {
        // 10+ constructor parameters = design smell
    }
}
```

**Issue**: Large constructors indicate too many responsibilities.

**Solution**: This is actually an **advantage**!
- Forces you to identify design problems
- Encourages refactoring into smaller, focused classes
- Promotes Single Responsibility Principle

### Best Practices

1. **Use Constructor Injection for Mandatory Dependencies**

```java
@Component
class UserService {
    private final UserRepository userRepository;  // Required
    private final EmailService emailService;      // Required

    public UserService(UserRepository userRepository, 
                       EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

2. **Combine with Setter Injection for Optional Dependencies**

```java
@Component
class UserService {
    private final UserRepository userRepository;  // Mandatory
    private NotificationService notificationService;  // Optional

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Autowired(required = false)
    public void setNotificationService(NotificationService service) {
        this.notificationService = service;
    }
}
```

3. **Use Lombok to Reduce Boilerplate** (Optional)

```java
@Component
@RequiredArgsConstructor  // Generates constructor for final fields
class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
}
```

---

## 9. Common DI Issues

Two major problems in Spring DI:

1. **Circular Dependency** - Beans depend on each other in a cycle
2. **Unsatisfied Dependency** - Multiple bean candidates, Spring cannot choose

---

## 10. Circular Dependency

### Problem Example

```java
@Component
class Order {
    @Autowired
    private Invoice invoice;  // Order needs Invoice
}

@Component
class Invoice {
    @Autowired
    private Order order;      // Invoice needs Order
}
```

**Cycle**: Order → Invoice → Order → ...

### Error Message

```
***************************
APPLICATION FAILED TO START
***************************

Description:
The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  order defined in file [Order.class]
↑     ↓
|  invoice defined in file [Invoice.class]
└─────┘
```

### Solutions (In Order of Preference)

####  Solution 1: Refactor Code (Best Approach)

**Problem**: Circular dependencies usually indicate design issues.

**Solution**: Extract common logic into a separate class.

```java
// Common logic extracted
@Component
class OrderProcessor {
    public void process(Order order, Invoice invoice) {
        // Common processing logic
    }
}

@Component
class Order {
    @Autowired
    private OrderProcessor processor;  // No circular dependency
}

@Component
class Invoice {
    @Autowired
    private OrderProcessor processor;  // No circular dependency
}
```

**Benefits**:
- Removes cycle
- Better design
- Single Responsibility Principle
- Easier to test

---

#### Solution 2: Use `@Lazy` on Autowired

```java
@Component
class Order {
    @Autowired
    @Lazy  // Injects a proxy instead of actual bean
    private Invoice invoice;
}

@Component
class Invoice {
    @Autowired
    private Order order;
}
```

### How `@Lazy` Breaks the Cycle

**Without `@Lazy`**:
1. Spring creates `Order` → needs `Invoice`
2. Spring creates `Invoice` → needs `Order`
3. Spring tries to create `Order` again → **CYCLE DETECTED** ❌

**With `@Lazy` on Order's invoice field**:
1. Spring creates `Invoice` → needs `Order`
2. Spring creates `Order` → needs `Invoice`
3. Spring sees `@Lazy` → injects **proxy** (not real bean)
4. Cycle broken! 
5. Real `Invoice` bean created when first accessed

**Key Points**:
- `@Lazy` on `@Autowired` creates a **proxy**
- Proxy delays actual bean creation
- Real bean created on first method call
- Works with field, setter, and constructor injection

**Example with Constructor Injection**:

```java
@Component
class Order {
    private final Invoice invoice;

    public Order(@Lazy Invoice invoice) {  // @Lazy on parameter
        this.invoice = invoice;
    }
}
```

---

####  Solution 3: `@PostConstruct` Manual Wiring (Hacky)

```java
@Component
class Order {
    @Autowired
    private Invoice invoice;  // Spring manages this
}

@Component
class Invoice {
    private Order order;  // NOT managed by Spring initially

    @PostConstruct
    public void init() {
        // Manually wire after bean creation
        // Usually need ApplicationContext to get Order bean
    }

    public void setOrder(Order order) {
        this.order = order;
    }
}
```

**How It Works**:
1. Spring creates `Order` and `Invoice` beans
2. Spring injects `Invoice` into `Order`
3. `@PostConstruct` runs **after** dependency injection
4. Manually set `Order` into `Invoice`

**Why It's Hacky**:
- Bypasses Spring's dependency management
- Manual wiring required
- Error-prone
- Hard to maintain

### Solution Priority

1. **🥇 Refactor** - Remove the cycle (best design)
2. **🥈 @Lazy** - Use when refactoring not possible
3. **🥉 @PostConstruct** - Last resort only

---

## 11. Unsatisfied Dependency

### Problem: Multiple Bean Candidates

```java
interface Order {
    void process();
}

@Component
class OnlineOrder implements Order {
    @Override
    public void process() {
        System.out.println("Processing online order");
    }
}

@Component
class OfflineOrder implements Order {
    @Override
    public void process() {
        System.out.println("Processing offline order");
    }
}

@Component
class User {
    @Autowired
    private Order order;  //  error: Which one? OnlineOrder or OfflineOrder?
}
```

### Error Message

```
***************************
APPLICATION FAILED TO START
***************************

Description:
Field order in com.example.User required a single bean, but 2 were found:
    - onlineOrder: defined in file [OnlineOrder.class]
    - offlineOrder: defined in file [OfflineOrder.class]

Action:
Consider marking one of the beans as @Primary, or use @Qualifier
```

### Solutions

####  Solution 1: Use `@Primary`

```java
@Component
@Primary  // This is the default choice
class OnlineOrder implements Order {
    @Override
    public void process() {
        System.out.println("Processing online order");
    }
}

@Component
class OfflineOrder implements Order {
    @Override
    public void process() {
        System.out.println("Processing offline order");
    }
}

@Component
class User {
    @Autowired
    private Order order;  //  Injects OnlineOrder (marked as @Primary)
}
```

**How It Works**:
- `@Primary` marks one bean as the **default** when multiple candidates exist
- Spring prefers `@Primary` bean when autowiring by type
- Other beans still available via `@Qualifier`

**When to Use**:
-  One implementation is used in **most cases**
-  Clear default implementation exists

---

####  Solution 2: Use `@Qualifier`

```java
@Component
@Qualifier("onlineOrderBean")
class OnlineOrder implements Order {
    @Override
    public void process() {
        System.out.println("Processing online order");
    }
}

@Component
@Qualifier("offlineOrderBean")
class OfflineOrder implements Order {
    @Override
    public void process() {
        System.out.println("Processing offline order");
    }
}

@Component
class User {
    @Autowired
    @Qualifier("offlineOrderBean")  //  Explicitly choose offline
    private Order order;
}
```

**How It Works**:
- `@Qualifier` gives beans **explicit names**
- At injection point, specify which bean to use
- More explicit than `@Primary`

**When to Use**:
-  No clear default implementation
-  Different beans used in different contexts
-  Need explicit control over which bean to inject

**With Constructor Injection**:

```java
@Component
class User {
    private final Order order;

    public User(@Qualifier("onlineOrderBean") Order order) {
        this.order = order;
    }
}
```

---

#### Combining `@Primary` and `@Qualifier`

```java
@Component
@Primary
@Qualifier("onlineOrderBean")
class OnlineOrder implements Order { }

@Component
@Qualifier("offlineOrderBean")
class OfflineOrder implements Order { }

// Uses @Primary (OnlineOrder)
@Component
class DefaultUser {
    @Autowired
    private Order order;  // OnlineOrder
}

// Explicitly uses offline
@Component
class SpecialUser {
    @Autowired
    @Qualifier("offlineOrderBean")
    private Order order;  // OfflineOrder
}
```

---

## 12. Summary and Best Practices

### Injection Type Comparison

| Feature | Field | Setter | Constructor |
|---------|-------|--------|-------------|
| **Simplicity** |  Very Simple |  Moderate | ⚠️ Moderate |
| **Immutability** |  No |  No |  **Yes** |
| **Mandatory Dependencies** |  No |  No |  **Yes** |
| **Fail-Fast** |  Runtime |  Runtime |  **Startup** |
| **Testing** |  Hard |  Moderate |  **Easy** |
| **Optional Dependencies** |  No |  Yes |  No |
| **Thread Safety** |  No |  No |  **Yes** (with final) |
| **IDE Support** |  Moderate |  Moderate |  **Excellent** |
| **Industry Recommendation** |  Avoid |  Sparingly |  **Preferred** |

### Best Practices

####  DO's

1. **Use Constructor Injection for Services**

```java
@Service
class UserService {
    private final UserRepository repository;
    private final EmailService emailService;

    public UserService(UserRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}
```

2. **Mark Dependencies as Final**

```java
private final UserRepository repository;  // ✅ Immutable
```

3. **Use `@Qualifier` for Explicit Control**

```java
@Autowired
@Qualifier("primaryDataSource")
private DataSource dataSource;
```

4. **Refactor Large Constructors**

```java
//  Too many dependencies (design smell)
public UserService(Dep1 d1, Dep2 d2, Dep3 d3, Dep4 d4, Dep5 d5) { }

//  Refactor into smaller services
@Service
class UserService {
    private final UserDataService dataService;
    private final UserNotificationService notificationService;

    public UserService(UserDataService dataService,
                       UserNotificationService notificationService) {
        this.dataService = dataService;
        this.notificationService = notificationService;
    }
}
```

5. **Use Lombok for Cleaner Code** (Optional)

```java
@Service
@RequiredArgsConstructor  // Generates constructor for final fields
class UserService {
    private final UserRepository repository;
    private final EmailService emailService;
}
```

####  DON'Ts

1. **Avoid Field Injection in Production**

```java
//  Avoid this
@Autowired
private UserRepository repository;
```

2. **Don't Create Circular Dependencies**

```java
//  Bad design
@Component
class A {
    @Autowired private B b;
}

@Component
class B {
    @Autowired private A a;
}
```

3. **Don't Use `new` for Spring Beans**

```java
//  Wrong - bypasses Spring DI
UserService service = new UserService();

//  Correct - let Spring manage
@Autowired
private UserService service;
```

4. **Don't Mix Injection Styles**

```java
//  Inconsistent
@Component
class User {
    @Autowired
    private Order order;  // Field injection

    private Invoice invoice;

    public User(Invoice invoice) {  // Constructor injection
        this.invoice = invoice;
    }
}

// Consistent
@Component
class User {
    private final Order order;
    private final Invoice invoice;

    public User(Order order, Invoice invoice) {
        this.order = order;
        this.invoice = invoice;
    }
}
```

### Quick Reference Guide

#### When Application Fails to Start

| Error | Cause | Solution |
|-------|-------|----------|
| Circular dependency detected | Beans depend on each other | 1. Refactor<br>2. Use `@Lazy`<br>3. `@PostConstruct` |
| NoUniqueBeanDefinitionException | Multiple beans of same type | 1. Use `@Primary`<br>2. Use `@Qualifier` |
| NoSuchBeanDefinitionException | Bean not found | 1. Add `@Component`<br>2. Check component scan |
| BeanCreationException | Cannot create bean | 1. Check constructor<br>2. Verify dependencies exist |

### Annotations Cheat Sheet

```java
// Bean Definition
@Component          // Generic Spring bean
@Service            // Service layer
@Repository         // Data access layer
@Controller         // Web controller
@RestController     // REST API controller

// Dependency Injection
@Autowired          // Inject dependency
@Qualifier("name")  // Specify which bean
@Primary            // Default bean when multiple exist
@Lazy               // Lazy initialization

// Bean Lifecycle
@PostConstruct      // After dependency injection
@PreDestroy         // Before bean destruction

// Configuration
@Configuration      // Java-based configuration
@Bean               // Method-level bean definition
```

### Testing Examples

#### Constructor Injection (Easy Testing)

```java
@Service
class UserService {
    private final UserRepository repository;

    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}

// Test
@Test
void testUserService() {
    UserRepository mockRepo = mock(UserRepository.class);
    UserService service = new UserService(mockRepo);  // ✅ Clean

    service.getUser(1L);
    verify(mockRepo).findById(1L);
}
```

#### Field Injection (Requires Framework)

```java
@Service
class UserService {
    @Autowired
    private UserRepository repository;
}

// Test
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository repository;

    @InjectMocks  // Uses reflection
    private UserService service;

    @Test
    void testGetUser() {
        service.getUser(1L);
        verify(repository).findById(1L);
    }
}
```

---

## Additional Resources

### Related Topics to Explore

- **Bean Scopes**: Singleton, Prototype, Request, Session
- **Bean Lifecycle**: Initialization and destruction callbacks
- **Lazy Initialization**: `@Lazy` annotation in depth
- **Conditional Beans**: `@Conditional`, `@ConditionalOnProperty`
- **Profiles**: Environment-specific beans with `@Profile`
- **Java Config**: `@Configuration` and `@Bean`
- **Component Scanning**: `@ComponentScan` configuration

### Spring Documentation

- [Spring Framework Core Documentation](https://docs.spring.io/spring-framework/reference/core.html)
- [Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies.html)
- [Bean Scopes](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html)

---

## Contributing

Found an error or want to add more examples? Contributions are welcome!

---


---

**Happy Coding! 🚀**
