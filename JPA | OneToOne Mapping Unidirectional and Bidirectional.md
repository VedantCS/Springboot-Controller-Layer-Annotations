
## **Topicss**


1. Unidirectional `@OneToOne` (parent → child)
2. Bidirectional `@OneToOne` (parent ↔ child via `mappedBy`)
3. Cascade types (`PERSIST`, `MERGE`, `REMOVE`, etc.)
4. Fetch strategies (`EAGER` vs `LAZY`)
5. **Lazy loading handling** for `@OneToOne`
6. **Cascade best practices** for bidirectional
7. **Infinite recursion fixes** for JSON
8. Composite key handling with `@JoinColumns`
9. DTO pattern for safe API responses

Spring Boot JPA `@OneToOne` mapping establishes a 1:1 relationship between entities, with unidirectional (one-way) or bidirectional (two-way) navigation, using foreign keys and annotations like `@JoinColumn` and `mappedBy`.

## **OneToOne Fundamentals**

**Core Theory**: One entity (A) references **exactly one** instance of another entity (B), creating a 1:1 relationship. The **foreign key (FK)** lives in **one table only** (owner side).

**Unidirectional**: Reference exists only **one direction** (parent → child). Child unaware of parent. From `UserDetails`, you can access `UserAddress`, but `UserAddress` cannot access back to `UserDetails`.

**Bidirectional**: Both entities hold references, but **table structure unchanged**. Only **owner side** has FK; inverse uses `mappedBy`.

**Default FK Naming**: `fieldName_id` (e.g., `userAddress_id`). Hibernate auto-selects referenced entity's **primary key**.

**Key Insight**: **Persistence Context manages relationships** during transaction. FK synchronization happens on `flush/commit`.

***

## **Unidirectional OneToOne**

Parent entity holds `@OneToOne` with `@JoinColumn` for FK control.

### **Theory**: Owner Side Controls Relationship

**Owner side** (UserDetails) owns the FK relationship. When you call `userRepository.save(userDetails)`, Hibernate:
1. Persists `UserDetails` to Persistence Context
2. Persists associated `UserAddress` (if `cascade=PERSIST`)
3. Creates FK `address_id` pointing to `UserAddress.id` **on commit/flush**

### Core Code

**UserDetails.java** (Owner/Parent):
```java
@Entity
@Table(name = "user_details")
public class UserDetails {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String phone;
    
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "address_id")  // Custom FK name
    private UserAddress userAddress;
    
    // getters/setters
}
```

**UserAddress.java** (Child - unaware of parent):
```java
@Entity
@Table(name = "user_address")
public class UserAddress {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String street, city, state, country, pinCode;
    // getters/setters
}
```

**Generated Schema**:
```sql
CREATE TABLE user_details (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    phone VARCHAR(255),
    address_id BIGINT,
    FOREIGN KEY (address_id) REFERENCES user_address(id)
);
```

**Usage**:
```java
UserDetails user = new UserDetails("John", "123456");
user.setUserAddress(new UserAddress("123 Main St", "Bangalore", ...));
userRepository.save(user);  // BOTH tables inserted!
```

***

## **Cascade Types - Complete Theory**

**Cascade = Propagate operations from parent → child**. Child lifecycle depends on parent.

### **JPA EntityManager Methods Mapped to Cascade**

| Cascade Type | EntityManager Method | Effect |
|--------------|---------------------|--------|
| `PERSIST` | `persist()` | Insert child when parent inserted |
| `MERGE` | `merge()` | Update child when parent updated |
| `REMOVE` | `remove()` | Delete child when parent deleted |
| `REFRESH` | `refresh()` | **Bypass cache**, reload child from DB |
| `DETACH` | `detach()` | Remove child from Persistence Context |
| `ALL` | All above | **Default for owned relationships** |

### **Lifecycle Dependency Theory**

**Child exists only because of parent**. Examples:
```
School ↔ Classroom (delete school → delete classrooms)
User ↔ Profile (delete user → delete profile)
Order ↔ ShippingAddress (delete order → delete address)
```

**Without Cascade**: Manual child management = **error-prone**:
```java
// ERROR-PRONE (No cascade)
userRepository.save(user);           // User saved
userAddressRepository.save(address); // Manual child save
```

**With Cascade**: Single operation handles both:
```java
userRepository.save(user);  // User + Address both saved!
```

### **Production Cascade Pattern**
```java
@OneToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE, CascadeType.REMOVE})
// Avoid CascadeType.ALL in complex relationships (REFRESH/DETACH rarely needed)
```

***

## **Fetch Strategies - Deep Theory**

**EAGER**: Load child **immediately** with parent (LEFT JOIN). **Default for @OneToOne/@ManyToOne**.

**LAZY**: Load child **on-demand** (Hibernate proxy). **Default for @OneToMany/@ManyToMany**.

### **Why These Defaults?**

| Relationship | # Children | Default | Rationale |
|--------------|------------|---------|-----------|
| `@OneToOne` | **1** | **EAGER** | Single child likely needed, low perf impact |
| `@ManyToOne` | **1** | **EAGER** | Same as OneToOne |
| `@OneToMany` | **Many** | **LAZY** | Unknown quantity, high perf risk |
| `@ManyToMany` | **Many** | **LAZY** | Same as OneToMany |

**SQL Patterns**:
```
EAGER:  SELECT u.*, a.* FROM user_details u 
           LEFT JOIN user_address a ON u.address_id = a.id

LAZY:   SELECT * FROM user_details WHERE id=?  -- Parent only
        SELECT * FROM user_address WHERE id=?   -- On access
```

***

## **Handling Lazy Loading in OneToOne Mappings**

**Problem**: `@OneToOne(fetch = LAZY)` + direct entity return → **Jackson LazyInitializationException**.

### **Root Cause Theory**

1. **Transaction ends** → `EntityManager` closes → Persistence Context destroyed
2. **Jackson serializes** response → accesses lazy `userAddress` → **no session available**
3. **Proxy fails**: `HibernateProxy` can't execute child query without open session

### **Issue Demonstration**

```java
// WRONG - Fails after transaction
@GetMapping("/{id}")
public UserDetails getUser(@PathVariable Long id) {
    return repository.findById(id).orElseThrow();  // Lazy proxy created
}
```

**Console Failure**:
```
SELECT * FROM user_details WHERE id=1
LazyInitializationException: could not initialize proxy - no Session
```

### **Solution 1: DTO Pattern (Recommended)**

**Explicit lazy trigger within transaction**:

```java
public class UserDetailsDTO {
    private Long id, userId;
    private String name, phone, street;
    
    public UserDetailsDTO(UserDetails entity) {
        this.id = entity.getId();
        this.name = entity.getName();
        this.phone = entity.getPhone();
        
        // TRIGGER LAZY LOAD (within transaction)
        UserAddress addr = entity.getUserAddress();
        this.street = (addr != null) ? addr.getStreet() : null;
        this.userId = (addr != null) ? addr.getId() : null;
    }
}

@GetMapping("/{id}")
public UserDetailsDTO getUser(@PathVariable Long id) {
    UserDetails user = repository.findById(id).orElseThrow();
    return new UserDetailsDTO(user);  // Safe!
}
```

**Console (Safe)**:
```
SELECT * FROM user_details WHERE id=?     -- 1. Parent
SELECT * FROM user_address WHERE id=?     -- 2. Child (triggered)
```

### **Solution 2: @Transactional(readOnly = true)**

**Extend transaction** for serialization:

```java
@Transactional(readOnly = true)  // Lazy access during serialization
@GetMapping("/{id}")
public UserDetails getUser(@PathVariable Long id) {
    return repository.findById(id).orElseThrow();
}
```

**Anti-pattern**: Controller transactions violate separation of concerns.

### **Solution 3: Hibernate.initialize()**

**Force initialization** (service layer):

```java
@Service
@Transactional(readOnly = true)
public class UserService {
    public UserDetails getUser(Long id) {
        UserDetails user = repository.findById(id).orElseThrow();
        Hibernate.initialize(user.getUserAddress());  // Force load
        return user;
    }
}
```

### **Solution Hierarchy**
```
1. DTOs  (Explicit control, API contract)
2. @Transactional(readOnly=true)  (Quick fix)
3. Hibernate.initialize()  (Manual)
4. EAGER  (Always loads)
5. @JsonIgnore  (Hides data)
```

***

## **Bidirectional OneToOne**

**Theory**: **Same FK as unidirectional**. Inverse side uses `mappedBy` (logical reference, no DB change).

### **Owner vs Inverse Side**

| Role | Annotation | FK? | Controls Relationship |
|------|------------|-----|---------------------|
| **Owner** | `@JoinColumn` | **Yes** | **Lifecycle owner** |
| **Inverse** | `mappedBy` | **No** | Maps to owner's field |

### Code

**UserDetails.java** (Owner):
```java
@Entity
public class UserDetails {
    // id, name, phone...
    
    @OneToOne(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    @JoinColumn(name = "address_id")  // FK column
    @JsonManagedReference
    private UserAddress userAddress;
}
```

**UserAddress.java** (Inverse):
```java
@Entity
public class UserAddress {
    // id, street...
    
    @OneToOne(mappedBy = "userAddress", fetch = FetchType.LAZY)
    @JsonBackReference
    private UserDetails userDetails;  // Logical reference only
}
```

**Schema**: **Identical** to unidirectional (FK only in `user_details`).

***

## **Best Practices: Bidirectional Cascade Types**

**Golden Rule**: **Cascade flows from owner → inverse only**. Never cascade both directions.

###  **Production Pattern**

```java
// OWNER (UserDetails) - Controls lifecycle
@OneToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE, CascadeType.REMOVE})
@JoinColumn(name = "address_id")
private UserAddress userAddress;

// INVERSE (UserAddress) - NO cascade
@OneToOne(mappedBy = "userAddress")
private UserDetails userDetails;
```

### **Why Owner-Only Cascade?**

```
Cascade Both Ways → Problems:
1. save(user) → saves address → address.save(user) → Infinite loop
2. delete(user) → deletes address → address.delete(user) → Duplicate delete
3. Orphan records if sync fails
```

### **Granular Control Matrix**

| Operation | When to Cascade | Example |
|-----------|-----------------|---------|
| `PERSIST` | New user + new address | `userRepository.save(new User())` |
| `MERGE` | Update existing user + address | `userRepository.save(existingUser)` |
| `REMOVE` | Delete user → delete address | `userRepository.delete(user)` |
| `REFRESH` | Rarely (cache bypass) | Internal Hibernate use |
| `DETACH` | Rarely (memory management) | Internal Hibernate use |

***

## **Fix Infinite Recursion: Bidirectional JSON**

**Problem**: `userAddress.userDetails.userAddress.userDetails...` infinite loop. 
### **Fix 1: @JsonManagedReference / @JsonBackReference (Recommended)**

```java
// Owner → Serializes child
@JsonManagedReference
@OneToOne(mappedBy = "userDetails", cascade = CascadeType.ALL)
private UserAddress userAddress;

// Inverse → Skips parent
@JsonBackReference
@OneToOne
@JoinColumn(name = "user_details_id")
private UserDetails userDetails;
```

**JSON Results**:
```json
GET /users/1:
{
  "id": 1, "name": "John",
  "userAddress": {"id": 2, "street": "123 Main St"}
}

GET /addresses/2:
{"id": 2, "street": "123 Main St"}  // No parent!
```

### **Fix 2: @JsonIdentityInfo (Bidirectional)**

```java
@Entity
@JsonIdentityInfo(
    generator = ObjectIdGenerators.PropertyGenerator.class,
    property = "id",
    scope = UserDetails.class  // Avoid conflicts
)
public class UserDetails { ... }

@Entity
@JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
public class UserAddress { ... }
```

**JSON**:
```json
{
  "id": 1,
  "userAddress": {"id": 2, "street": "123 Main St"}
}
```

### **Fix 3: DTO Pattern **

**No annotations needed**. Explicit mapping = no cycles:

```java
public class UserDetailsDTO {
    private Long id;
    private String name;
    private AddressDTO address;  // Nested DTO
    
    public UserDetailsDTO(UserDetails entity) {
        this.id = entity.getId();
        this.name = entity.getName();
        if (entity.getUserAddress() != null) {
            this.address = new AddressDTO(entity.getUserAddress());
        }
    }
}
```

***

## **Composite Key OneToOne**

**Child with `@EmbeddedId`** requires `@JoinColumns`:

```java
@Embeddable
public class AddressKey implements Serializable {
    private String street;
    private String pinCode;
    // equals(), hashCode()
}

@Entity
public class UserAddress {
    @EmbeddedId private AddressKey id;
    private String city;
}

@Entity
public class UserDetails {
    @OneToOne(cascade = CascadeType.ALL)
    @JoinColumns({
        @JoinColumn(name = "addr_street", referencedColumnName = "street"),
        @JoinColumn(name = "addr_pin", referencedColumnName = "pinCode")
    })
    private UserAddress userAddress;
}
```

***

## **Production Controller Example**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @PostMapping
    @Transactional
    public UserDetailsDTO create(@RequestBody CreateUserRequest req) {
        UserDetails user = userMapper.toEntity(req);
        return new UserDetailsDTO(userRepository.save(user));
    }
    
    @GetMapping("/{id}")
    @Transactional(readOnly = true)
    public UserDetailsDTO get(@PathVariable Long id) {
        UserDetails user = userRepository.findById(id).orElseThrow();
        return new UserDetailsDTO(user);
    }
    
    @DeleteMapping("/{id}")
    @Transactional
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        userRepository.deleteById(id);  // Cascades!
        return ResponseEntity.noContent().build();
    }
}
```

***

## **Key Theory Summary**

1. **Owner owns FK** → Cascade from owner only
2. **LAZY needs explicit access** → DTOs solve safely
3. **Bidirectional = unidirectional FK + inverse mapping**
4. **mappedBy = "ownerFieldName"** → logical reference
5. **Cascade propagates EntityManager operations**
6. **EAGER for single child, LAZY for collections**
7. **DTOs = API contract + lazy safety + no cycles**

**Production Checklist**:
```
 DTOs for all APIs
 LAZY everywhere
 Owner-side CascadeType.ALL
 @JsonManagedReference/@JsonBackReference
 mappedBy on inverse side
 @Transactional(readOnly=true) only if needed
```

**Self-Assessment**:
1. **LazyInitializationException** occurs when?
2. **mappedBy** references what field?
3. **Owner vs Inverse** cascade rule?
4. **DTO constructor** triggers what query?
5. **Why EAGER default** for OneToOne?
