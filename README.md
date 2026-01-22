# Springboot-Controlle-Layer-Annotations

# Spring Boot Annotations - Comprehensive Notes

## 1. Controller Annotations

### @Controller vs @RestController

**@Controller**
- Indicates that a class is responsible for handling incoming HTTP requests
- Spring Boot includes this class in the list of controller classes eligible to handle HTTP requests
- Requires `@ResponseBody` annotation on each method to return HTTP response directly
- Without `@ResponseBody`, Spring Boot treats return values as view names (e.g., "hello.jsp")
- Use case: When you need to render views

**@RestController**
- Equivalent to `@Controller` + `@ResponseBody`
- No need to add `@ResponseBody` on individual methods
- All method return types are automatically considered as HTTP response bodies
- More convenient for REST APIs
- **Preferred** for modern REST API development

**Key Difference:**
```java
// With @Controller - need @ResponseBody on each method
@Controller
public class SampleController {
    @ResponseBody
    public String getUser() { return "hello"; }
}

// With @RestController - no @ResponseBody needed
@RestController
public class SampleController {
    public String getUser() { return "hello"; }
}
```

---

## 2. Request Mapping Annotations

### @RequestMapping
- **Purpose:** Maps API endpoints to controller methods
- Maps requests to specific methods based on path and HTTP method
- Can be used at class level (for common base paths) and method level

**Parameters:**
- `path` or `value`: Defines the URL path (interchangeable - use either)
- `method`: Specifies HTTP method (GET, POST, PUT, DELETE, PATCH)

**Class-Level Mapping:**
```java
@RestController
@RequestMapping("/api")  // Common base path
public class SampleController {
    @RequestMapping(path = "/fetch-user", method = RequestMethod.GET)
    public String getUser() { }

    @RequestMapping(path = "/save-user", method = RequestMethod.POST)
    public String saveUser() { }
}
// Results in: /api/fetch-user and /api/save-user
```

### Specialized Mapping Annotations (Preferred)

**@GetMapping, @PostMapping, @PutMapping, @DeleteMapping**
- Simplified alternatives to `@RequestMapping`
- HTTP method is predefined by default
- Better readability
- Only need to specify the path

**Example:**
```java
@GetMapping("/fetch-user")  // Cleaner than @RequestMapping
public String getUser() { }

@PostMapping("/save-user")
public String saveUser() { }
```

**Internal Implementation:**
- Uses `@Mapping` and `@ReflectiveController` annotations
- Mapping logic resolved through reflection
- `ReflectiveControllerMappingProcessor` performs the actual controller-to-API mapping

---

## 3. @RequestParam

### Purpose
Binds URL query parameters to method parameters

### Syntax
```java
@GetMapping("/fetch-user")
public String getUser(
    @RequestParam(name = "firstName") String firstName,
    @RequestParam(name = "lastName", required = false) String lastName
) { }
```

**URL Example:**
```
/api/fetch-user?firstName=Shreyansh&lastName=Doe&age=25
```

### Key Properties
- `name` or `value`: Specifies the parameter name in the URL
- `required`: Boolean flag (default = `true`)
  - If `true`: Parameter must be present, otherwise API throws error
  - If `false`: Parameter is optional, value will be `null` if not provided

### Automatic Type Conversion
Spring Framework automatically converts string parameters to:
- Primitive types (int, long, boolean, etc.)
- Wrapper classes (Integer, Long, Boolean, etc.)
- Strings
- Enums
- Custom object types (via custom converters)

**Example:**
```
?age=20  // String "20" automatically converted to int
```

---

## 4. Custom Type Conversion with @InitBinder

### Purpose
Perform custom preprocessing on request parameters before they're assigned to method parameters

### Implementation
```java
@RestController
@RequestMapping("/api")
public class SampleController {

    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(
            String.class,  // Target type
            "firstName",   // Parameter name
            new MyFirstNamePropertyEditor()  // Custom editor
        );
    }

    @GetMapping("/fetch-user")
    public String getUser(@RequestParam("firstName") String firstName) { }
}
```

### Custom Property Editor
```java
public class MyFirstNamePropertyEditor extends PropertyEditorSupport {
    @Override
    public void setAsText(String text) {
        // Custom logic: trim and convert to lowercase
        setValue(text.trim().toLowerCase());
    }
}
```

**How It Works:**
1. `@InitBinder` runs before each method invocation
2. Checks if any parameters need preprocessing
3. Applies custom editor if type and name match
4. Processes the value before assigning to method parameter

---

## 5. @PathVariable

### Purpose
Extracts values from the URL path itself (not query parameters)

### Syntax
```java
@GetMapping("/fetch-user/{firstName}")
public String getUser(@PathVariable("firstName") String firstName) { }
```

**URL Example:**
```
/api/fetch-user/Shreyansh  // "Shreyansh" is extracted as firstName
```

### Difference from @RequestParam
- **@RequestParam:** `/api/user?id=123` (query parameter)
- **@PathVariable:** `/api/user/123` (path segment)

### Use Cases
- RESTful resource identifiers
- Dynamic URL segments
- Cleaner, more semantic URLs

---

## 6. @RequestBody

### Purpose
Binds HTTP request body (typically JSON or XML) to a Java object

### Implementation
```java
@PostMapping("/save-user")
public String saveUser(@RequestBody User user) {
    return "User saved: " + user.getUsername();
}
```

**JSON Request Body:**
```json
{
    "user_name": "Shreyansh",
    "email": "test@example.com"
}
```

### Java Class Mapping
```java
public class User {
    @JsonProperty("user_name")  // Maps JSON key to Java field
    private String username;

    private String email;  // Direct mapping (same name)

    // Getters and Setters required
}
```

### Key Points
- Spring uses **Jackson** or **JSON** libraries for conversion
- Field names should match JSON keys (case-sensitive)
- Use `@JsonProperty` when names differ
- Requires getters/setters for serialization/deserialization

---

## 7. ResponseEntity

### Purpose
Represents the complete HTTP response (status, headers, body)

### Components
1. **Status Code:** HTTP status (200, 400, 404, 500, etc.)
2. **Headers:** Custom headers
3. **Body:** Response content

### Implementation
```java
@GetMapping("/fetch-user")
public ResponseEntity<String> getUser() {
    return ResponseEntity
        .status(HttpStatus.OK)  // 200
        .header("Custom-Header", "value")
        .body("User details fetched successfully");
}
```

### @RestController vs ResponseEntity

**@RestController behavior:**
- Automatically wraps return values in HTTP response
- Implicitly creates `ResponseEntity` with default status (200)
- Only controls the body

**ResponseEntity advantages:**
- **Explicit control** over status codes
- Ability to add custom headers
- Better for error handling and different response scenarios

**Example Comparison:**
```java
// Simple @RestController return
@GetMapping("/user")
public String getUser() {
    return "User data";  // Status 200 by default
}

// With ResponseEntity
@GetMapping("/user")
public ResponseEntity<String> getUser() {
    return ResponseEntity
        .status(HttpStatus.OK)
        .body("User data");  // Explicit status control
}
```

### @Controller Behavior (Without @ResponseBody)
- Does NOT create `ResponseEntity`
- Treats return value as view name
- Attempts to find and render a view file (e.g., JSP)
- Results in errors if view not found

---

## Key Takeaways

1. **Use @RestController** for REST APIs (most common)
2. **Specialized mappings** (@GetMapping, @PostMapping) are preferred over @RequestMapping
3. **@RequestParam** for query parameters, **@PathVariable** for path segments
4. **@InitBinder** enables custom preprocessing logic
5. **@RequestBody** for JSON/XML deserialization
6. **ResponseEntity** provides complete HTTP response control
7. All mapping logic uses **reflection** internally
8. Type conversion is **automatic** for standard types

---

## Notes on Internal Implementation
- Handler mapping uses reflection to identify controllers
- `@ReflectiveController` and `@Mapping` annotations handle mapping logic
- `ReflectiveControllerMappingProcessor` performs controller-to-API binding
- Detailed annotation mechanics covered in Java Annotations video (prerequisite)

---

## Practical Tips
- Always use `@RestController` for REST APIs
- Put common base paths at class level with `@RequestMapping`
- Use `required = false` for optional parameters
- Leverage `@JsonProperty` for JSON field name mismatches
- Use `ResponseEntity` when you need explicit status code control
- Custom type conversion via `@InitBinder` for complex preprocessing needs
