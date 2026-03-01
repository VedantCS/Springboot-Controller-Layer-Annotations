## Spring boot: JPA | OneToMany, ManyToOne and ManyToMany, OneToOne
1. `@OneToMany` unidirectional (join table vs `@JoinColumn`)
2. `@OneToMany` bidirectional (`mappedBy` on parent)
3. `@ManyToOne` (child → parent, often FK owner)
4. `@ManyToMany` unidirectional/bidirectional (`@JoinTable`)
5. **OrphanRemoval** vs Cascade.REMOVE
6. **N+1 problem** with LAZY fetch
7. Bidirectional **sync requirements** (set both sides)
8. **DTOs** for serialization safety 
## **Relationship Fundamentals**

**Parent-Child**: Child existence depends on parent (cascade deletes child on parent delete). User (parent) → Orders (children). 
**Owner Side**: Holds **physical FK** (`@JoinColumn`). Controls persistence.

**Inverse Side**: Logical reference (`mappedBy`). No FK column.

**OneToMany Default**: **Join table** (separate association table). Use `@JoinColumn` for **direct FK in child table**. [stackoverflow](https://stackoverflow.com/questions/11938253/jpa-joincolumn-vs-mappedby)

Spring Boot JPA `@OneToMany`, `@ManyToOne`, and `@ManyToMany` mappings handle parent-child and many-to-many relationships, with unidirectional (one-way) or bidirectional (two-way) navigation using FKs, join tables, and annotations like `@JoinColumn`, `mappedBy`, and `@JoinTable`.

**Fetch Defaults**:
| Mapping | Default Fetch |
|---------|---------------|
| `@OneToMany/@ManyToMany` | **LAZY** |
| `@ManyToOne/@OneToOne` | **EAGER** |

***

## **OneToMany Unidirectional**

**Parent** (UserDetails) → **List<Child>** (OrderDetails). Child unaware of parent.

### **Default: Join Table**
**UserDetails.java**:
```java
@Entity
public class UserDetails {
    @Id @GeneratedValue private Long id;
    private String name, phone;
    
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    // Default: Creates user_details_order_details table
    private List<OrderDetails> orders = new ArrayList<>();
    // getters/setters
}
```

**Schema**: 
```sql
-- user_details: id, name, phone
-- order_details: id, productName
-- user_details_order_details: user_details_id, order_details_id (join table)
```

**Console** (Save):
```
insert into user_details ...
insert into order_details ...
insert into user_details_order_details (user_details_id, order_details_id) values (...)
```

### **Optimized: Direct FK in Child** (Recommended) 
```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "user_id_fk")  // FK in order_details table!
private List<OrderDetails> orders = new ArrayList<>();
```

**Schema**: 
```sql
-- order_details: id, productName, user_id_fk (FK to user_details.id)
-- NO join table!
```

**Why?** Normalized (no extra table), faster queries. [vladmihalcea](https://vladmihalcea.com/the-best-way-to-map-a-onetomany-association-with-jpa-and-hibernate/)

**Usage**:
```java
UserDetails user = new UserDetails("John", "123");
user.getOrders().add(new OrderDetails("Ice Cream"));
user.getOrders().add(new OrderDetails("Cold Drink"));
userRepository.save(user);  // User + 2 Orders inserted!
```

***

## **OrphanRemoval: Unique to Collections**

**Problem**: Remove child from `List` → FK becomes NULL (orphan), but row survives.
**Without orphanRemoval**:
```java
UserDetails user = userRepository.findById(1L).get();
user.getOrders().remove(0);  // Remove first order
userRepository.save(user);   // UPDATE order_details SET user_id_fk = NULL
-- Orphan row remains!
```

**With orphanRemoval=true**: 
```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "user_id_fk")
private List<OrderDetails> orders;
```

**Result**:
```java
user.getOrders().remove(0);
userRepository.save(user);  // DELETE from order_details WHERE id=?
```

**orphanRemoval vs Cascade.REMOVE**:
| Feature | orphanRemoval | Cascade.REMOVE |
|---------|---------------|----------------|
| Trigger | Remove from **collection** | Delete **parent** |
| Effect | **Auto-delete** orphan child | Delete all children |
| Scope | **Collections only** | Any association |

***

## **OneToMany Bidirectional** (= ManyToOne Bidirectional)

**FK Owner**: **Child** (OrderDetails holds `user_id_fk`). Parent uses `mappedBy`. [stackoverflow](https://stackoverflow.com/questions/11938253/jpa-joincolumn-vs-mappedby)

### **Code** 
**OrderDetails.java** (Owner - ManyToOne):
```java
@Entity
public class OrderDetails {
    @Id @GeneratedValue private Long id;
    private String productName;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id_fk")  // FK here!
    private UserDetails user;
    // getters/setters
}
```

**UserDetails.java** (Inverse - OneToMany):
```java
@OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
@JsonManagedReference
private List<OrderDetails> orders = new ArrayList<>();
```

**Schema**: Same as unidirectional (`user_id_fk` in `order_details`).

### **Bidirectional Sync Required** 
**WRONG** (Null FK):
```java
UserDetails user = new UserDetails("John");
OrderDetails order1 = new OrderDetails("Ice Cream");
user.getOrders().add(order1);  // Only parent side!
userRepository.save(user);     // user_id_fk = NULL!
```

**CORRECT**: 
```java
UserDetails user = new UserDetails("John");
OrderDetails order1 = new OrderDetails("Ice Cream");
order1.setUser(user);      // Owner side (FK)!
user.getOrders().add(order1);  // Inverse side
userRepository.save(user);     // FK populated!
```

**Helper Method**:
```java
public void addOrder(OrderDetails order) {
    orders.add(order);
    order.setUser(this);
}
```

***

## **ManyToOne Unidirectional**

**Child** → **Single Parent**. Often standalone FK owner.

**OrderDetails.java**:
```java
@ManyToOne(cascade = CascadeType.PERSIST)
@JoinColumn(name = "user_id_fk")
private UserDetails user;  // No parent List<Orders>
```

**Schema**: Identical to above.

**Note**: `@OneToMany` unidirectional + `@ManyToOne` unidirectional = **two independent mappings** using same FK. [stackoverflow](https://stackoverflow.com/questions/11938253/jpa-joincolumn-vs-mappedby)

***

## **ManyToMany Unidirectional**

**No parent-child**. Independent entities. **Always join table**. 
**OrderDetails → List<ProductDetails>**:

**OrderDetails.java**:
```java
@Entity
public class OrderDetails {
    @Id @GeneratedValue private Long id;
    
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "order_product",           // Join table name
        joinColumns = @JoinColumn(name = "order_id"),
        inverseJoinColumns = @JoinColumn(name = "product_id")
    )
    private List<ProductDetails> products = new ArrayList<>();
}
```

**Schema**: 
```sql
-- order_product: order_id (FK), product_id (FK)
```

**Usage**: 
```java
// Existing products
ProductDetails p1 = productRepo.findById(1L).get();  // Ice Cream
ProductDetails p2 = productRepo.findById(2L).get();  // Cold Drink

OrderDetails order = new OrderDetails();
order.getProducts().add(p1);
order.getProducts().add(p2);
orderRepo.save(order);  // Populates order_product!
```

**Console**:
```
insert into order_product (order_id, product_id) values (1,1), (1,2)
```

***

## **ManyToMany Bidirectional**

**Owner** (`@JoinTable`) manages join table. **Inverse** uses `mappedBy`.
**OrderDetails.java** (Owner):
```java
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
@JoinTable(name = "order_product",
           joinColumns = @JoinColumn(name = "order_id"),
           inverseJoinColumns = @JoinColumn(name = "product_id"))
@JsonManagedReference
private List<ProductDetails> products;
```

**ProductDetails.java** (Inverse):
```java
@ManyToMany(mappedBy = "products")
@JsonBackReference
private List<OrderDetails> orders;
```

**Sync Required**: 
```java
order.getProducts().add(product);
product.getOrders().add(order);  // Both sides!
```

**Choose Owner**: Business logic dictates (e.g., Order owns products). 
***

## **Lazy Loading & N+1 Problem**

**Default LAZY** → **N+1 queries**. 
**Problem**:
```java
List<UserDetails> users = userRepo.findAll();  // 1 query
users.forEach(u -> u.getOrders().size());     // N queries!
```

**Fixes**:
1. **JOIN FETCH**: `@Query("SELECT u FROM UserDetails u JOIN FETCH u.orders")`
2. **EntityGraph**: `@EntityGraph(attributePaths = "orders")`
3. **DTO Projection**: `SELECT new UserDTO(u.id, u.name, o) FROM UserDetails u JOIN u.orders o`

**EAGER Demo**:
```java
@OneToMany(fetch = FetchType.EAGER)  // Single JOIN query!
```

***

## **Cascade Best Practices**

| Mapping | Recommended Cascade | OrphanRemoval? |
|---------|---------------------|----------------|
| `@OneToMany` (parent→children) | `{PERSIST, MERGE, REMOVE}` | **true** |
| `@ManyToOne` (child→parent) | `{PERSIST}` | N/A |
| `@ManyToMany` | `{PERSIST, MERGE}` | **false** (no orphans) |

**Avoid Cascade.REMOVE on ManyToMany** (deletes independent entities). [dev](https://dev.to/jhonifaber/hibernate-onetoone-onetomany-manytoone-and-manytomany-8ba)

***

## **JSON Recursion Fixes** 
**Bidirectional Loop**: User→Orders→User...

1. **`@JsonManagedReference/@JsonBackReference`**:
   ```java
   // UserDetails (parent)
   @JsonManagedReference
   @OneToMany(mappedBy = "user")
   private List<OrderDetails> orders;
   
   // OrderDetails (child)
   @JsonBackReference
   @ManyToOne
   private UserDetails user;
   ```

2. **`@JsonIdentityInfo`** (both sides):
   ```java
   @JsonIdentityInfo(generator = ObjectIdGenerators.PropertyGenerator.class, property = "id")
   ```

3. **DTOs** (safest).

***

## **Production Controller + DTO**

```java
@PostMapping("/users")
public UserDetailsDTO create(@RequestBody CreateUserDTO dto) {
    UserDetails user = new UserDetails(dto.getName(), dto.getPhone());
    dto.getOrders().forEach(o -> {
        OrderDetails order = new OrderDetails(o.getProductName());
        order.setUser(user);  // Sync!
        user.getOrders().add(order);
    });
    return new UserDetailsDTO(userRepository.save(user));
}

public class UserDetailsDTO {
    private Long id;
    private String name;
    private List<OrderDTO> orders;
    
    public UserDetailsDTO(UserDetails user) {
        this.id = user.getId();
        this.name = user.getName();
        this.orders = user.getOrders().stream()
            .map(OrderDTO::new)
            .collect(Collectors.toList());  // Triggers LAZY safely
    }
}
```

***

## **Summary ** 
1. **OneToMany Default**: Join table → Use `@JoinColumn` for child FK (optimized)
2. **Bidirectional**: **Child owns FK** (`@ManyToOne` + `@JoinColumn`). Parent: `@OneToMany(mappedBy="user")`
3. **OrphanRemoval**: Auto-delete children removed from collection (not Cascade.REMOVE)
4. **ManyToMany**: **Always join table** (`@JoinTable`). Choose owner by business logic
5. **Sync Bidirectional**: Set **both sides** or use helpers
6. **LAZY Default**: Use DTOs/EntityGraph to avoid N+1
7. **Cascade Owner-Only**: Parent cascades to children

**Interview Points**:
- **orphanRemoval vs Cascade.REMOVE**: Collection removal vs parent delete
- **Owner = FK holder**: Child in OneToMany [stackoverflow](https://stackoverflow.com/questions/11938253/jpa-joincolumn-vs-mappedby)
- **mappedBy**: Inverse side field name

**Practice set**: Build User→Orders with bidirectional + DTOs + orphanRemoval=true. 

## Simple SpringBoot Project for Reference: 

**Relationships:**
* **Student ↔ Address** (OneToOne)
* **Student → Orders** (OneToMany / ManyToOne)
* **Student → Department** (ManyToOne)
* **Student ↔ Course** (ManyToMany)

---

# 1. Project Structure

```
com.example.school
│
├── entity
│     ├── Student.java
│     ├── Address.java
│     ├── Order.java
│     ├── Department.java
│     └── Course.java
│
├── repository
│     ├── StudentRepository.java
│     ├── AddressRepository.java
│     ├── OrderRepository.java
│     ├── DepartmentRepository.java
│     └── CourseRepository.java
│
├── service
│     └── StudentService.java
│
├── controller
│     └── StudentController.java
│
└── dto
      ├── StudentDTO.java
      └── CreateStudentRequest.java
```

---

# 2. Entities Layer

---

## 2.1 Student Entity

```java
@Entity
@Table(name = "students")
public class Student {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    /* =======================
       OneToOne (Student ↔ Address)
       ======================= */
    @OneToOne(mappedBy = "student",
              cascade = CascadeType.ALL,
              orphanRemoval = true)
    private Address address;

    /* =======================
       ManyToOne (Student → Department)
       ======================= */
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    /* =======================
       OneToMany (Student → Orders)
       ======================= */
    @OneToMany(mappedBy = "student",
               cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();

    /* =======================
       ManyToMany (Student ↔ Course)
       ======================= */
    @ManyToMany
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private List<Course> courses = new ArrayList<>();

    /* =======================
       Helper Methods
       ======================= */

    public void setAddress(Address address) {
        this.address = address;
        address.setStudent(this);
    }

    public void addOrder(Order order) {
        orders.add(order);
        order.setStudent(this);
    }

    public void enrollCourse(Course course) {
        courses.add(course);
        course.getStudents().add(this);
    }

    // getters and setters
}
```

---

## 2.2 Address (Owner of OneToOne)

```java
@Entity
@Table(name = "addresses")
public class Address {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String street;
    private String city;

    @OneToOne
    @JoinColumn(name = "student_id", unique = true)
    private Student student;

    // getters and setters
}
```

---

## 2.3 Department

```java
@Entity
@Table(name = "departments")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "department")
    private List<Student> students = new ArrayList<>();

    // getters and setters
}
```

---

## 2.4 Order (Owner of ManyToOne)

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String itemName;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "student_id")
    private Student student;

    // getters and setters
}
```

---

## 2.5 Course

```java
@Entity
@Table(name = "courses")
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @ManyToMany(mappedBy = "courses")
    private List<Student> students = new ArrayList<>();

    // getters and setters
}
```

---

# 3. Repository Layer

```java
public interface StudentRepository extends JpaRepository<Student, Long> {

    @EntityGraph(attributePaths = {"orders", "courses"})
    List<Student> findAll();
}
```

```java
public interface DepartmentRepository extends JpaRepository<Department, Long> {}
public interface CourseRepository extends JpaRepository<Course, Long> {}
public interface AddressRepository extends JpaRepository<Address, Long> {}
public interface OrderRepository extends JpaRepository<Order, Long> {}
```

---

# 4. Service Layer

```java
@Service
@Transactional
public class StudentService {

    private final StudentRepository studentRepo;
    private final DepartmentRepository deptRepo;
    private final CourseRepository courseRepo;

    public StudentService(StudentRepository studentRepo,
                          DepartmentRepository deptRepo,
                          CourseRepository courseRepo) {
        this.studentRepo = studentRepo;
        this.deptRepo = deptRepo;
        this.courseRepo = courseRepo;
    }

    public StudentDTO createStudent(CreateStudentRequest request) {

        Student student = new Student();
        student.setName(request.getName());

        Department dept = deptRepo.findById(request.getDepartmentId())
                .orElseThrow();
        student.setDepartment(dept);

        Address address = new Address();
        address.setStreet(request.getStreet());
        address.setCity(request.getCity());
        student.setAddress(address);

        request.getOrders().forEach(item -> {
            Order order = new Order();
            order.setItemName(item);
            student.addOrder(order);
        });

        request.getCourseIds().forEach(id -> {
            Course course = courseRepo.findById(id).orElseThrow();
            student.enrollCourse(course);
        });

        return new StudentDTO(studentRepo.save(student));
    }

    public List<StudentDTO> getAll() {
        return studentRepo.findAll()
                .stream()
                .map(StudentDTO::new)
                .toList();
    }
}
```

---

# 5. Controller Layer

```java
@RestController
@RequestMapping("/students")
public class StudentController {

    private final StudentService service;

    public StudentController(StudentService service) {
        this.service = service;
    }

    @PostMapping
    public StudentDTO create(@RequestBody CreateStudentRequest request) {
        return service.createStudent(request);
    }

    @GetMapping
    public List<StudentDTO> getAll() {
        return service.getAll();
    }
}
```

---

# 6. DTO Layer

## CreateStudentRequest

```java
public class CreateStudentRequest {

    private String name;
    private Long departmentId;
    private String street;
    private String city;
    private List<String> orders;
    private List<Long> courseIds;

    // getters and setters
}
```

## StudentDTO

```java
public class StudentDTO {

    private Long id;
    private String name;
    private String department;
    private List<String> orders;
    private List<String> courses;

    public StudentDTO(Student student) {
        this.id = student.getId();
        this.name = student.getName();
        this.department = student.getDepartment().getName();
        this.orders = student.getOrders()
                .stream()
                .map(Order::getItemName)
                .toList();
        this.courses = student.getCourses()
                .stream()
                .map(Course::getTitle)
                .toList();
    }
}
```

---

# 7. Full SQL Logs for Each Relationship

Assume Hibernate SQL logging enabled.

---

## 7.1 OneToOne Insert (Student + Address)

```sql
insert into students (name, department_id) values ('John', 1);

insert into addresses (street, city, student_id)
values ('Main St', 'NY', 1);
```

---

## 7.2 ManyToOne (Student → Department)

```sql
select department0_.id as id1_0_0_, department0_.name as name2_0_0_
from departments department0_ where department0_.id=1;

insert into students (name, department_id)
values ('John', 1);
```

---

## 7.3 OneToMany (Student → Orders)

```sql
insert into students (name, department_id) values ('John', 1);

insert into orders (item_name, student_id)
values ('Laptop', 1);

insert into orders (item_name, student_id)
values ('Book', 1);
```

---

## 7.4 ManyToMany (Student ↔ Course)

```sql
insert into students (name, department_id) values ('John', 1);

insert into student_course (student_id, course_id)
values (1, 2);

insert into student_course (student_id, course_id)
values (1, 3);
```

---

## 7.5 N+1 Example

```sql
select * from students;  -- 1 query

select * from orders where student_id=1;
select * from orders where student_id=2;
select * from orders where student_id=3;
```

N additional queries.

---

## Fixed with EntityGraph

```sql
select s.*, o.*
from students s
left join orders o on s.id = o.student_id;
```

Single optimized query.

---

# 8. Architectural Notes

* `@ManyToOne` always owns the foreign key.
* Parent controls lifecycle in OneToMany.
* Use `orphanRemoval` for dependent entities.
* Avoid `CascadeType.REMOVE` in ManyToMany.
* Always use helper methods for bidirectional consistency.
* Prefer LAZY + EntityGraph or JOIN FETCH.
* Always expose DTOs from controllers.

---



