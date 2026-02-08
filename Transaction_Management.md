

# Spring Boot – Transaction Management Notes

## 1. Transaction Management in Spring Boot

Transaction management ensures data consistency and reliability by following the **ACID properties**:

* **Atomicity** – A transaction is treated as a single unit; either all operations succeed or all fail.
* **Consistency** – Database moves from one valid state to another.
* **Isolation** – Concurrent transactions do not affect each other.
* **Durability** – Once committed, data remains permanent even after system failure.

Spring Boot provides **built-in transaction management** using Spring Framework abstractions.

---

## 2. Approaches to Transaction Management in Spring Boot

Spring supports two transaction management approaches:

1. Declarative Transaction Management
2. Programmatic Transaction Management

---

## 3. Declarative Transaction Management

Declarative transaction management is the **most preferred and widely used approach**.

### How it works

* Uses **annotations or XML configuration**
* Transaction logic is separated from business logic
* Managed internally using **Spring AOP (Aspect-Oriented Programming)**

### @Transactional Annotation

The `@Transactional` annotation is the core of declarative transaction management.

It can be applied at:

* Method level
* Class level (applies to all public methods)

```java
@Transactional
public void createOrder() {
    // business logic
}
```

### Transaction Lifecycle

* Transaction starts before method execution
* Commits if method completes successfully
* Rolls back if a runtime exception occurs

### Rollback Rules

* By default, rollback occurs for:

  * `RuntimeException`
  * `Error`
* Checked exceptions do not trigger rollback unless specified

```java
@Transactional(rollbackFor = Exception.class)
```

### Advantages

* Clean and readable code
* Minimal boilerplate
* Easy to maintain
* Suitable for most business applications

### Limitations

* Does not work on private methods
* Self-invocation within the same class does not trigger transaction
* Works only on public methods when using proxies

---

## 4. Programmatic Transaction Management

Programmatic transaction management allows **manual control** over transactions.

### How it works

* Developer explicitly controls transaction boundaries
* Uses:

  * `PlatformTransactionManager`
  * `TransactionTemplate`

### Example using TransactionTemplate

```java
transactionTemplate.execute(status -> {
    try {
        // business logic
    } catch (Exception e) {
        status.setRollbackOnly();
        throw e;
    }
    return null;
});
```

### Advantages

* Fine-grained control over transactions
* Useful when transaction behavior depends on complex conditions

### Disadvantages

* More boilerplate code
* Business logic mixed with transaction logic
* Harder to read and maintain

### Use Case

Used only when declarative transactions are insufficient or too restrictive.

---

## 5. Propagation Types

Propagation behavior defines **how a transactional method behaves when called from another transactional method**.

### Common Propagation Types

#### REQUIRED (Default)

* Joins existing transaction if present
* Creates a new transaction if none exists
* Most commonly used

```java
@Transactional(propagation = Propagation.REQUIRED)
```

---

#### REQUIRES_NEW

* Always creates a new transaction
* Suspends the existing transaction
* Used for independent operations such as logging

---

#### SUPPORTS

* Uses existing transaction if available
* Executes non-transactionally if no transaction exists

---

#### MANDATORY

* Requires an existing transaction
* Throws exception if no transaction exists

---

#### NOT_SUPPORTED

* Suspends current transaction
* Executes without a transaction

---

#### NEVER

* Throws exception if a transaction exists
* Ensures non-transactional execution

---

#### NESTED

* Creates a nested transaction using savepoints
* Inner transaction can roll back independently
* Supported only by certain databases (e.g., JDBC)

---

## 6. Isolation Levels

Isolation level controls **how and when the changes made by one transaction are visible to others**.

### Common Transaction Problems

* **Dirty Read** – Reading uncommitted data
* **Non-Repeatable Read** – Data changes between reads in the same transaction
* **Phantom Read** – New rows appear in repeated queries

---

## 7. Isolation Levels in Spring

### DEFAULT

* Uses the database’s default isolation level

```java
@Transactional(isolation = Isolation.DEFAULT)
```

---

### READ_UNCOMMITTED

* Allows dirty reads
* Lowest isolation level
* Fast but unsafe

---

### READ_COMMITTED

* Allows only committed data to be read
* Prevents dirty reads
* Non-repeatable reads may occur
* Most commonly used in production systems

---

### REPEATABLE_READ

* Ensures repeated reads return the same data
* Prevents dirty and non-repeatable reads
* Phantom reads may still occur

---

### SERIALIZABLE

* Highest isolation level
* Transactions execute sequentially
* Prevents dirty, non-repeatable, and phantom reads
* Lowest performance due to locking

---

### Isolation Levels Summary Table

| Isolation Level  | Dirty Read | Non-Repeatable Read | Phantom Read |
| ---------------- | ---------- | ------------------- | ------------ |
| READ_UNCOMMITTED | Allowed    | Allowed             | Allowed      |
| READ_COMMITTED   | Prevented  | Allowed             | Allowed      |
| REPEATABLE_READ  | Prevented  | Prevented           | Allowed      |
| SERIALIZABLE     | Prevented  | Prevented           | Prevented    |

---

## 8. @Transactional Annotation – Important Attributes

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED,
    rollbackFor = Exception.class,
    timeout = 30,
    readOnly = false
)
```

### Attribute Explanation

* `propagation`: Defines transaction boundary behavior
* `isolation`: Defines data visibility between transactions
* `rollbackFor`: Specifies exceptions that trigger rollback
* `timeout`: Maximum time allowed for transaction execution
* `readOnly`: Optimization hint for read-only operations

---

## 9. Key Interview and Exam Points

* Declarative transaction management is preferred over programmatic
* `@Transactional` works using Spring AOP proxies
* Self-invocation does not trigger transactional behavior
* Default propagation is REQUIRED
* Default isolation is DEFAULT
* Rollback occurs only for unchecked exceptions by default
* Nested transactions depend on database support

---


# Real-World Use Cases for Transaction Propagation Types

## 1. REQUIRED (Default)

### Description

* Joins an existing transaction if one exists
* Creates a new transaction if none exists

### Real-World Use Case

**Order Processing System**

```java
placeOrder()
   ├── saveOrder()
   ├── saveOrderItems()
   └── updateInventory()
```

All operations must succeed or fail together.

### Why REQUIRED?

* If `saveOrderItems()` fails, the entire order must roll back
* Ensures atomicity of business operation

### Example

```java
@Transactional(propagation = Propagation.REQUIRED)
public void saveOrder() { }
```

Used in:

* Payment processing
* Account creation
* Business workflows where all steps are dependent

---

## 2. REQUIRES_NEW

### Description

* Always starts a new transaction
* Suspends the existing transaction

### Real-World Use Case

**Audit Logging or Notification Service**

Even if the main transaction fails, logs must be saved.

```java
processPayment()
   ├── debitAmount()
   └── saveAuditLog()   // must commit independently
```

### Why REQUIRES_NEW?

* Audit log should persist even if payment fails
* Failure in logging should not affect main business logic

### Example

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveAuditLog() { }
```

Used in:

* Audit trails
* Error logging
* Email/SMS notification tracking

---

## 3. SUPPORTS

### Description

* Uses existing transaction if present
* Runs without a transaction if none exists

### Real-World Use Case

**Optional Transactional Read Operations**

```java
getUserDetails()
```

* Called sometimes from transactional service
* Called sometimes directly (non-transactional)

### Why SUPPORTS?

* No need to force a transaction
* Improves performance for read operations

### Example

```java
@Transactional(propagation = Propagation.SUPPORTS)
public User getUserById(Long id) { }
```

Used in:

* Read-only services
* Reporting queries
* Cache lookups

---

## 4. MANDATORY

### Description

* Must run inside an existing transaction
* Throws exception if no transaction exists

### Real-World Use Case

**Critical Internal Method**

```java
processLoan()
   └── updateAccountBalance()   // must not run alone
```

### Why MANDATORY?

* Prevents accidental execution without transaction
* Ensures data integrity

### Example

```java
@Transactional(propagation = Propagation.MANDATORY)
public void updateAccountBalance() { }
```

Used in:

* Core financial updates
* Sensitive database operations
* Low-level DAO methods

---

## 5. NOT_SUPPORTED

### Description

* Suspends current transaction
* Executes non-transactionally

### Real-World Use Case

**External API Calls**

```java
placeOrder()
   ├── saveOrder()
   └── callShippingService()   // should not be transactional
```

### Why NOT_SUPPORTED?

* External calls should not participate in DB transaction
* Prevents long-running transactions

### Example

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void callShippingService() { }
```

Used in:

* REST API calls
* File I/O
* Messaging systems

---

## 6. NEVER

### Description

* Throws exception if a transaction exists

### Real-World Use Case

**System Monitoring or Health Check**

```java
checkSystemHealth()
```

Should never run inside a transaction.

### Why NEVER?

* Prevents misuse inside transactional flows
* Ensures lightweight execution

### Example

```java
@Transactional(propagation = Propagation.NEVER)
public void healthCheck() { }
```

Used in:

* Health checks
* Monitoring endpoints
* Cache warm-up jobs

---

## 7. NESTED

### Description

* Creates a nested transaction using savepoints
* Allows partial rollback

### Real-World Use Case

**Bulk Processing with Partial Failure Handling**

```java
processBulkOrders()
   ├── order1 (success)
   ├── order2 (fails → rollback only this)
   └── order3 (success)
```

### Why NESTED?

* Failure in one item should not affect others
* Parent transaction remains active

### Example

```java
@Transactional(propagation = Propagation.NESTED)
public void processSingleOrder() { }
```

Used in:

* Batch processing
* Bulk imports
* Multi-step workflows

Note:

* Requires database support for savepoints
* Not supported by all transaction managers

---

## Quick Interview Mapping

| Propagation   | Typical Use Case        |
| ------------- | ----------------------- |
| REQUIRED      | Business workflows      |
| REQUIRES_NEW  | Logging, audit          |
| SUPPORTS      | Read-only operations    |
| MANDATORY     | Critical internal logic |
| NOT_SUPPORTED | External services       |
| NEVER         | Monitoring              |
| NESTED        | Batch processing        |

---



