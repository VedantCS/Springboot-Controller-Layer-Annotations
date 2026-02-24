Spring Boot JPA entity-table mapping uses annotations to precisely control how Java classes map to database tables, columns, constraints, and primary keys.

## **Contents of this file:**

1. Configure `spring.jpa.hibernate.ddl-auto` for different environments
2. Use `@Table` for custom table names, schemas, and constraints
3. Implement composite primary keys using `@IdClass` and `@EmbeddedId`
4. Generate primary key values with `@GeneratedValue` (IDENTITY vs SEQUENCE)
5. Apply column-level constraints with `@Column`
6. Understand database schema vs table relationships
7. Choose optimal ID generation strategies for production

## **Schema Management with ddl-auto**

The `spring.jpa.hibernate.ddl-auto` property controls how Hibernate manages your database schema on startup. [stackoverflow](https://stackoverflow.com/questions/42135114/how-does-spring-jpa-hibernate-ddl-auto-property-exactly-work-in-spring)

### Configuration Options

| Value | Startup Behavior | Shutdown Behavior | Best For | Production? |
|-------|------------------|-------------------|----------|-------------|
| `none` | No action | No action | **Production**  [stackoverflow](https://stackoverflow.com/questions/42135114/how-does-spring-jpa-hibernate-ddl-auto-property-exactly-work-in-spring) | **Recommended** |
| `update` | Creates/updates tables | No action | **Development** | no |
| `validate` | Validates schema match | No action | QA/Staging |yes |
| `create` | Drops + recreates schema | No action | Testing | no |
| `create-drop` | Creates schema | **Drops schema** | Unit tests  [stackoverflow](https://stackoverflow.com/questions/42135114/how-does-spring-jpa-hibernate-ddl-auto-property-exactly-work-in-spring) | no |

**Production Rule**: Always use `none`. Use database migration tools (Flyway, Liquibase) instead. [stackoverflow](https://stackoverflow.com/questions/42135114/how-does-spring-jpa-hibernate-ddl-auto-property-exactly-work-in-spring)

**Development Setup**:
```properties
# application.properties (Development)
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.h2.console.enabled=true
```

**Production Setup**:
```properties
# application-prod.properties
spring.jpa.hibernate.ddl-auto=none
```

**Testing Setup**:
```properties
# application-test.properties
spring.jpa.hibernate.ddl-auto=create-drop
```

### Schema vs Database Clarification

**Database**: Container for schemas/tables 
**Schema**: Logical grouping of tables within a database
```
Database (userdb)
├── Schema: onboarding
│   ├── Table: user_details
│   └── Table: order_details
├── Schema: billing
│   └── Table: invoices
└── Tables without schema
    └── legacy_table
```

Schemas enable **team isolation** in shared databases.
***

## **Concept Map**

```
@Entity → @Table(name, schema, constraints, indexes)
    ↓
Fields → @Column(name, unique, nullable, length)
    ↓
Primary Key → @Id + @GeneratedValue(strategy)
    ↓ Composite Keys
        ├── @IdClass + @Id fields
        └── @EmbeddedId + @Embeddable

hibernate.ddl-auto
┌─────────────┬─────────────┬─────────────┐
│ Production  │ Development │ Testing     │
│ none        │ update      │ create-drop │
└─────────────┴─────────────┴─────────────┘
```

***

## **@Table Annotation**

Customizes table generation. 
```java
@Entity
@Table(
    name = "user_details",           // Custom table name
    schema = "onboarding",           // Schema name (create first!)
    uniqueConstraints = {            // Table-level unique constraints
        @UniqueConstraint(
            columnNames = {"phone"}
        ),
        @UniqueConstraint(
            columnNames = {"name", "email"}  // Composite unique
        )
    },
    indexes = {                      // Table indexes
        @Index(name = "phone_idx", columnList = "phone"),
        @Index(name = "name_email_idx", columnList = "name,email")
    }
)
public class UserDetails {
    // fields...
}
```

**Generated Schema**:
```sql
CREATE TABLE onboarding.user_details (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(20),
    UNIQUE KEY uk_phone (phone),
    UNIQUE KEY uk_name_email (name, email),
    INDEX phone_idx (phone),
    INDEX name_email_idx (name, email)
);
```

**Schema Creation** (H2 example):
```properties
spring.datasource.url=jdbc:h2:mem:userdb;INIT=CREATE SCHEMA IF NOT EXISTS onboarding
```

***

## **@Column Annotation**

Column-level customization (optional, defaults applied if absent).
```java
@Column(
    name = "full_name",      // Custom column name
    unique = true,           // Single-column unique constraint
    nullable = false,        // NOT NULL
    length = 100,            // VARCHAR(100)
    columnDefinition = "TEXT" // Custom type (DB-specific)
)
private String name;

@Column(name = "phone_number", unique = true)
private String phone;
```

**Default Behavior** (if no `@Column`):
- `name` → `NAME` (uppercase, underscores for camelCase)
- `nullable = true`, `unique = false`
- `length = 255` for String
***

## **Primary Key Generation**

### Single Primary Key

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-increment
private Long id;
```

### IDENTITY vs SEQUENCE

| Strategy | How It Works | Pros | Cons | Best For |
|----------|--------------|------|------|----------|
| **IDENTITY** | DB auto-increment column  [stackoverflow](https://stackoverflow.com/questions/8955074/generatedvaluestrategy-identity-vs-generatedvaluestrategy-sequence) | Simple, DB-native | **Table-specific**, slow batch inserts | Small apps, MySQL  [stackoverflow](https://stackoverflow.com/questions/8955074/generatedvaluestrategy-identity-vs-generatedvaluestrategy-sequence) |
| **SEQUENCE** | DB sequence object  [stackoverflow](https://stackoverflow.com/questions/8955074/generatedvaluestrategy-identity-vs-generatedvaluestrategy-sequence) | **DB-independent**, cacheable, fast batch  [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/82314936/0f4e052b-5e02-450e-bccb-8c2b0a689bce/paste.txt) | More complex setup | **Production**  [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/82314936/0f4e052b-5e02-450e-bccb-8c2b0a689bce/paste.txt) |

**SEQUENCE Example**:
```java
@Id
@GeneratedValue(
    strategy = GenerationType.SEQUENCE,
    generator = "user_seq"
)
@SequenceGenerator(
    name = "user_seq",
    sequenceName = "user_sku",     // DB sequence name
    initialValue = 200,
    allocationSize = 3             // Cache 3 values [file:11]
)
private Long id;
```

**Generated SQL**:
```sql
-- Startup: Creates sequence
CREATE SEQUENCE user_sku START WITH 200 INCREMENT BY 3;

-- First 3 inserts (cached):
INSERT → id=200
INSERT → id=201  
INSERT → id=202

-- 4th insert (fetches next batch):
SELECT nextval('user_sku') → 203,204,205 (cached)
```

**Benefits of SEQUENCE**: 
1. **Batch efficient**: Cache multiple IDs [stackoverflow](https://stackoverflow.com/questions/8955074/generatedvaluestrategy-identity-vs-generatedvaluestrategy-sequence)
2. **DB portable**: Works across databases
3. **Centralized**: Multiple tables share sequences
4. **Custom logic**: Microservice for ID generation

**Avoid TABLE Strategy**: Creates dedicated table for IDs (slow, locking issues).
***

## **Composite Primary Keys**

Two approaches for multiple-column PKs. [baeldung](https://www.baeldung.com/jpa-composite-primary-keys)

### 1. @IdClass (Separate Class)

**Composite Key Class** (must be `public`, `Serializable`, no-arg constructor, `equals()`/`hashCode()`): [baeldung](https://www.baeldung.com/jpa-composite-primary-keys)
```java
import java.io.Serializable;

@IdClass(UserDetailsCK.class)
@Entity
@Table(name = "user_details")
public class UserDetails {
    @Id private String name;
    @Id private String address;
    private String phone;
    
    // Getters/Setters
}

public class UserDetailsCK implements Serializable {
    private String name;
    private String address;
    
    public UserDetailsCK() {}  // No-arg constructor
    
    // equals() and hashCode() - REQUIRED!
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof UserDetailsCK)) return false;
        UserDetailsCK other = (UserDetailsCK) obj;
        return Objects.equals(name, other.name) && 
               Objects.equals(address, other.address);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name, address);
    }
}
```

### 2. @EmbeddedId (Embedded Object) [logicbig](https://www.logicbig.com/tutorials/java-ee-tutorial/jpa/embedded-id.html)

**Cleaner approach** - treats composite key as single object: [codingtechroom](https://codingtechroom.com/question/choosing-between-idclass-and-embeddedid-in-jpa)
```java
@EmbeddedId
@Entity
@Table(name = "user_details")
public class UserDetails {
    private UserDetailsPK id;  // Single field for composite key
    private String phone;
}

@Embeddable
public class UserDetailsPK implements Serializable {
    private String name;
    private String address;
    
    public UserDetailsPK() {}
    
    // equals() and hashCode()
}
```

**@IdClass vs @EmbeddedId**:

| Feature | @IdClass | @EmbeddedId |
|---------|----------|-------------|
| Fields | **Duplicated** in entity + key class  [baeldung](https://www.baeldung.com/jpa-composite-primary-keys) | **Single reference** |
| Usage | `repository.findById(PKObject)` | `repository.findById(PKObject)` |
| Clarity | Explicit field access | Object composition  [logicbig](https://www.logicbig.com/tutorials/java-ee-tutorial/jpa/embedded-id.html) |
| **Recommended** | Legacy | **Modern apps**  [codingtechroom](https://codingtechroom.com/question/choosing-between-idclass-and-embeddedid-in-jpa) |

**Why Serializable/equals/hashCode?**
1. **HashMap storage** in first/second-level caches
2. **Serialization** for distributed caching/clustering [baeldung](https://www.baeldung.com/jpa-composite-primary-keys)

***

## **Complete Working Example**

```java
@Entity
@Table(
    name = "users",
    schema = "app",
    uniqueConstraints = {
        @UniqueConstraint(columnNames = "email")
    }
)
public class User {
    @Id
    @GeneratedValue(
        strategy = GenerationType.SEQUENCE,
        generator = "user_seq"
    )
    @SequenceGenerator(
        name = "user_seq",
        sequenceName = "users_sku",
        initialValue = 1000,
        allocationSize = 10
    )
    private Long id;
    
    @Column(name = "full_name", nullable = false, length = 100)
    private String name;
    
    @Column(unique = true)
    private String email;
}
```

**application.properties**:
```properties
spring.jpa.hibernate.ddl-auto=update
spring.datasource.url=jdbc:h2:mem:testdb;INIT=CREATE SCHEMA IF NOT EXISTS app
```

**Repository**:
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
}
```

***

## **Summary**

JPA mapping annotations provide precise control over database schema generation:
1. **`ddl-auto=update`** for development, **`none`** for production [stackoverflow](https://stackoverflow.com/questions/42135114/how-does-spring-jpa-hibernate-ddl-auto-property-exactly-work-in-spring)
2. **`@Table`** customizes table name/schema/constraints/indexes
3. **`@Column`** controls column properties (name, unique, nullable, length)
4. **`@Id + @GeneratedValue`** for single PK with IDENTITY/SEQUENCE [stackoverflow](https://stackoverflow.com/questions/8955074/generatedvaluestrategy-identity-vs-generatedvaluestrategy-sequence)
5. **Composite keys**: `@IdClass` (explicit) or `@EmbeddedId` (recommended) [logicbig](https://www.logicbig.com/tutorials/java-ee-tutorial/jpa/embedded-id.html)
6. **SEQUENCE** preferred for production (batch-efficient, portable) 
**Production Checklist**:
- `ddl-auto=none`
- Use migration tools (Flyway/Liquibase)
- SEQUENCE generation with allocationSize
- `@EmbeddedId` for composite keys
- Proper indexes/unique constraints

***

## **Application: Production-Ready Entity**

Create an `Order` entity with:
1. Composite key (`orderNumber`, `customerId`) using `@EmbeddedId`
2. `SEQUENCE` ID for `orderId`
3. Proper indexes on `customerId`, `status`
4. Unique constraint on `orderNumber` per customer
5. Test with `ddl-auto=create-drop` then switch to `none`

***

## **Revision**

1. **Configuration**: What's the production value for `spring.jpa.hibernate.ddl-auto`? Why?

2. **Mapping**: Entity `UserDetails` without `@Table` becomes what table name in H2?

3. **Composite Keys**: Compare `@IdClass` vs `@EmbeddedId`. Which requires field duplication?

4. **Generation**: 5 inserts with SEQUENCE `allocationSize=3`. How many sequence queries?

5. **Constraints**: How do you create unique constraint on `email` + `phone` columns?

6. **Schema**: `spring.datasource.url` with `INIT=CREATE SCHEMA onboarding`. Where does `@Table(schema="onboarding")` table go?

7. **Performance**: Why SEQUENCE better than IDENTITY for 1000 inserts/second?

8. **Practical**: Entity has `String name`. Without `@Column`, what's column type/length?
