
---

#  Spring MVC  Notes

## 1. What is Spring MVC?

**Spring MVC** is a **web framework** in the Spring ecosystem based on the **Model–View–Controller (MVC)** design pattern.

It is used to build:

* Web applications (HTML pages)
* RESTful web services (with JSON/XML)

---

## 2. MVC Architecture
 <img width="295" height="171" alt="image" src="https://github.com/user-attachments/assets/ae4d5277-6328-4480-ab53-ceb94f56e72d" />

### MVC Components

### 1️ Model

* Holds **data**
* Represents business objects
* Passed from controller to view

```java
model.addAttribute("user", user);
```

---

### 2️ View

* Responsible for **presentation**
* Technologies: Thymeleaf, JSP, FreeMarker
* Renders HTML

Example:

```html
<p th:text="${user.name}"></p>
```

---

### 3️ Controller

* Handles HTTP requests
* Interacts with service layer
* Returns view name or data

--- 

---
Summary in short: 
# Core Architecture Components
The framework revolves around a central "Front Controller" called the DispatcherServlet, which manages the entire request-handling lifecycle. 

   ## Model:
    Encapsulates application data, often using POJOs (Plain Old Java Objects) or entities.
 ## View:
    Responsible for rendering the UI, transforming model data into formats like HTML, JSON, or XML using technologies like JSP, Thymeleaf, or FreeMarker.

  ## Controller:
    Contains the business logic, processes user requests via annotations like @Controller, and returns a ModelAndView or view name to the dispatcher.
    HandlerMapping: Maps incoming HTTP requests to specific controller methods.
   
 ## ViewResolver:
    Translates logical view names (e.g., "home") into physical view resources (e.g., /WEB-INF/views/home.jsp). 

# Key Features:

 ## Annotation-Driven: 
  Uses annotations such as @RequestMapping, @GetMapping, and @PostMapping to simplify configuration and URL mapping
  
 ## Flexible Data Binding:
  Automatically populates Java objects from request parameters and performs validation using frameworks like Hibernate Validator.
   
##  REST Support:
  Since version 3.0, it provides extensive support for building RESTful web services using @RestController and @ResponseBody.
  Content Negotiation: Can serve different representations of the same data (JSON, XML, HTML) based on the client's Accept header. 



---


## 3. Spring MVC Request Flow (Very Important)

1. Client sends HTTP request
2. **DispatcherServlet** receives request (Front Controller)
3. DispatcherServlet consults **HandlerMapping**
4. Controller method is invoked
5. Controller returns:

   * View name (MVC)
   * Data (REST)
6. **ViewResolver** resolves view
7. Response sent to client

---

## 4. DispatcherServlet (Core Component)

* Acts as **Front Controller**
* Central entry point
* Manages entire request lifecycle

```text
Client → DispatcherServlet → Controller → View → Client
```

---

## 5. Controllers in Spring MVC

### `@Controller`

* Used for **web pages**
* Returns view names

```java
@Controller
public class HomeController {
    @GetMapping("/home")
    public String home() {
        return "home";
    }
}
```

---

### `@RestController`

* Used for **REST APIs**
* Returns JSON/XML
* No view rendering

```java
@RestController
public class UserController {
    @GetMapping("/user")
    public User getUser() {
        return new User("Alice");
    }
}
```

Internally:

```java
@RestController = @Controller + @ResponseBody
```

---

## 6. Request Mapping Annotations

### Common Mapping Annotations

| Annotation        | Purpose         |
| ----------------- | --------------- |
| `@RequestMapping` | Generic mapping |
| `@GetMapping`     | HTTP GET        |
| `@PostMapping`    | HTTP POST       |
| `@PutMapping`     | HTTP PUT        |
| `@DeleteMapping`  | HTTP DELETE     |

### Example

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable int id) {
    return userService.findById(id);
}
```

---

## 7. Request Data Handling

### Path Variable

```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable int id) {}
```

### Request Parameter

```java
@GetMapping("/search")
public String search(@RequestParam String keyword) {}
```

### Request Body (JSON → Object)

```java
@PostMapping("/users")
public User save(@RequestBody User user) {}
```

---

## 8. Model and ModelAndView

### Model

```java
model.addAttribute("name", "John");
```

### ModelAndView

```java
ModelAndView mv = new ModelAndView("home");
mv.addObject("name", "John");
return mv;
```

---

## 9. View Resolver

Maps view names to actual files.

### Example (Thymeleaf – Spring Boot)

```text
return "home";
→ /templates/home.html
```

---

## 10. Form Handling

### HTML Form

```html
<form action="/save" method="post">
    <input name="name"/>
    <button>Submit</button>
</form>
```

### Controller

```java
@PostMapping("/save")
public String save(@ModelAttribute User user) {
    return "success";
}
```

---

## 11. Validation

### Using Bean Validation

```java
@NotNull
@Size(min = 3)
private String name;
```

### Controller

```java
@PostMapping("/save")
public String save(@Valid User user, BindingResult result) {
    if (result.hasErrors()) {
        return "form";
    }
    return "success";
}
```

---

## 12. Exception Handling

### Local Exception Handling

```java
@ExceptionHandler(Exception.class)
public String handleException() {
    return "error";
}
```

---

### Global Exception Handling

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handle(Exception ex) {
        return ResponseEntity.badRequest().body(ex.getMessage());
    }
}
```

---

## 13. RESTful Web Services in Spring MVC

### HTTP Status Codes

```java
return ResponseEntity.status(HttpStatus.CREATED).body(user);
```

### Content Negotiation

* JSON
* XML

Handled by **HttpMessageConverters**

---

## 14. Spring MVC vs Spring Boot MVC

| Feature          | Spring MVC    | Spring Boot |
| ---------------- | ------------- | ----------- |
| Configuration    | Manual        | Auto        |
| XML needed       | Yes (earlier) | No          |
| Setup            | Complex       | Easy        |
| Production ready | Needs setup   | Built-in    |

---

## 15. Annotations Summary

| Annotation          | Purpose          |
| ------------------- | ---------------- |
| `@Controller`       | Web controller   |
| `@RestController`   | REST controller  |
| `@RequestMapping`   | URL mapping      |
| `@PathVariable`     | URL data         |
| `@RequestParam`     | Query param      |
| `@RequestBody`      | JSON input       |
| `@ResponseBody`     | Return data      |
| `@ModelAttribute`   | Form binding     |
| `@ControllerAdvice` | Global exception |

---

## 16. Advantages of Spring MVC

 Clear separation of concerns
 Powerful annotation-based programming
 Flexible view technologies
 REST API support
 Easy integration with Spring ecosystem

---

## 17. Typical Project Structure

```
controller/
service/
repository/
model/
templates/
```

---

## 18. One-Liner summary tbh:

> **Spring MVC** is a web framework that implements the **MVC pattern**, uses **DispatcherServlet** as the front controller, and supports both **server-side rendered views and RESTful APIs**.

---

