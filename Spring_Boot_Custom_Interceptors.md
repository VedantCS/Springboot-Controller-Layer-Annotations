# Spring Boot Custom Interceptors -
> Covering HandlerInterceptor, Custom Annotations, and AOP-based Interceptors

---

## Table of Contents
- [1. Introduction to Interceptors](#1-introduction-to-interceptors)
- [2. Types of Interceptors](#2-types-of-interceptors)
- [3. Before Controller - HandlerInterceptor](#3-before-controller---handlerinterceptor)
  - [3.1 Creating Custom Interceptor](#31-creating-custom-interceptor)
  - [3.2 Registering Interceptor](#32-registering-interceptor)
  - [3.3 Execution Flow](#33-execution-flow)
  - [3.4 Methods Explained](#34-methods-explained)
- [4. After Controller - Custom Annotations](#4-after-controller---custom-annotations)
  - [4.1 Creating Custom Annotations](#41-creating-custom-annotations)
  - [4.2 Meta Annotations](#42-meta-annotations)
  - [4.3 Annotation Fields](#43-annotation-fields)
  - [4.4 AOP-based Interceptor](#44-aop-based-interceptor)
- [5. Complete Examples](#5-complete-examples)
- [6. Comparison Table](#6-comparison-table)
- [7. Use Cases](#7-use-cases)
- [8. Best Practices](#8-best-practices)
- [9. Common Pitfalls](#9-common-pitfalls)
- [10. Quick Reference](#10-quick-reference)

---

## 1. Introduction to Interceptors

### What are Interceptors?

**Interceptors** allow you to intercept HTTP requests and responses **before** they reach the controller or **after** controller execution.

### Request Flow in Spring Boot

```
Client Request
    ↓
Servlet Container (Tomcat/Jetty)
    ↓
DispatcherServlet
    ↓
[HandlerInterceptor - preHandle()]  ← First Interception Point
    ↓
Controller Method
    ↓
[HandlerInterceptor - postHandle()]
    ↓
View Rendering (if any)
    ↓
[HandlerInterceptor - afterCompletion()]
    ↓
Response to Client
```

### Why Use Interceptors?

| Use Case | Description |
|----------|-------------|
| **Authentication** | Check if user is logged in before accessing protected resources |
| **Authorization** | Verify user permissions |
| **Logging** | Log request/response details |
| **Performance Monitoring** | Track execution time |
| **Request Validation** | Validate headers, tokens, etc. |
| **Response Modification** | Add custom headers to responses |

---

## 2. Types of Interceptors

### Two Main Approaches

| Type | When It Intercepts | Implementation |
|------|-------------------|----------------|
| **Before Controller** | Before request reaches controller | `HandlerInterceptor` |
| **After Controller** | After controller invocation (method level) | Custom Annotation + AOP |

### Visual Comparison

```
┌─────────────────────────────────────────────────────────┐
│                    REQUEST FLOW                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Client → DispatcherServlet                            │
│              ↓                                          │
│         [HandlerInterceptor]  ← Type 1 (Before)        │
│              ↓                                          │
│         Controller Method                               │
│              ↓                                          │
│         [AOP Interceptor]  ← Type 2 (After)            │
│              ↓                                          │
│         Business Logic                                  │
│              ↓                                          │
│         Response                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Before Controller - HandlerInterceptor

### 3.1 Creating Custom Interceptor

**Purpose**: Intercept requests **before** they reach the controller.

#### Step 1: Create Interceptor Class

```java
package com.example.interceptor;

import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

@Component
public class MyCustomInterceptor implements HandlerInterceptor {

    /**
     * Executed BEFORE controller method
     * Return true: Continue to controller
     * Return false: Stop request processing
     */
    @Override
    public boolean preHandle(
            HttpServletRequest request, 
            HttpServletResponse response, 
            Object handler) throws Exception {

        System.out.println("preHandle: Before controller execution");
        System.out.println("Request URL: " + request.getRequestURL());

        // Authentication check example
        String authToken = request.getHeader("Authorization");
        if (authToken == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;  // Stop processing
        }

        return true;  // Continue to controller
    }

    /**
     * Executed AFTER controller method (only on success)
     * Before view rendering
     */
    @Override
    public void postHandle(
            HttpServletRequest request, 
            HttpServletResponse response, 
            Object handler, 
            ModelAndView modelAndView) throws Exception {

        System.out.println("postHandle: After controller execution");
        // Can modify ModelAndView here
    }

    /**
     * Executed AFTER complete request processing
     * Always executed (even on exception)
     * Similar to 'finally' block
     */
    @Override
    public void afterCompletion(
            HttpServletRequest request, 
            HttpServletResponse response, 
            Object handler, 
            Exception ex) throws Exception {

        System.out.println("afterCompletion: After everything (cleanup)");

        if (ex != null) {
            System.out.println("Exception occurred: " + ex.getMessage());
        }

        // Cleanup resources, logging, etc.
    }
}
```

---

### 3.2 Registering Interceptor

#### Step 2: Create Configuration Class

```java
package com.example.config;

import com.example.interceptor.MyCustomInterceptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class AppConfig implements WebMvcConfigurer {

    @Autowired
    private MyCustomInterceptor myCustomInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(myCustomInterceptor)
                .addPathPatterns("/api/**")           // Include patterns
                .excludePathPatterns(                 // Exclude patterns
                    "/api/update-user",
                    "/api/delete-user"
                );
    }
}
```

#### Path Patterns Explained

```java
// Apply to all URLs starting with /api/
.addPathPatterns("/api/**")

// Apply to specific URLs
.addPathPatterns("/api/user", "/api/product")

// Apply to all URLs
.addPathPatterns("/**")

// Exclude specific URLs
.excludePathPatterns("/api/public/**", "/api/health")
```

#### Complete Registration Example

```java
@Configuration
public class AppConfig implements WebMvcConfigurer {

    @Autowired
    private MyCustomInterceptor myCustomInterceptor;

    @Autowired
    private AuthInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // Interceptor 1: Auth check for all APIs
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/login", "/api/register");

        // Interceptor 2: Logging for specific APIs
        registry.addInterceptor(myCustomInterceptor)
                .addPathPatterns("/api/user/**", "/api/product/**");

        // Multiple interceptors execute in order of registration
    }
}
```

---

### 3.3 Execution Flow

#### Example Controller

```java
@RestController
@RequestMapping("/api")
public class UserController {

    @GetMapping("/get-user")
    public String getUser() {
        System.out.println("Controller: Executing getUser()");
        return "User data";
    }
}
```

#### Request: GET /api/get-user

**Console Output:**

```
preHandle: Before controller execution
Request URL: http://localhost:8080/api/get-user

Controller: Executing getUser()

postHandle: After controller execution

afterCompletion: After everything (cleanup)
```

#### With Exception in Controller

```java
@GetMapping("/get-user")
public String getUser() {
    System.out.println("Controller: Executing getUser()");
    throw new RuntimeException("Something went wrong!");
}
```

**Console Output:**

```
preHandle: Before controller execution
Request URL: http://localhost:8080/api/get-user

Controller: Executing getUser()

[postHandle NOT EXECUTED - because exception occurred]

afterCompletion: After everything (cleanup)
Exception occurred: Something went wrong!
```

---

### 3.4 Methods Explained

#### Comparison Table

| Method | When Executed | Executes on Exception? | Return Type | Purpose |
|--------|---------------|------------------------|-------------|---------|
| **preHandle()** | Before controller | N/A | `boolean` | Authentication, validation |
| **postHandle()** | After controller (success) | ❌ NO | `void` | Modify response, add data |
| **afterCompletion()** | After everything | ✅ YES | `void` | Cleanup, logging |

#### Visual Flow

```
SUCCESS CASE:
─────────────
preHandle() → Controller → postHandle() → afterCompletion()
                                          

EXCEPTION CASE:
───────────────
preHandle() → Controller → [Exception] → afterCompletion()
                                         
                         [postHandle skipped]
```

#### Detailed Example

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {

    private static final Logger logger = LoggerFactory.getLogger(LoggingInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        long startTime = System.currentTimeMillis();
        request.setAttribute("startTime", startTime);

        logger.info("Request URL: {}", request.getRequestURL());
        logger.info("HTTP Method: {}", request.getMethod());

        // Check authentication
        String token = request.getHeader("Authorization");
        if (token == null || !isValidToken(token)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;  // Stop processing
        }

        return true;  // Continue
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, 
                          Object handler, ModelAndView modelAndView) {
        logger.info("Response Status: {}", response.getStatus());
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, 
                               Object handler, Exception ex) {
        long startTime = (Long) request.getAttribute("startTime");
        long endTime = System.currentTimeMillis();
        long executeTime = endTime - startTime;

        logger.info("Request completed in {} ms", executeTime);

        if (ex != null) {
            logger.error("Exception: {}", ex.getMessage());
        }
    }

    private boolean isValidToken(String token) {
        // Token validation logic
        return token.startsWith("Bearer ");
    }
}
```

---

## 4. After Controller - Custom Annotations

### 4.1 Creating Custom Annotations

**Purpose**: Create your own annotations like `@GetMapping`, `@Autowired`, etc.

#### Basic Syntax

```java
@interface AnnotationName {
    // Annotation body
}
```

#### Simple Example

```java
package com.example.annotation;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyCustomAnnotation {
    // This is a custom annotation
}
```

**Usage:**

```java
@RestController
public class UserController {

    @MyCustomAnnotation  // Using our custom annotation
    @GetMapping("/get-user")
    public String getUser() {
        return "User data";
    }
}
```

---

### 4.2 Meta Annotations

**Meta Annotation** = Annotation applied **on** an annotation.

#### Two Most Important Meta Annotations

### 1. `@Target` - Where Can Annotation Be Used?

**Defines WHERE the annotation can be applied.**

```java
@Target(ElementType.METHOD)
public @interface MyAnnotation {
    // Can only be used on methods
}
```

#### Available ElementTypes

| ElementType | Where It Can Be Used | Example |
|-------------|---------------------|---------|
| `METHOD` | On methods only | `@GetMapping` |
| `TYPE` | On classes, interfaces, enums | `@RestController` |
| `FIELD` | On class fields | `@Autowired` |
| `PARAMETER` | On method parameters | `@RequestParam` |
| `CONSTRUCTOR` | On constructors | Custom validation |
| `LOCAL_VARIABLE` | On local variables | Rarely used |
| `ANNOTATION_TYPE` | On other annotations | Meta-annotations |
| `PACKAGE` | On packages | Rarely used |

#### Multiple Targets

```java
@Target({ElementType.METHOD, ElementType.FIELD})
public @interface MyAnnotation {
    // Can be used on methods AND fields
}
```

**Usage Examples:**

```java
// Valid - on method
@MyAnnotation
public void doSomething() { }

// Valid - on field
@MyAnnotation
private String name;

// Invalid - on class (will cause compilation error)
@MyAnnotation  // Error!
public class MyClass { }
```

---

### 2. `@Retention` - How Long Annotation Lives?

**Defines HOW LONG the annotation information is retained.**

#### Three Retention Policies

| Policy | Description | Available At | Use Case |
|--------|-------------|--------------|----------|
| `SOURCE` | Discarded by compiler | Source code only | Code hints (e.g., `@Override`) |
| `CLASS` | In .class file, ignored by JVM | Compile time | Build tools, bytecode processing |
| `RUNTIME` | Available during runtime | Runtime via Reflection | Most Spring annotations, AOP |

#### Visual Explanation

```
SOURCE
──────
.java file → @Override present
             ↓ [javac compile]
.class file → @Override REMOVED 
             ↓ [JVM runtime]
Runtime     → @Override NOT available 


CLASS
─────
.java file → @MyAnnotation present
             ↓ [javac compile]
.class file → @MyAnnotation present 
             ↓ [JVM runtime]
Runtime     → @MyAnnotation IGNORED (not accessible) 


RUNTIME
───────
.java file → @MyAnnotation present
             ↓ [javac compile]
.class file → @MyAnnotation present 
             ↓ [JVM runtime]
Runtime     → @MyAnnotation AVAILABLE  (can be read via reflection)
```

#### Example 1: SOURCE (Like @Override)

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface MyCustomAnnotation {
}
```

**Java File:**
```java
public class User {
    @MyCustomAnnotation
    public void updateUser() {
        // Code
    }
}
```

**After Compilation (.class file):**
```java
public class User {
    // @MyCustomAnnotation is GONE! 
    public void updateUser() {
        // Code
    }
}
```

**At Runtime:**
```java
// Cannot detect @MyCustomAnnotation
Method method = User.class.getMethod("updateUser");
boolean hasAnnotation = method.isAnnotationPresent(MyCustomAnnotation.class);
System.out.println(hasAnnotation);  // false 
```

---

#### Example 2: CLASS

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.CLASS)
public @interface MyCustomAnnotation {
}
```

**Java File:**
```java
public class User {
    @MyCustomAnnotation
    public void updateUser() { }
}
```

**After Compilation (.class file):**
```java
public class User {
    @MyCustomAnnotation  // Still present in bytecode 
    public void updateUser() { }
}
```

**At Runtime:**
```java
// JVM ignores it - cannot access
Method method = User.class.getMethod("updateUser");
boolean hasAnnotation = method.isAnnotationPresent(MyCustomAnnotation.class);
System.out.println(hasAnnotation);  // false  (ignored by JVM)
```

---

#### Example 3: RUNTIME (Most Common for Spring)

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)  // Available at runtime
public @interface MyCustomAnnotation {
}
```

**Java File:**
```java
public class User {
    @MyCustomAnnotation
    public void updateUser() { }
}
```

**After Compilation (.class file):**
```java
public class User {
    @MyCustomAnnotation  // Present 
    public void updateUser() { }
}
```

**At Runtime:**
```java
// Can detect and process annotation 
Method method = User.class.getMethod("updateUser");
boolean hasAnnotation = method.isAnnotationPresent(MyCustomAnnotation.class);
System.out.println(hasAnnotation);  // true 

// Can read annotation details
MyCustomAnnotation annotation = method.getAnnotation(MyCustomAnnotation.class);
// Process annotation...
```

---

### 4.3 Annotation Fields

**Annotations can have fields (methods that act like fields) to pass values.**

#### Rules for Annotation Fields

| Rule | Description |
|------|-------------|
| **No parameters** | Methods cannot have parameters |
| **No body** | Methods cannot have implementation |
| **Restricted return types** | Only: primitives, String, Class, enum, annotations, arrays of these |
| **Default values** | Can provide default values |

#### Allowed Return Types

```java
public @interface MyAnnotation {
    //  Primitives (8 types)
    int intValue();
    boolean boolValue();
    double doubleValue();
    // ... other primitives

    //  String
    String stringValue();

    //  Class type
    Class<?> classType();

    //  Enum
    Priority priority();

    //  Another annotation
    MyOtherAnnotation nested();

    //  Array of above types
    String[] tags();
    int[] numbers();

    //  NOT ALLOWED
    // List<String> list();  // Collections not allowed
    // Map<String, String> map();  // Maps not allowed
    // Object obj();  // Object not allowed
}
```

---

#### Example: Annotation with Fields

```java
package com.example.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyCustomAnnotation {

    // Field 1: String with default value
    String name() default "";

    // Field 2: int with default value
    int priority() default 0;

    // Field 3: Class type
    Class<?> type() default String.class;

    // Field 4: Enum
    LogLevel logLevel() default LogLevel.INFO;

    // Field 5: String array
    String[] tags() default {};
}

// Enum for logLevel
enum LogLevel {
    DEBUG, INFO, WARN, ERROR
}
```

---

#### Using Annotation with Fields

```java
@RestController
public class UserController {

    // Example 1: Using default values
    @MyCustomAnnotation
    @GetMapping("/user1")
    public String getUser1() {
        return "User 1";
    }

    // Example 2: Providing custom values
    @MyCustomAnnotation(
        name = "user-key",
        priority = 10,
        type = User.class,
        logLevel = LogLevel.DEBUG,
        tags = {"important", "user-api"}
    )
    @GetMapping("/user2")
    public String getUser2() {
        return "User 2";
    }

    // Example 3: Partial values
    @MyCustomAnnotation(name = "user-endpoint")
    @GetMapping("/user3")
    public String getUser3() {
        return "User 3";
    }
}
```

---

#### Complete Example with Multiple Fields

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface CacheConfig {

    String key();  // Required (no default)

    int ttl() default 300;  // Time to live in seconds

    boolean enabled() default true;

    CacheType type() default CacheType.REDIS;

    String[] tags() default {};
}

enum CacheType {
    REDIS, MEMCACHED, IN_MEMORY
}
```

**Usage:**

```java
@Service
public class UserService {

    @CacheConfig(
        key = "user",
        ttl = 600,
        enabled = true,
        type = CacheType.REDIS,
        tags = {"user", "critical"}
    )
    public User getUserDetails(String userId) {
        // Method implementation
    }
}
```

---

### 4.4 AOP-based Interceptor

**Purpose**: Intercept methods annotated with custom annotation.

#### Complete Example

**Step 1: Create Custom Annotation**

```java
package com.example.annotation;

import java.lang.annotation.*;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyCustomAnnotation {
    String name() default "";
}
```

**Step 2: Create Service with Annotation**

```java
package com.example.service;

import com.example.annotation.MyCustomAnnotation;
import org.springframework.stereotype.Component;

@Component
public class User {

    @MyCustomAnnotation(name = "user")
    public String getUserDetails() {
        System.out.println("Getting user details...");
        return "User data";
    }
}
```

**Step 3: Create Controller**

```java
package com.example.controller;

import com.example.service.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api")
public class UserController {

    @Autowired
    private User user;

    @GetMapping("/get-user")
    public String getUser() {
        return user.getUserDetails();
    }
}
```

**Step 4: Create AOP Interceptor**

```java
package com.example.interceptor;

import com.example.annotation.MyCustomAnnotation;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

@Aspect
@Component
public class MyCustomInterceptor {

    @Around("@annotation(com.example.annotation.MyCustomAnnotation)")
    public Object interceptMethod(ProceedingJoinPoint joinPoint) throws Throwable {

        // Get method information
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();

        // Check if annotation is present
        if (method.isAnnotationPresent(MyCustomAnnotation.class)) {

            // Get annotation details
            MyCustomAnnotation annotation = method.getAnnotation(MyCustomAnnotation.class);
            String name = annotation.name();

            System.out.println("Intercepted method with annotation name: " + name);
        }

        // Do something BEFORE actual method
        System.out.println("Do something before actual method");
        long startTime = System.currentTimeMillis();

        // Execute actual method
        Object result = joinPoint.proceed();

        // Do something AFTER actual method
        long executionTime = System.currentTimeMillis() - startTime;
        System.out.println("Do something after actual method");
        System.out.println("Execution time: " + executionTime + " ms");

        return result;
    }
}
```

---

#### Execution Flow

**Request:** `GET /api/get-user`

**Console Output:**

```
Intercepted method with annotation name: user
Do something before actual method

Getting user details...

Do something after actual method
Execution time: 15 ms
```

---

#### Detailed Flow Diagram

```
1. Client calls: /api/get-user
   ↓
2. UserController.getUser()
   ↓
3. user.getUserDetails()  ← Method has @MyCustomAnnotation
   ↓
4. AOP INTERCEPTS (because of @annotation pointcut)
   ↓
5. MyCustomInterceptor.interceptMethod() starts
   ↓
6. Check if annotation present → YES 
   ↓
7. Read annotation.name() → "user"
   ↓
8. Print: "Intercepted method with annotation name: user"
   ↓
9. Print: "Do something before actual method"
   ↓
10. joinPoint.proceed() → Execute ACTUAL method
    ↓
11. Print: "Getting user details..."
    ↓
12. Return: "User data"
    ↓
13. Back to interceptor (after proceed)
    ↓
14. Print: "Do something after actual method"
    ↓
15. Return result to controller
    ↓
16. Response sent to client
```

---

## 5. Complete Examples

### Example 1: Authentication Interceptor

**Scenario**: Check authentication token before allowing access.

```java
@Component
public class AuthenticationInterceptor implements HandlerInterceptor {

    @Autowired
    private AuthService authService;

    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {

        String token = request.getHeader("Authorization");

        if (token == null || token.isEmpty()) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("Missing authentication token");
            return false;  // Stop request
        }

        if (!authService.validateToken(token)) {
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.getWriter().write("Invalid token");
            return false;  // Stop request
        }

        // Store user info in request for later use
        String userId = authService.getUserIdFromToken(token);
        request.setAttribute("userId", userId);

        return true;  // Allow request
    }
}
```

**Configuration:**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private AuthenticationInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/secure/**")
                .excludePathPatterns("/api/login", "/api/register");
    }
}
```

---

### Example 2: Logging Interceptor with Custom Annotation

**Custom Annotation:**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecution {
    String value() default "";
    LogLevel level() default LogLevel.INFO;
}

enum LogLevel {
    DEBUG, INFO, WARN, ERROR
}
```

**AOP Interceptor:**

```java
@Aspect
@Component
@Slf4j
public class LoggingInterceptor {

    @Around("@annotation(com.example.annotation.LogExecution)")
    public Object logExecution(ProceedingJoinPoint joinPoint) throws Throwable {

        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        LogExecution annotation = method.getAnnotation(LogExecution.class);

        String methodName = method.getName();
        String logMessage = annotation.value();
        LogLevel level = annotation.level();

        // Log before execution
        long startTime = System.currentTimeMillis();
        log.info("Executing {}: {}", methodName, logMessage);

        Object result;
        try {
            result = joinPoint.proceed();

            // Log success
            long duration = System.currentTimeMillis() - startTime;
            log.info("Completed {}: {} in {} ms", methodName, logMessage, duration);

        } catch (Exception e) {
            // Log error
            log.error("Error in {}: {} - {}", methodName, logMessage, e.getMessage());
            throw e;
        }

        return result;
    }
}
```

**Usage:**

```java
@Service
public class UserService {

    @LogExecution(value = "Fetching user data", level = LogLevel.INFO)
    public User getUser(String userId) {
        // Implementation
    }

    @LogExecution(value = "Updating user profile", level = LogLevel.WARN)
    public void updateUser(User user) {
        // Implementation
    }
}
```

---

### Example 3: Caching Interceptor

**Custom Annotation:**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Cacheable {
    String key();
    int ttl() default 300;  // seconds
}
```

**AOP Interceptor:**

```java
@Aspect
@Component
public class CachingInterceptor {

    @Autowired
    private CacheService cacheService;

    @Around("@annotation(com.example.annotation.Cacheable)")
    public Object handleCaching(ProceedingJoinPoint joinPoint) throws Throwable {

        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        Cacheable cacheable = method.getAnnotation(Cacheable.class);

        String cacheKey = cacheable.key();
        int ttl = cacheable.ttl();

        // Check cache first
        Object cachedValue = cacheService.get(cacheKey);
        if (cachedValue != null) {
            System.out.println("Cache hit for key: " + cacheKey);
            return cachedValue;
        }

        System.out.println("Cache miss for key: " + cacheKey);

        // Execute method
        Object result = joinPoint.proceed();

        // Store in cache
        cacheService.put(cacheKey, result, ttl);

        return result;
    }
}
```

**Usage:**

```java
@Service
public class ProductService {

    @Cacheable(key = "product:all", ttl = 600)
    public List<Product> getAllProducts() {
        // Expensive database query
        return productRepository.findAll();
    }

    @Cacheable(key = "product:{id}", ttl = 300)
    public Product getProductById(String id) {
        return productRepository.findById(id);
    }
}
```

---

### Example 4: Performance Monitoring

**Custom Annotation:**

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MonitorPerformance {
    String operation() default "";
    int threshold() default 1000;  // milliseconds
}
```

**AOP Interceptor:**

```java
@Aspect
@Component
@Slf4j
public class PerformanceInterceptor {

    @Around("@annotation(com.example.annotation.MonitorPerformance)")
    public Object monitorPerformance(ProceedingJoinPoint joinPoint) throws Throwable {

        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        MonitorPerformance annotation = method.getAnnotation(MonitorPerformance.class);

        String operation = annotation.operation();
        int threshold = annotation.threshold();

        long startTime = System.currentTimeMillis();

        Object result = joinPoint.proceed();

        long executionTime = System.currentTimeMillis() - startTime;

        log.info("Operation '{}' took {} ms", operation, executionTime);

        if (executionTime > threshold) {
            log.warn("SLOW OPERATION: '{}' exceeded threshold ({} ms > {} ms)", 
                    operation, executionTime, threshold);
            // Send alert, metrics, etc.
        }

        return result;
    }
}
```

**Usage:**

```java
@Service
public class ReportService {

    @MonitorPerformance(operation = "Generate Sales Report", threshold = 2000)
    public Report generateSalesReport(Date startDate, Date endDate) {
        // Heavy computation
    }

    @MonitorPerformance(operation = "Export Data", threshold = 5000)
    public void exportToExcel(List<Data> data) {
        // Time-consuming export
    }
}
```

---

## 6. Comparison Table

### HandlerInterceptor vs AOP Interceptor

| Aspect | HandlerInterceptor | AOP Interceptor |
|--------|-------------------|-----------------|
| **When to use** | Before controller | After controller (method level) |
| **Intercepts** | HTTP requests | Method calls |
| **Granularity** | URL patterns | Specific methods (via annotations) |
| **Access to** | Request, Response, Handler | Method arguments, return value |
| **Implementation** | `HandlerInterceptor` interface | `@Aspect` + `@Around/@Before/@After` |
| **Registration** | `WebMvcConfigurer` | Automatic (via `@Aspect`) |
| **Use cases** | Auth, logging requests | Caching, transactions, monitoring |

---

## 7. Use Cases

### When to Use HandlerInterceptor

 **Authentication/Authorization**
```java
// Check if user is logged in before accessing any API
```

 **Request Logging**
```java
// Log all incoming requests with URL, method, headers
```

 **CORS Handling**
```java
// Add CORS headers to responses
```

 **Rate Limiting**
```java
// Check if user exceeded request limits
```

 **Request Validation**
```java
// Validate common headers, tokens
```

---

### When to Use AOP Interceptor

 **Method-level Caching**
```java
// Cache specific method results
```

 **Transaction Management**
```java
// @Transactional implementation
```

 **Performance Monitoring**
```java
// Track execution time of specific methods
```

 **Retry Logic**
```java
// Retry failed operations
```

 **Audit Logging**
```java
// Log who called what method with what parameters
```

---

## 8. Best Practices

###  DO's

1. **Return true in preHandle() to continue**
   ```java
   @Override
   public boolean preHandle(...) {
       // Your logic
       return true;  // Always return true to continue
   }
   ```

2. **Use specific path patterns**
   ```java
   .addPathPatterns("/api/secure/**")  // Specific
   // vs
   .addPathPatterns("/**")  // Too broad
   ```

3. **Clean up resources in afterCompletion()**
   ```java
   @Override
   public void afterCompletion(...) {
       // Close connections, release locks, etc.
   }
   ```

4. **Use @Retention(RUNTIME) for custom annotations**
   ```java
   @Retention(RetentionPolicy.RUNTIME)  // Required for AOP
   ```

5. **Provide default values in annotations**
   ```java
   String name() default "";  // User doesn't need to provide
   ```

---

###  DON'Ts

1. **Don't forget to return in preHandle()**
   ```java
   //  Bad
   public boolean preHandle(...) {
       // No return statement - compilation error
   }
   ```

2. **Don't use RetentionPolicy.SOURCE for AOP**
   ```java
   //  Bad - Won't work at runtime
   @Retention(RetentionPolicy.SOURCE)
   ```

3. **Don't put heavy logic in interceptors**
   ```java
   //  Bad
   public boolean preHandle(...) {
       // Heavy database query
       // Complex computation
       return true;
   }
   ```

4. **Don't forget to call proceed() in @Around**
   ```java
   //  Bad
   @Around("...")
   public Object intercept(ProceedingJoinPoint jp) {
       // Forgot to call jp.proceed()
       return null;  // Method never executes!
   }
   ```

---

## 9. Common Pitfalls

### Issue 1: Interceptor Not Executing

**Problem**: HandlerInterceptor not being called.

**Solutions**:
1.  Check `@Component` on interceptor class
2.  Verify path patterns in configuration
3.  Ensure `WebMvcConfigurer` is in a `@Configuration` class
4.  Check if URL matches pattern

---

### Issue 2: AOP Interceptor Not Working

**Problem**: Custom annotation interceptor not executing.

**Solutions**:
1.  Check `@Retention(RetentionPolicy.RUNTIME)`
2.  Verify `@Aspect` and `@Component` on interceptor
3.  Ensure `spring-boot-starter-aop` dependency is added
4.  Check pointcut expression matches annotation path

---

### Issue 3: preHandle() Returns false But Processing Continues

**Problem**: Returning `false` doesn't stop processing.

**Cause**: Another interceptor in chain returned `true`.

**Solution**: Check interceptor order and ensure all return appropriate values.

---

## 10. Quick Reference

### HandlerInterceptor Template

```java
@Component
public class MyInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {
        // Before controller
        return true;  // Continue
    }

    @Override
    public void postHandle(HttpServletRequest request, 
                          HttpServletResponse response, 
                          Object handler, 
                          ModelAndView modelAndView) throws Exception {
        // After controller (success only)
    }

    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) throws Exception {
        // Always executed
    }
}
```

### Configuration Template

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private MyInterceptor myInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(myInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/public/**");
    }
}
```

### Custom Annotation Template

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
    String value() default "";
}
```

### AOP Interceptor Template

```java
@Aspect
@Component
public class MyAspect {

    @Around("@annotation(com.example.annotation.MyAnnotation)")
    public Object intercept(ProceedingJoinPoint joinPoint) throws Throwable {
        // Before
        Object result = joinPoint.proceed();
        // After
        return result;
    }
}
```

---

## Summary

### Key Takeaways

1. **Two Types of Interceptors**:
   - `HandlerInterceptor` - Before controller
   - AOP with custom annotations - After controller

2. **HandlerInterceptor Methods**:
   - `preHandle()` - Before controller
   - `postHandle()` - After controller (success only)
   - `afterCompletion()` - Always (like finally)

3. **Custom Annotations**:
   - Created with `@interface`
   - Require `@Target` and `@Retention`
   - Use `RetentionPolicy.RUNTIME` for AOP

4. **Common Use Cases**:
   - Authentication
   - Logging
   - Caching
   - Performance monitoring
   - Transaction management

---

## Additional Resources

- [Spring HandlerInterceptor Documentation](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html)
- [Spring AOP Documentation](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [Java Annotations Tutorial](https://docs.oracle.com/javase/tutorial/java/annotations/)

