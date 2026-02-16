JPA first-level caching is a built-in optimization mechanism that significantly improves performance by reducing database hits within a single transaction.

## **Topicss**

1. How the Persistence Context acts as first-level cache
2. Why fetching the same entity twice doesn't generate duplicate queries
3. The relationship between EntityManager and Persistence Context (1:1)
4. Why first-level cache is transaction-scoped and thread-isolated
5. Entity lifecycle states and their impact on caching
6. How HTTP requests create separate cache instances in Spring Boot
7. When first-level caching is used vs bypassed

## **First-Level Cache Fundamentals**

First-level caching is **mandatory** in JPA/Hibernate and operates at the **EntityManager level**. It's called "first-level" because it's the closest cache to your application code. [en.wikibooks](https://en.wikibooks.org/wiki/Java_Persistence/Caching)

### Core Principles

1. **1 EntityManager = 1 Persistence Context = 1 First-Level Cache** [baeldung](https://www.baeldung.com/jpa-hibernate-persistence-context)
2. **Thread-isolated**: Each thread/HTTP request gets its own cache [codingtechroom](https://codingtechroom.com/question/understanding-java-cdi-persistencecontext-thread-safety-concerns)
3. **Transaction-scoped**: Cache exists only during transaction lifetime [linkedin](https://www.linkedin.com/posts/saphalpathak_jpa-caching-first-level-and-second-level-activity-7231875496160993280-cRSR)
4. **Primary Key based**: Entities are cached by their ID (not other fields) [linkedin](https://www.linkedin.com/posts/deepak-yadav-b04998157_hibernate-first-level-cache-what-it-activity-7395426873159131138-rkIP)
5. **Automatic**: Enabled by default, no configuration needed [vladmihalcea](https://vladmihalcea.com/jpa-hibernate-first-level-cache/)

### Internal Storage

Hibernate stores entities in a **Map** structure:
```
Map<EntityClass, Map<PrimaryKey, EntityInstance>>
```

Example: `Map<UserDetails, Map<Long, UserDetails>>`

***

## **Concept Map**

```
HTTP Request 1 → EntityManager 1 → Persistence Context 1 (Cache 1)
                    ↓
                    ├─ persist(user1) → user1 stored in Cache 1
                    ├─ find(id=1) → Returns from Cache 1 (NO DB query)
                    └─ Transaction Commit → Flush to DB

HTTP Request 2 → EntityManager 2 → Persistence Context 2 (Cache 2)
                    ↓
                    ├─ find(id=1) → Cache 2 empty → DB query → Store in Cache 2
                    └─ find(id=1) → Returns from Cache 2 (NO DB query)

Key: Each HTTP Request = Separate Cache (Thread Isolation) [web:24][web:26]
```

***

## **Entity Lifecycle & Caching Interaction**

Understanding caching requires knowing entity states from previous material:
### Entity Lifecycle Review (Quick Reference)

```
Transient (new) → persist() → Managed → commit() → Detached
     ↓                    ↓              ↓
  (No Cache)       (Cache)      (Flush)    (Cache Cleared)
```

**Caching Behavior by State:**

| State | In Cache? | DB Sync? | Example |
|-------|-----------|----------|---------|
| **Transient** | No | No | `new UserDetails()` |
| **Managed** | Yes | On commit/flush | `entityManager.persist(user)` |
| **Removed** | Yes (marked) | On commit | `entityManager.remove(user)` |
| **Detached** | No | Already synced | After transaction ends |

**Key Insight**: **All caching operations happen in memory until commit/flush**. Database is only touched when transaction commits. [baeldung](https://www.baeldung.com/jpa-hibernate-persistence-context)

***

## **How First-Level Cache Works (Step-by-Step)**

### Scenario 1: Same Transaction, Same EntityManager

```java
@Transactional  // Creates EntityManager + Persistence Context
public UserDetails testCaching() {
    // 1. Save new entity (Transient → Managed)
    UserDetails user = new UserDetails("John", "john@test.com");
    userService.save(user);  // persist() → INSERT queued (not executed yet)
                             // user stored in Persistence Context cache
    
    // 2. Fetch same entity (same EntityManager)
    UserDetails found1 = userService.findById(user.getId());
    // NO SELECT QUERY! Returned from cache
    
    // 3. Fetch again
    UserDetails found2 = userService.findById(user.getId());
    // NO SELECT QUERY! Same cache instance
    // found1 == found2 (object identity preserved) [web:23][web:28]
    
    return found1;  // Transaction commits → Actual INSERT to DB
}
```

**Console Output:**
```
Hibernate: insert into user_details (name, email) values (?, ?)
```
**No SELECT queries!** Both finds return from cache. [stackoverflow](https://stackoverflow.com/questions/67580370/spring-boot-with-hibernate-first-level-cache)

### Scenario 2: Different HTTP Requests (Different EntityManagers)

**Request 1** (`/test-jpa` - save + find):
```java
// HTTP Request 1 → EntityManager 1 → Cache 1
// persist() → Cache 1 stores user
// find() → Cache 1 hit → NO SELECT
// Commit → INSERT to DB
```

**Request 2** (`/read-jpa` - find only):
```java
// HTTP Request 2 → EntityManager 2 → Cache 2 (EMPTY)
// find() → Cache 2 miss → SELECT from DB
// Cache 2 populated
// find() → Cache 2 hit → NO SELECT
```

**Console Output:**
```
Request 1: insert into user_details ...
Request 2: select from user_details where id=?  ← Only 1 SELECT!
```

**Why?** **New HTTP request = New EntityManager = New cache**. [discourse.hibernate](https://discourse.hibernate.org/t/1st-level-cache-size/5525)

### DispatcherServlet Magic

Spring Boot's **DispatcherServlet** automatically creates an **EntityManager per HTTP request**: [codingtechroom](https://codingtechroom.com/question/understanding-java-cdi-persistencecontext-thread-safety-concerns)

```
DispatcherServlet → Transaction Interceptor → EntityManager (per request)
    ↓
Controller → Service → Repository → EntityManager (same instance)
```

This ensures **thread safety** - each request has its isolated cache. [stackoverflow](https://stackoverflow.com/questions/58338509/is-jpa-persistence-context-isolated-between-threads)

***

## **Manual EntityManager Example**

To visualize better, create EntityManagers explicitly: 
```java
@Service
public class CacheDemoService {
    
    @PersistenceUnit  // Inject EntityManagerFactory
    private EntityManagerFactory emf;
    
    public void demonstrateManualCache() {
        // Session/EntityManager 1
        EntityManager em1 = emf.createEntityManager();
        EntityTransaction tx1 = em1.getTransaction();
        tx1.begin();
        
        UserDetails user = new UserDetails("John", "john@test.com");
        em1.persist(user);  // 1. Stored in em1's cache (INSERT queued)
        
        UserDetails found1 = em1.find(UserDetails.class, user.getId());
        // Cache hit → NO DB query
        
        UserDetails found2 = em1.find(UserDetails.class, user.getId());
        // Cache hit → NO DB query
        
        tx1.commit();  // 2. Flush → Actual INSERT to DB
        em1.close();   // 3. Cache cleared
        
        // Session/EntityManager 2 (isolated)
        EntityManager em2 = emf.createEntityManager();
        EntityTransaction tx2 = em2.getTransaction();
        tx2.begin();
        
        UserDetails found3 = em2.find(UserDetails.class, user.getId());
        // Cache miss → SELECT from DB → Populates em2's cache
        
        UserDetails found4 = em2.find(UserDetails.class, user.getId());
        // Cache hit → NO DB query
        
        tx2.commit();
        em2.close();
    }
}
```

**Console Output:**
```
Hibernate: insert into user_details ...  ← From em1 commit
Hibernate: select from user_details where id=?  ← From em2 first find
```

**Proof**: Different EntityManagers = Different caches = Isolated. [stackoverflow](https://stackoverflow.com/questions/58338509/is-jpa-persistence-context-isolated-between-threads)

***

## **Cache Behavior Rules**

### **Cache Hits (No DB Query)**

| Scenario | Same EntityManager? | Result |
|----------|-------------------|--------|
| `find(id)` twice | Yes | Cache hit |
| `persist()` → `find(id)` | Yes | Cache hit |
| Update managed entity | Yes | Dirty checking (no query needed) |
| `find(id)` → `find(id)` | Yes | Same object instance |

### **Cache Misses (DB Query)**

| Scenario | Same EntityManager? | Result |
|----------|-------------------|--------|
| Different HTTP requests | No | New cache → DB query |
| New EntityManager | No | New cache → DB query |
| Query by non-PK field | Always | DB query (but object reused if found)  [linkedin](https://www.linkedin.com/posts/deepak-yadav-b04998157_hibernate-first-level-cache-what-it-activity-7395426873159131138-rkIP) |

### Non-Primary Key Queries

```java
// These ALWAYS hit DB (not cached by name/email)
List<UserDetails> users = repository.findByName("John");
UserDetails user = repository.findByEmail("john@test.com");
```

**Exception**: If query returns an entity already in cache (by ID), Hibernate reuses the cached instance. [linkedin](https://www.linkedin.com/posts/deepak-yadav-b04998157_hibernate-first-level-cache-what-it-activity-7395426873159131138-rkIP)

***

## **Thread Safety & Isolation**

**EntityManager is NOT thread-safe**. Spring Boot solves this by: [stackoverflow](https://stackoverflow.com/questions/5113716/spring-persistencecontext-and-autowired-thread-safety)

1. **Request-scoped EntityManager**: Each HTTP request gets its own instance [discourse.hibernate](https://discourse.hibernate.org/t/1st-level-cache-size/5525)
2. **Transaction isolation**: Each transaction has separate Persistence Context [stackoverflow](https://stackoverflow.com/questions/58338509/is-jpa-persistence-context-isolated-between-threads)
3. **DispatcherServlet proxying**: Automatically manages EntityManager lifecycle [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/82314936/0f4e052b-5e02-450e-bccb-8c2b0a689bce/paste.txt)

**Guarantee**: Changes in Thread 1's Persistence Context are **never visible** to Thread 2. [stackoverflow](https://stackoverflow.com/questions/58338509/is-jpa-persistence-context-isolated-between-threads)

```
Thread 1 (Request 1): EntityManager1 → Cache1
Thread 2 (Request 2): EntityManager2 → Cache2
```

**Isolation ensures consistency** but means **no sharing across requests**. [codingtechroom](https://codingtechroom.com/question/understanding-java-cdi-persistencecontext-thread-safety-concerns)

***

## **First-Level Cache Benefits & Limitations**

###  **Benefits**

| Benefit | Impact |
|---------|--------|
| **Reduced DB hits** | Up to 90% fewer SELECTs  [vladmihalcea](https://vladmihalcea.com/jpa-hibernate-first-level-cache/) |
| **Object identity** | `user1 == user2` within transaction  [linkedin](https://www.linkedin.com/posts/deepak-yadav-b04998157_hibernate-first-level-cache-what-it-activity-7395426873159131138-rkIP) |
| **Dirty checking** | Auto-detects changes, no manual saves |
| **Transaction consistency** | Repeatable reads within transaction  [en.wikibooks](https://en.wikibooks.org/wiki/Java_Persistence/Caching) |
| **Automatic** | No configuration needed |

###  **Limitations**

| Limitation | Workaround |
|------------|------------|
| **Transaction-scoped only** | Use second-level cache for cross-transaction |
| **PK-only caching** | Queries by name/email always hit DB |
| **Memory usage** | Cache grows with entities loaded  [discourse.hibernate](https://discourse.hibernate.org/t/1st-level-cache-size/5525) |
| **No cross-request sharing** | Separate cache per HTTP request |

***

## **Summary**

JPA first-level caching is implemented through the **Persistence Context** - a mandatory, automatic cache tied to each **EntityManager**. Key takeaways: [en.wikibooks](https://en.wikibooks.org/wiki/Java_Persistence/Caching)

1. **1:1 relationship**: EntityManager ↔ Persistence Context ↔ First-Level Cache
2. **HTTP request scoped**: Each request gets isolated cache [discourse.hibernate](https://discourse.hibernate.org/t/1st-level-cache-size/5525)
3. **Transaction lifecycle**: Cache cleared at transaction end [linkedin](https://www.linkedin.com/posts/saphalpathak_jpa-caching-first-level-and-second-level-activity-7231875496160993280-cRSR)
4. **Primary key based**: Efficient for `findById()` operations [vladmihalcea](https://vladmihalcea.com/jpa-hibernate-first-level-cache/)
5. **Automatic flushing**: Changes synced to DB on commit [baeldung](https://www.baeldung.com/jpa-hibernate-persistence-context)

The magic happens because **save() and find() within the same transaction use the same EntityManager/cache**. Different requests create new caches, ensuring thread safety but requiring second-level caching for cross-request optimization. [stackoverflow](https://stackoverflow.com/questions/67580370/spring-boot-with-hibernate-first-level-cache)

***

## **Application: Cache Optimization Strategies**

### 1. **Batch Operations**
```java
@Transactional
public void batchSave(List<UserDetails> users) {
    users.forEach(repository::save);  // Single transaction = Single cache
    // Multiple finds reuse cache automatically
}
```

### 2. **Read-Heavy Operations**
```java
@Transactional(readOnly = true)  // Read-only tx = Optimized
public List<UserDetails> getAllUsers() {
    return repository.findAll();  // All cached during tx
}
```

### 3. **Avoid N+1 Problem**
```java
// BAD: Multiple queries
for(User u : users) {
    u.getOrders().size();  // N queries!
}

// GOOD: Single query + cache
users.forEach(u -> u.getOrders().size());  // Cache hits
```

***

## **Check these questionss:**

1. **Conceptual**: Why does `save()` + `findById()` generate only 1 query instead of 2?

2. **Lifecycle**: Trace an entity's journey: `new User() → persist() → findById() → commit()`. Which operations use cache?

3. **Architecture**: If two HTTP requests call `findById(1)`, how many SELECT queries execute? Why?

4. **Threading**: Thread 1 modifies User(id=1). Can Thread 2 see changes immediately? Why?

5. **Practical**: You have 3 finds by email in one transaction. How many DB queries? (Hint: non-PK)

6. **Debugging**: Logs show INSERT but no SELECT after save(). What's happening?

7. **Optimization**: Method does 10 `findById()` calls. How to minimize DB hits?

8. **Comparison**: First-level vs second-level cache - scope and lifetime?

**Next Topic**: Second-level caching for cross-transaction, cross-request optimization. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/82314936/0f4e052b-5e02-450e-bccb-8c2b0a689bce/paste.txt)
