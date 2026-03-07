When working with **Spring Boot + Spring Data JPA**, basic CRUD operations such as:

```java
save()
findById()
deleteById()
findAll()
```

are automatically provided by `JpaRepository`.

However, real-world applications require **custom queries, filtering, joins, pagination, projections, and performance optimizations**. Spring Data JPA provides several ways to achieve this while still keeping code **clean, readable, and database-agnostic**.

Spring Data JPA supports four major query approaches:

1. **Built-in Repository Methods**
2. **Derived Queries** (method name parsing)
3. **JPQL Queries**
4. **Native SQL Queries**

Additionally, you must understand advanced concepts such as:

* Pagination
* Sorting
* Fetch joins
* DTO projections
* N+1 query problems
* Bulk updates



---

### Core Query Types

1. **Derived Queries**

   * Method name parsing
   * Keywords and operators
   * Automatic JPQL generation

2. **JPQL Queries**

   * Entity-based query language
   * Named parameters (`@Param`)
   * Complex joins
   * Fetch joins

3. **Pagination and Sorting**

   * `Page`
   * `Pageable`
   * `Sort`
   * Multi-column sorting

### Advanced Topics

4. **N+1 Query Problem**

   * Why it happens
   * How to fix it (`JOIN FETCH`, `@EntityGraph`, `@BatchSize`)

5. **@Modifying Queries**

   * Bulk UPDATE
   * Bulk DELETE
   * Persistence context behavior

6. **DTO Projections**

   * Constructor expressions
   * Interface projections

7. **Named Queries**

   * Reusable JPQL queries
   * Query compilation at startup

8. **Multi-table Joins**

   * Entity relationship joins
   * Nested joins

---

# Query Types Overview

| Query Type    | Syntax                                                  | Use Case            | Database Independent |
| ------------- | ------------------------------------------------------- | ------------------- | -------------------- |
| Built-in      | `save(), findById()`                                    | Simple CRUD         | Yes                  |
| Derived Query | `findByNameAndPhone()`                                  | Simple filtering    | Yes                  |
| JPQL          | `@Query("SELECT u FROM User u WHERE u.name=:name")`     | Complex queries     | Yes                  |
| Native SQL    | `@Query(value="SELECT * FROM users", nativeQuery=true)` | DB-specific queries | No                   |

### Key Concept

Derived queries and JPQL operate on **entities**, not database tables.

Example:

Entity:

```java
@Entity
class UserDetails {
    String name;
}
```

JPQL:

```
SELECT u FROM UserDetails u WHERE u.name = :name
```

SQL generated:

```
SELECT * FROM user_details WHERE name = ?
```

Because JPQL uses **entity objects**, your code becomes **database independent**.

This means:

* Switching **MySQL → PostgreSQL**
* Switching **Oracle → MariaDB**

will usually require **no query changes**.

---

# Derived Queries

Derived queries are one of the most powerful features of **Spring Data JPA**.

Spring automatically **parses method names** and converts them into **JPQL queries**.

Example:

```java
List<UserDetails> findByName(String name);
```

Spring automatically generates:

```
SELECT u FROM UserDetails u WHERE u.name = :name
```

This works because Spring internally parses the method name using:

```
PartTreeJpaQuery
QueryUtils
PartTree
```

It identifies:

* **Query type**
* **Fields**
* **Operators**

---

# Derived Query Naming Convention

General syntax:

```
find|read|get|query|search|stream|count + By + Field + Condition
```

Example:

```
findByName
countByAge
deleteByEmail
existsByUsername
```

Common prefixes:

| Prefix   | Meaning                 |
| -------- | ----------------------- |
| findBy   | fetch results           |
| readBy   | same as find            |
| getBy    | same as find            |
| queryBy  | same as find            |
| countBy  | count rows              |
| existsBy | boolean existence check |
| deleteBy | delete rows             |
| removeBy | delete rows             |

---

# Derived Query Examples

### Simple equality

```java
List<UserDetails> findByName(String name);
```

Generated JPQL:

```
SELECT u FROM UserDetails u WHERE u.name = :name
```

---

### Multiple conditions (AND)

```java
List<UserDetails> findByNameAndPhone(String name, String phone);
```

Generated query:

```
SELECT u FROM UserDetails u
WHERE u.name = :name AND u.phone = :phone
```

---

### OR condition

```java
List<UserDetails> findByNameOrPhone(String name, String phone);
```

---

### Combination

```java
List<UserDetails> findByNameAndPhoneOrId(String name,String phone,Long id);
```

Generated:

```
WHERE (name AND phone) OR id
```

Parentheses may vary depending on parsing.

---

# Derived Query Keywords

Spring provides many keywords.

### Comparison operators

| Keyword     | Meaning  |
| ----------- | -------- |
| GreaterThan | `>`      |
| LessThan    | `<`      |
| Between     | range    |
| IsNull      | null     |
| IsNotNull   | not null |

Example:

```java
List<UserDetails> findByIdGreaterThan(Long id);
```

SQL:

```
WHERE id > ?
```

---

### String matching

| Keyword      | SQL            |
| ------------ | -------------- |
| StartingWith | `LIKE 'abc%'`  |
| EndingWith   | `LIKE '%abc'`  |
| Containing   | `LIKE '%abc%'` |

Example:

```java
List<UserDetails> findByNameStartingWith(String prefix);
```

SQL:

```
WHERE name LIKE 'prefix%'
```

---

### IN clause

```java
List<UserDetails> findByNameIn(List<String> names);
```

SQL:

```
WHERE name IN (?, ?, ?)
```

---

### Null checks

```java
List<UserDetails> findByNameIsNull();
```

---

# Delete Queries

Spring can generate delete queries.

```java
void deleteByName(String name);
```

But internally this performs:

```
SELECT
DELETE
DELETE
DELETE
```

for each row.

So it is **not efficient for bulk deletes**.

Use `@Modifying` instead for large operations.

---

# Testing Derived Queries

Controller example:

```java
@GetMapping("/name/{name}")
public List<UserDetails> findByName(@PathVariable String name) {
    return repo.findByName(name);
}
```

Execution flow:

```
Controller
   ↓
Service
   ↓
Repository
   ↓
Spring Data JPA parses method
   ↓
JPQL generated
   ↓
SQL executed
```

---

# Pagination and Sorting

Large datasets must **never be returned entirely**.

Pagination solves this by splitting results into pages.

Spring uses:

```
Pageable
Page
Sort
```

---

# Pageable

A `Pageable` object defines:

```
page number
page size
sorting
```

Example:

```java
Pageable pageable =
    PageRequest.of(0,5, Sort.by("name").descending());
```

Meaning:

```
page = 0
size = 5
order by name desc
```

---

# Repository Pagination

```java
Page<UserDetails> findByNameStartingWith(String prefix, Pageable pageable);
```

Usage:

```java
Pageable pageable = PageRequest.of(0,5);

Page<UserDetails> page =
    repo.findByNameStartingWith("A", pageable);
```

---

# Page Object

`Page<T>` contains both **data and metadata**.

| Method             | Meaning         |
| ------------------ | --------------- |
| getContent()       | actual results  |
| getTotalElements() | total rows      |
| getTotalPages()    | total pages     |
| isFirst()          | first page      |
| isLast()           | last page       |
| hasNext()          | next exists     |
| hasPrevious()      | previous exists |

---

# Multi-field Sorting

Sort by multiple columns.

Example:

```java
Sort sort =
    Sort.by("name").descending()
        .and(Sort.by("phone").ascending());
```

SQL:

```
ORDER BY name DESC, phone ASC
```

---

# JPQL (Java Persistence Query Language)

JPQL is **object-oriented SQL**.

It works with:

* **Entities**
* **Entity fields**
* **Relationships**

instead of database tables.

---

### JPQL Query Example

```java
@Query("SELECT u FROM UserDetails u WHERE u.name = :name")
List<UserDetails> findByNameCustom(@Param("name") String name);
```

Explanation:

```
SELECT u
FROM UserDetails u
WHERE u.name = :name
```

`UserDetails` → entity
`u.name` → entity field

---

# Named Parameters

Instead of positional parameters (`?1`), we use named parameters.

Example:

```java
@Query("""
SELECT u
FROM UserDetails u
WHERE u.name = :name
AND u.phone = :phone
""")
UserDetails findByNameAndPhone(
    @Param("name") String name,
    @Param("phone") String phone
);
```

This improves:

* readability
* maintainability

---

# JPQL vs SQL

| JPQL          | SQL            |
| ------------- | -------------- |
| `UserDetails` | `user_details` |
| `u.name`      | `name`         |
| `:name`       | `?`            |

JPQL is converted into SQL by the **JPA provider (Hibernate)**.

---

# JPQL Return Types

JPQL supports multiple return types.

| Type             | Use Case         |
| ---------------- | ---------------- |
| `List<T>`        | multiple results |
| `T`              | single row       |
| `Page<T>`        | pagination       |
| `List<Object[]>` | multiple columns |
| `DTO`            | projections      |

---

# JPQL Joins

JPQL joins follow **entity relationships**.

You **do not manually write ON conditions**.

Instead, JPA uses entity mappings.

Example entity:

```
UserDetails
   |
   | OneToMany
   ↓
Orders
```

---

# Inner Join

```java
@Query("""
SELECT u
FROM UserDetails u
JOIN u.userAddress ua
WHERE ua.city = :city
""")
List<UserDetails> findUsersByCity(String city);
```

SQL generated:

```
SELECT *
FROM user_details u
INNER JOIN user_address ua
ON u.address_id = ua.id
WHERE ua.city = ?
```

---

# Fetch Join (Important)

Fetch joins solve **lazy loading problems**.

Example:

```java
@Query("""
SELECT u
FROM UserDetails u
JOIN FETCH u.orders
WHERE u.name = :name
""")
List<UserDetails> findUserWithOrders(String name);
```

Without fetch join:

```
1 query for users
N queries for orders
```

With fetch join:

```
1 query total
```

---

# Multi-Level Joins

Example:

```
Order → User → Address
```

Query:

```java
@Query("""
SELECT o
FROM OrderDetails o
JOIN o.user u
JOIN u.addresses a
WHERE a.city = :city
""")
List<OrderDetails> findOrdersByCity(String city);
```

This joins **three tables automatically**.

---

# DTO Projections

Fetching entire entities can be wasteful.

Example:

```
User entity
id
name
phone
email
password
createdDate
```

If we only need:

```
name
phone
```

DTO projections help.

---

### DTO Class

```java
public class UserDTO {

    private String name;
    private String phone;

    public UserDTO(String name, String phone) {
        this.name = name;
        this.phone = phone;
    }
}
```

---

### JPQL Projection

```java
@Query("""
SELECT new com.example.dto.UserDTO(u.name, u.phone)
FROM UserDetails u
""")
List<UserDTO> findUserSummaries();
```

This returns **only selected fields**, improving:

* memory usage
* query speed
* API performance

---

# @Modifying Queries (Bulk Updates)

JPQL normally only supports **SELECT**.

For UPDATE or DELETE you must add:

```
@Modifying
```

Example:

```java
@Modifying
@Query("""
UPDATE UserDetails u
SET u.phone = :phone
WHERE u.name = :name
""")
int updatePhone(String name,String phone);
```

Return value:

```
number of rows updated
```

---

# Important: Persistence Context

Bulk queries bypass Hibernate’s cache.

Therefore you should add:

```
@Modifying(clearAutomatically = true, flushAutomatically = true)
```

This ensures the persistence context remains consistent.

---

# N+1 Query Problem

One of the most common performance issues.

Example:

```
User
   |
   | OneToMany
   ↓
Orders
```

Code:

```
List<User> users = repo.findAll();

for(User u : users){
   u.getOrders();
}
```

Queries executed:

```
1 query for users
N queries for orders
```

Total:

```
N + 1 queries
```

If 100 users exist:

```
101 queries
```

Very slow.

---

# Fixing N+1

### 1️⃣ Fetch Join

```
JOIN FETCH
```

Most common solution.

---

### 2️⃣ EntityGraph

```java
@EntityGraph(attributePaths = "orders")
List<User> findAll();
```

---

### 3️⃣ BatchSize

```
@BatchSize(size=10)
```

Hibernate loads associations in batches.

---

# Named Queries

Named queries are defined **once** and reused.

Defined on the entity.

Example:

```java
@NamedQuery(
 name = "UserDetails.findByName",
 query = "SELECT u FROM UserDetails u WHERE u.name = :name"
)
```

Repository:

```java
List<UserDetails> findByName(String name);
```

Advantages:

* compiled at startup
* better performance
* reusable

---

# When to Use Each Query Type

| Situation                  | Best Option     |
| -------------------------- | --------------- |
| Simple filtering           | Derived queries |
| Complex filtering          | JPQL            |
| Database specific features | Native SQL      |
| Large data pagination      | Pageable        |
| Partial data               | DTO projection  |
| Performance optimization   | Fetch join      |

---

# Practical Rule Used by Most Teams

```
Derived Queries → 70%
JPQL → 25%
Native SQL → 5%
```

Derived queries are usually sufficient for **most CRUD operations**.

---

# Examples: 

**realistic production-style Spring Data JPA repository examples**.
These mimic what you typically see in **enterprise Spring Boot projects** (e-commerce, HR systems, fintech apps).
Contents: 
1. **User Repository**
2. **Order Repository**
3. **Product Repository**
4. **Pagination + Filtering**
5. **DTO Projection**
6. **Fetch Join (to avoid N+1 problem)**

---

# 1. UserRepository (Authentication + Search)

### Entity

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;
    private String username;
    private String password;
    private boolean active;
    private LocalDateTime createdAt;
}
```

---

### Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    Optional<User> findByUsername(String username);

    boolean existsByEmail(String email);

    List<User> findByActiveTrue();

    Page<User> findByActive(boolean active, Pageable pageable);

}
```

---

### Production Use Case

```java
Optional<User> user = userRepository.findByEmail(loginRequest.getEmail());

if(user.isEmpty()) {
    throw new RuntimeException("User not found");
}
```

---

# 2. ProductRepository (Search + Filters)

### Entity

```java
@Entity
public class Product {

    @Id
    private Long id;

    private String name;
    private String category;
    private double price;
    private boolean available;
}
```

---

### Repository

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    List<Product> findByCategory(String category);

    List<Product> findByPriceBetween(double min, double max);

    List<Product> findByAvailableTrue();

    Page<Product> findByCategoryAndAvailableTrue(String category, Pageable pageable);

}
```

---

### JPQL Example

```java
@Query("""
       SELECT p
       FROM Product p
       WHERE p.price > :price
       AND p.available = true
       """)
List<Product> findExpensiveProducts(double price);
```

---

# 3. OrderRepository (Joins + Fetch)

### Entity

```java
@Entity
public class Order {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne
    private User user;

    private double totalAmount;

    private LocalDateTime orderDate;
}
```

---

### Repository

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByUserId(Long userId);

    Page<Order> findByUserId(Long userId, Pageable pageable);

}
```

---

### Fetch Join Example (Avoid N+1 Problem)

```java
@Query("""
       SELECT o
       FROM Order o
       JOIN FETCH o.user
       WHERE o.id = :id
       """)
Optional<Order> findOrderWithUser(Long id);
```

This loads:

```
Order + User in one query
```

instead of multiple queries.

---

# 4. Pagination + Sorting (Real API Usage)

### Repository

```java
Page<Product> findByCategory(String category, Pageable pageable);
```

---

### Service Layer

```java
Pageable pageable = PageRequest.of(
        page,
        size,
        Sort.by("price").descending()
);

Page<Product> products =
        productRepository.findByCategory("Electronics", pageable);
```

---

### Controller API

```java
@GetMapping("/products")
public Page<Product> getProducts(
        @RequestParam int page,
        @RequestParam int size) {

    Pageable pageable = PageRequest.of(page, size);

    return productRepository.findAll(pageable);
}
```

---

# 5. DTO Projection (Very Common in Production)

Instead of returning entities, return **DTOs**.

---

### DTO

```java
public class ProductSummary {

    private String name;
    private double price;

    public ProductSummary(String name, double price) {
        this.name = name;
        this.price = price;
    }
}
```

---

### Repository

```java
@Query("""
       SELECT new com.example.dto.ProductSummary(p.name, p.price)
       FROM Product p
       WHERE p.available = true
       """)
List<ProductSummary> getAvailableProductSummaries();
```

Benefits:

* Less data transferred
* Faster queries
* No entity exposure

---

# 6. Complex Filter Query (Real Enterprise Query)

Example: **Admin searching orders**

```java
@Query("""
       SELECT o
       FROM Order o
       WHERE (:userId IS NULL OR o.user.id = :userId)
       AND (:minAmount IS NULL OR o.totalAmount >= :minAmount)
       AND (:maxAmount IS NULL OR o.totalAmount <= :maxAmount)
       """)
Page<Order> searchOrders(
        Long userId,
        Double minAmount,
        Double maxAmount,
        Pageable pageable);
```

---

# 7. Count Query

Used for **analytics dashboards**.

```java
@Query("SELECT COUNT(o) FROM Order o WHERE o.user.id = :userId")
long countUserOrders(Long userId);
```

---

# 8. Bulk Update Query

```java
@Modifying
@Query("""
       UPDATE Product p
       SET p.available = false
       WHERE p.price < :price
       """)
int disableCheapProducts(double price);
```

Needs:

```java
@Transactional
```

---

# 9. Delete Query

```java
void deleteByCategory(String category);
```

Spring automatically generates:

```
DELETE FROM product WHERE category = ?
```

---

# Production Best Practices
 Use **DTO projections** instead of entities in APIs
 
 Use **Pageable for large datasets**
 
 Use **FETCH JOIN to avoid N+1 queries**
 
 Use **Optional for single results**
 
Use **@Modifying + @Transactional for updates**

---

 **Real enterprise repositories often reach 50–100 methods**, combining:

* Derived queries
* JPQL
* DTO projections
* Pagination
* Specifications

---




