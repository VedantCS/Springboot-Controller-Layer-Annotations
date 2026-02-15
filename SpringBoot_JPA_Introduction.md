#  Objectives

1. Understand the complete data persistence architecture in Spring Boot applications
2. Differentiate between JDBC, JPA, and their implementations (Hibernate)
3. Recognize the problems plain JDBC introduces and how Spring Boot solves them
4. Implement database operations using Spring Boot JDBC Template
5. Configure and use connection pooling (HikariCP) effectively
6. Apply different JDBC Template methods for CRUD operations

***

## **What is the Database Access Stack?**

When building Spring Boot applications that interact with databases, multiple layers work together. Here's the complete flow: [springboottutorial](https://www.springboottutorial.com/introduction-to-jpa-with-spring-boot-data-jpa)

<img width="1086" height="268" alt="image" src="https://github.com/user-attachments/assets/f4358105-237b-45cf-a62b-88058b032d0c" />

**Application Logic** → **JPA (Interface)** → **Hibernate (Implementation)** → **JDBC API (Interface)** → **Database Drivers (Implementation)** → **Database**

### Understanding Each Layer

- **JPA (Java Persistence API)**: A specification that provides interfaces for object-relational mapping, not actual implementation [dev](https://dev.to/yigi/hibernate-vs-jdbc-vs-jpa-vs-spring-data-jpa-1421)
- **Hibernate**: The most popular implementation of JPA that provides the actual code [springboottutorial](https://www.springboottutorial.com/introduction-to-jpa-with-spring-boot-data-jpa)
- **JDBC API**: Java Database Connectivity interface for database operations
- **Database Drivers**: Vendor-specific implementations (MySQL Connector, PostgreSQL Driver, H2 Driver) that know how to communicate with specific databases [dev](https://dev.to/yigi/hibernate-vs-jdbc-vs-jpa-vs-spring-data-jpa-1421)
- **ORM**: ORM acts as bridge between Java Objects and Database tables, we can interact with databases using Java Objects.
This architecture means you write code against stable interfaces (JPA, JDBC), while implementations can change without affecting your code. [stackoverflow](https://stackoverflow.com/questions/42470060/spring-data-jdbc-spring-data-jpa-vs-hibernate)

## How Does JPA Work?

JPA was designed with a different approach: instead of focusing on queries, it maps Java objects directly to database tables.

Key concepts include:

    Entities – Java classes that represent database tables
    Attributes – Fields in the class that correspond to table columns
    Relationships – Associations between entities, such as one-to-many or many-to-many

This process is known as Object-Relational Mapping (ORM).
Before JPA, the term ORM was commonly used to describe frameworks like Hibernate, which is why Hibernate is often referred to as an ORM framework.
***

## **Concept Map**

```
                    Spring Boot Application
                            |
        ┌───────────────────┴───────────────────┐
        |                                       |
    Plain JDBC                          Spring Boot JDBC
        |                                       |
   [Problems]                            [Solutions]
        |                                       |
├─ Manual Connection          ├─ Auto Connection (DataSource)
├─ Driver Loading             ├─ Auto Driver Loading
├─ Boilerplate Code          ├─ JdbcTemplate (abstraction)
├─ Generic SQL Exception     ├─ Granular Exceptions
├─ Manual Resource Closing   ├─ Auto Resource Management
└─ Manual Pool Management    └─ HikariCP (default pool)
                                       |
                                [Methods]
                                       |
                    ┌──────────────────┼──────────────────┐
                    |                  |                  |
              update()           query()          queryForObject()
             (Insert/             (Multiple            (Single
          Update/Delete)            Rows)               Row/Value)
```

***

## **Plain JDBC: The Problems**

### 1. Manual Driver Loading
Every application must explicitly load database drivers using `Class.forName()`: [dev](https://dev.to/yigi/hibernate-vs-jdbc-vs-jpa-vs-spring-data-jpa-1421)

```java
Class.forName("org.h2.Driver"); // Must be done manually
```

### 2. Manual Connection Management
Developers must create connections manually for every operation:

```java
Connection conn = DriverManager.getConnection(url, username, password);
```

### 3. Boilerplate Code Everywhere
Each database operation requires:
- Creating connection
- Creating PreparedStatement
- Setting parameters
- Executing query
- Processing ResultSet
- Handling exceptions
- Closing resources in finally block

This results in repetitive code across every DAO method. [dev](https://dev.to/yigi/hibernate-vs-jdbc-vs-jpa-vs-spring-data-jpa-1421)

### 4. Generic Exception Handling
JDBC throws generic `SQLException` for all errors - whether it's a duplicate key, timeout, or connection failure. This makes debugging and specific error handling difficult. [stackoverflow](https://stackoverflow.com/questions/42470060/spring-data-jdbc-spring-data-jpa-vs-hibernate)

### 5. Manual Resource Management
Developers must explicitly close connections, statements, and result sets in finally blocks to prevent memory leaks.

### 6. No Built-in Connection Pooling
Plain JDBC creates a new connection for every operation, which is expensive. Developers must manually implement connection pooling. [geeksforgeeks](https://www.geeksforgeeks.org/springboot/optimizing-performance-with-spring-data-jpa/)

***

## **Spring Boot JDBC: The Solution**

### Setup Requirements

Add these dependencies to `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Configuration in application.properties

```properties
spring.datasource.url=jdbc:h2:mem:userdb
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=
```

Spring Boot reads these properties and automatically creates a `DataSource` bean that handles connection creation and pooling. [spring](https://spring.io/guides/gs/accessing-data-jpa)

***

## **JdbcTemplate: Your Best Friend**

`JdbcTemplate` is Spring's central class that eliminates JDBC boilerplate code. It automatically handles: [spring](https://spring.io/guides/gs/accessing-data-jpa)

- Connection acquisition and release
- Exception translation (SQL exceptions → Spring's DataAccessException hierarchy)
- Resource cleanup
- Integration with Spring's transaction management

### Repository Pattern with JdbcTemplate

```java
@Repository
public class UserRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public void createTable() {
        jdbcTemplate.execute(
            "CREATE TABLE users (user_id INT AUTO_INCREMENT, username VARCHAR(255), age INT)"
        );
    }
}
```

The `@Repository` annotation serves two purposes:
1. Marks the class as a Spring bean for dependency injection [springboottutorial](https://www.springboottutorial.com/introduction-to-jpa-with-spring-boot-data-jpa)
2. Enables automatic exception translation to Spring's granular exception hierarchy [linkedin](https://www.linkedin.com/pulse/best-practice-using-spring-data-jpa-chamseddine-toujani-wgcae)

***

## **JdbcTemplate Methods Reference**

### 1. update() - For Insert/Update/Delete

**Syntax**: `update(String sql, Object... args)`

```java
// Insert
jdbcTemplate.update(
    "INSERT INTO users (username, age) VALUES (?, ?)",
    "John", 25
);

// Update
jdbcTemplate.update(
    "UPDATE users SET age = ? WHERE username = ?",
    26, "John"
);

// Delete
jdbcTemplate.update(
    "DELETE FROM users WHERE user_id = ?",
    1
);
```

### 2. update() with PreparedStatementSetter - For Complex Queries

For more control over parameter setting:

```java
jdbcTemplate.update(
    "INSERT INTO users (username, age) VALUES (?, ?)",
    (PreparedStatement ps) -> {
        ps.setString(1, user.getUsername());
        ps.setInt(2, user.getAge());
    }
);
```

### 3. query() - For Multiple Rows

**Syntax**: `query(String sql, RowMapper<T> rowMapper)`

```java
List<User> users = jdbcTemplate.query(
    "SELECT * FROM users",
    (ResultSet rs, int rowNum) -> {
        User user = new User();
        user.setUserId(rs.getInt("user_id"));
        user.setUsername(rs.getString("username"));
        user.setAge(rs.getInt("age"));
        return user;
    }
);
```

The `RowMapper` is a functional interface that converts each row of the ResultSet into a Java object. [springboottutorial](https://www.springboottutorial.com/introduction-to-jpa-with-spring-boot-data-jpa)

### 4. queryForList() - For Single Column, Multiple Rows

```java
List<String> usernames = jdbcTemplate.queryForList(
    "SELECT username FROM users",
    String.class
);
```

### 5. queryForObject() - For Single Row

**Syntax**: `queryForObject(String sql, Object[] args, Class<T> requiredType)`

```java
User user = jdbcTemplate.queryForObject(
    "SELECT * FROM users WHERE user_id = ?",
    new Object[]{1},
    (ResultSet rs, int rowNum) -> {
        User u = new User();
        u.setUserId(rs.getInt("user_id"));
        u.setUsername(rs.getString("username"));
        u.setAge(rs.getInt("age"));
        return u;
    }
);
```

### 6. queryForObject() - For Single Value

```java
Integer count = jdbcTemplate.queryForObject(
    "SELECT COUNT(*) FROM users",
    Integer.class
);
```

***

## **HikariCP: Default Connection Pooling**

Spring Boot automatically configures HikariCP as the default connection pool. HikariCP is known for being fast and lightweight. [linkedin](https://www.linkedin.com/pulse/best-practice-using-spring-data-jpa-chamseddine-toujani-wgcae)

### Default Configuration

- **Maximum pool size**: 10 connections
- **Minimum idle connections**: 10

### Custom Configuration

```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
```

### Why Connection Pooling Matters

Creating database connections is expensive. Connection pools maintain a pool of pre-created connections that can be reused, significantly improving performance. [geeksforgeeks](https://www.geeksforgeeks.org/springboot/optimizing-performance-with-spring-data-jpa/)

***

## **Exception Handling: From Generic to Granular**

Plain JDBC throws generic `SQLException` for all errors. Spring translates these into specific exceptions: [stackoverflow](https://stackoverflow.com/questions/42470060/spring-data-jdbc-spring-data-jpa-vs-hibernate)

- `DuplicateKeyException` - When inserting duplicate primary keys
- `DataIntegrityViolationException` - Constraint violations
- `QueryTimeoutException` - Query execution timeout
- `CannotAcquireLockException` - Database locking issues
- `EmptyResultDataAccessException` - Expected result not found

This granular exception hierarchy allows you to handle specific error scenarios in your code. [linkedin](https://www.linkedin.com/pulse/best-practice-using-spring-data-jpa-chamseddine-toujani-wgcae)

***

## **Summary**

Spring Boot JDBC eliminates the pain points of plain JDBC through automated configuration and the `JdbcTemplate` abstraction. Key improvements include: [spring](https://spring.io/guides/gs/accessing-data-jpa)

1. **Automatic driver loading** during application startup
2. **DataSource bean** that manages connections automatically
3. **JdbcTemplate** that eliminates boilerplate code
4. **HikariCP connection pooling** configured by default
5. **Granular exception hierarchy** for better error handling
6. **Automatic resource management** - no manual closing required

While JDBC Template simplifies database access, it still requires you to write SQL queries and manually map results to objects. This is where JPA and Hibernate (covered in Part 2) provide even higher-level abstractions through Object-Relational Mapping (ORM). [springboottutorial](https://www.springboottutorial.com/introduction-to-jpa-with-spring-boot-data-jpa)

***

## **Application: Practical Implementation Steps**

### Step 1: Add Dependencies
Include `spring-boot-starter-jdbc` and your database driver in `pom.xml`.

### Step 2: Configure Database
Set connection properties in `application.properties`.

### Step 3: Create Entity Class
Define a simple POJO representing your table.

### Step 4: Create Repository
Use `@Repository` and inject `JdbcTemplate` to implement data access methods.

### Step 5: Create Service Layer
Implement business logic that uses the repository. [springboottutorial](https://www.springboottutorial.com/introduction-to-jpa-with-spring-boot-data-jpa)

### Step 6: Test Your Implementation
Create a controller or test class to verify CRUD operations work correctly.

***

## **Self-Assessment/ Revision Questions**

1. **Conceptual**: What are the four main problems that Spring Boot JDBC solves compared to plain JDBC?

2. **Technical**: Explain the difference between `query()` and `queryForObject()` methods. When would you use each?

3. **Architecture**: Draw the complete flow from your Spring Boot application to the database, labeling each layer.

4. **Practical**: If you need to insert 1000 records, which `JdbcTemplate` method would you use and why?

5. **Exception Handling**: What specific exception would Spring throw for a duplicate primary key violation, and how is this better than plain JDBC's `SQLException`?

6. **Configuration**: How would you change from H2 to MySQL database? What files need modification?

7. **Performance**: Explain how HikariCP improves application performance. What happens when all 10 connections are in use?

8. **Application**: Write a complete repository method to update a user's age given their user ID, including proper exception handling.

***
