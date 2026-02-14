# Spring Boot Exception Handling –

Topics covered are as given below:

* Core concepts
* Internal architecture
* All important classes involved
* Resolver chaining mechanism
* Detailed decision flow
* Annotation-based handling
* Boot fallback mechanism
* Validation handling
* Enterprise patterns
* Common confusion scenarios
* Microservices considerations
* Interview questions with answers
* Final mental model

---

# Why Exception Handling Matters

Exception handling in Spring Boot ensures:

* Clean and consistent API responses
* Proper HTTP status mapping
* Centralized error handling
* Separation of concerns
* Secure error exposure (no stack traces leaked)
* Production-ready API contracts

Spring Boot builds on top of Spring MVC’s exception handling mechanism.

---

# 1. High-Level Request to Exception Flow

```
Client
   ↓
DispatcherServlet
   ↓
HandlerMapping
   ↓
Controller
   ↓ (Exception Thrown)
HandlerExceptionResolver Chain
   ↓
If resolved → Response returned
If not resolved → /error
   ↓
BasicErrorController
   ↓
JSON Response
```

The most important piece is the **HandlerExceptionResolver chain**.

---

# 2. Core Classes Involved in Exception Handling

## 2.1 DispatcherServlet

Package:

```
org.springframework.web.servlet
```

* Front controller in Spring MVC.
* Catches exceptions thrown by controllers.
* Delegates exception resolution to HandlerExceptionResolver.

---

## 2.2 HandlerExceptionResolver

```java
public interface HandlerExceptionResolver {
    ModelAndView resolveException(
        HttpServletRequest request,
        HttpServletResponse response,
        Object handler,
        Exception ex);
}
```

Core strategy interface for resolving exceptions.

---

# 3. HandlerExceptionResolverComposite (Chaining Mechanism)

Class:

```
org.springframework.web.servlet.handler.HandlerExceptionResolverComposite
```

* Holds all registered resolvers
* Executes them sequentially
* Stops at first resolver that returns non-null

Pseudo logic:

```java
for (HandlerExceptionResolver resolver : resolvers) {
    ModelAndView mav = resolver.resolveException(request, response, handler, ex);
    if (mav != null) {
        return mav;
    }
}
return null;
```

If all return null → exception unresolved → forwarded to `/error`.

Pattern used: **Chain of Responsibility**

---

# 4. Default Resolver Order

1. ExceptionHandlerExceptionResolver
2. ResponseStatusExceptionResolver
3. DefaultHandlerExceptionResolver

Order matters.

---

# 5. Detailed Internal Decision Process

## Step 1: ExceptionHandlerExceptionResolver

Handles:

* @ExceptionHandler
* @ControllerAdvice
* @RestControllerAdvice

Highest priority.

---

## Step 2: ResponseStatusExceptionResolver

Handles:

* @ResponseStatus
* ResponseStatusException

Maps exception → HTTP status.

---

## Step 3: DefaultHandlerExceptionResolver

Handles predefined Spring MVC exceptions:

* HttpRequestMethodNotSupportedException → 405
* MissingServletRequestParameterException → 400
* HttpMediaTypeNotSupportedException → 415
* HttpMessageNotReadableException → 400

---

## If None Resolve

Forwarded to `/error` → handled by Spring Boot.

---

# 6. Spring Boot Fallback Layer

## BasicErrorController

Package:

```
org.springframework.boot.autoconfigure.web.servlet.error
```

Handles `/error`.

---

## ErrorAttributes

```java
public interface ErrorAttributes {
    Map<String, Object> getErrorAttributes(
        WebRequest webRequest,
        ErrorAttributeOptions options);
}
```

---

## DefaultErrorAttributes

Builds default JSON response:

```json
{
  "timestamp": "...",
  "status": 500,
  "error": "Internal Server Error",
  "message": "...",
  "path": "/api/users"
}
```

Important:

DefaultErrorAttributes is NOT part of resolver chain.
It runs only after fallback to `/error`.

---

# 7. Annotation-Based Handling

## @ExceptionHandler

Controller-level.

## @ControllerAdvice

Global.

## @RestControllerAdvice

@ControllerAdvice + @ResponseBody.
Recommended for REST APIs.

---

# 8. Custom Exceptions

Define:

```java
public class ResourceNotFoundException extends RuntimeException {}
```

Throw and handle globally.

---

# 9. Standardized Error Response

```java
public class ErrorResponse {
    private LocalDateTime timestamp;
    private String message;
    private String errorCode;
    private String path;
}
```

---

# 10. Validation Handling

Exceptions involved:

* MethodArgumentNotValidException
* BindingResult
* FieldError
* ConstraintViolationException

---

# 11. ResponseStatusException

Programmatic way:

```java
throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Invalid Input");
```

---

# 12. @ResponseStatus

Static mapping to HTTP status.

---

# 13. ResponseEntityExceptionHandler (Enterprise)

Extend:

```
org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler
```

Override built-in methods.

---

# 14. Handling 404

Enable:

```
spring.mvc.throw-exception-if-no-handler-found=true
spring.web.resources.add-mappings=false
```

Handle NoHandlerFoundException.

---

# 15. Logging Strategy

* Log internally
* Do not expose stack traces
* Use SLF4J

---

# 16. Common Confusions

Covered earlier (signature mismatch, resolver order, missing @Valid, etc.)

---

# 17. Production Best Practices

* @RestControllerAdvice
* Standardized DTO
* Separate business/system exceptions
* Correlation IDs
* i18n
* Centralized logging

---

# 18. Microservices Considerations

* Consistent error contract
* Feign error decoder
* Avoid leaking internal errors

---

# 19. Complete Internal Mental Model

```
Controller
   ↓
Exception
   ↓
DispatcherServlet
   ↓
HandlerExceptionResolverComposite
   ↓
   1. ExceptionHandlerExceptionResolver
   2. ResponseStatusExceptionResolver
   3. DefaultHandlerExceptionResolver
   ↓
If unresolved
   ↓
Forward to /error
   ↓
BasicErrorController
   ↓
DefaultErrorAttributes
   ↓
HTTP Response
```

---

# Interview Questions WITH Answers

---

## Basic Level

### 1. What is @ControllerAdvice?

@ControllerAdvice is a specialization of @Component used to define global exception handling, data binding, and model attributes across multiple controllers. It allows centralized exception handling using @ExceptionHandler methods.

---

### 2. Difference between @ControllerAdvice and @RestControllerAdvice?

@RestControllerAdvice = @ControllerAdvice + @ResponseBody.

@ControllerAdvice returns views unless annotated with @ResponseBody.
@RestControllerAdvice directly returns JSON/XML responses.

Used mainly in REST APIs.

---

### 3. What is @ExceptionHandler?

It is an annotation used to handle specific exceptions within a controller or globally via @ControllerAdvice.

It defines a method that executes when a specific exception type is thrown.

---

### 4. How do you handle validation errors?

* Use @Valid on request body.
* Catch MethodArgumentNotValidException.
* Extract errors from BindingResult.
* Return structured error response.

---

### 5. What is ResponseStatusException?

A programmatic exception that allows setting HTTP status and reason dynamically.

Handled by ResponseStatusExceptionResolver.

---

## Intermediate Level

### 6. What is HandlerExceptionResolver?

A strategy interface used by DispatcherServlet to resolve exceptions into HTTP responses.

---

### 7. What are default resolvers in Spring MVC?

1. ExceptionHandlerExceptionResolver
2. ResponseStatusExceptionResolver
3. DefaultHandlerExceptionResolver

Executed in that order.

---

### 8. How does Spring Boot handle exceptions internally?

* Exception thrown
* DispatcherServlet delegates to HandlerExceptionResolverComposite
* Resolvers execute sequentially
* If none resolve → forward to /error
* BasicErrorController handles it
* DefaultErrorAttributes builds response

---

### 9. What is ResponseEntityExceptionHandler?

An abstract base class providing pre-implemented methods for handling common Spring MVC exceptions. Used in enterprise apps for centralized control.

---

### 10. How do you handle 404 globally?

Enable:

```
spring.mvc.throw-exception-if-no-handler-found=true
```

Handle NoHandlerFoundException in @ControllerAdvice.

---

## Advanced Level

### 11. How would you design exception handling in microservices?

* Centralized global handler
* Standardized error DTO
* Unique error codes
* Correlation IDs
* Avoid exposing internal details
* Map downstream exceptions properly
* Implement Feign error decoder

---

### 12. How do you standardize error responses?

Create a common ErrorResponse DTO with:

* timestamp
* message
* errorCode
* path
* possibly correlationId

Return it from global exception handler.

---

### 13. How do exceptions affect transaction rollback?

By default:

* RuntimeException → rollback
* Checked Exception → no rollback

Can configure using:

```java
@Transactional(rollbackFor = Exception.class)
```

---

### 14. How do you customize DefaultErrorAttributes?

Extend DefaultErrorAttributes and override getErrorAttributes().

Register it as a @Component.

---

### 15. What happens if multiple resolvers match?

Resolver chain executes sequentially.
The first resolver that returns non-null stops execution.

Order determines behavior.

---

### 16. How do you prevent stack trace leakage?

* Disable stacktrace in properties:

  ```
  server.error.include-stacktrace=never
  ```
* Do not expose exception.getMessage() directly
* Log internally
* Return sanitized error DTO

---

# Final Architecture Recommendation

For enterprise systems:

1. Use @RestControllerAdvice
2. Extend ResponseEntityExceptionHandler
3. Create custom business exceptions
4. Implement standardized error DTO
5. Log internally
6. Avoid leaking stack traces
7. Separate technical vs business exceptions

---
