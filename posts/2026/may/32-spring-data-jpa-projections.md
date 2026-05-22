# Post #32: Spring Data JPA Projections - Query Only What You Need

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** May 24, 2026  
**Topic:** Spring Data JPA, Projections, DTOs, Performance Optimization

---

## The Problem

Loading entire entities wastes memory and bandwidth. Fetch only what you need.

## Code Example

### ❌ Loading Full Entities - Wasteful

```java
@Entity
public class User {
    @Id
    private Long id;
    private String firstName;
    private String lastName;
    private String email;
    private String phone;
    private String address;
    private LocalDate birthDate;
    
    @Lob
    private byte[] profilePicture;  // Large Binary Data!
    
    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;  // Potentially Hundreds of Orders
}

// Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByLastName(String lastName);
}

// Controller - Only Needs Name and Email!
@GetMapping("/users")
public List<UserDTO> getUsers() {
    List<User> users = userRepository.findByLastName("Smith");
    
    // Loading Profile Pictures, Orders - All Wasted!
    return users.stream()
        .map(u -> new UserDTO(u.getFirstName(), u.getEmail()))
        .collect(Collectors.toList());
}
```

### ✅ Interface-Based Projection - Fetch Only Required Fields

```java
// Projection Interface
public interface UserNameView {
    String getFirstName();
    String getLastName();
    String getEmail();
}

// Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<UserNameView> findByLastName(String lastName);
}

// Generated SQL - Only Selected Columns!
// SELECT u.first_name, u.last_name, u.email FROM users u WHERE u.last_name = ?
```

### ✅ Class-Based Projection - DTOs with Constructor

```java
// DTO Class
public class UserDTO {
    private final String firstName;
    private final String email;
    
    public UserDTO(String firstName, String email) {
        this.firstName = firstName;
        this.email = email;
    }
    
    // Getters
}

// Repository with Constructor Expression
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT new com.example.dto.UserDTO(u.firstName, u.email) " +
           "FROM User u WHERE u.lastName = :lastName")
    List<UserDTO> findDTOByLastName(@Param("lastName") String lastName);
}
```

### ✅ Record-Based Projection - Modern Java Approach

```java
// Record DTO (Java 16+)
public record UserSummary(String firstName, String lastName, String email) {}

// Spring Data JPA Auto-Generates Constructor Expression!
public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findByLastName(String lastName);
    
    // OR with @Query
    @Query("SELECT u.firstName, u.lastName, u.email FROM User u WHERE u.active = true")
    List<UserSummary> findActiveUsers();
}
```

### ✅ Dynamic Projections - Runtime Type Selection

```java
public interface UserRepository extends JpaRepository<User, Long> {
    <T> List<T> findByLastName(String lastName, Class<T> type);
}

// Usage - Choose Projection at Runtime
List<User> fullEntities = userRepository.findByLastName("Smith", User.class);
List<UserNameView> names = userRepository.findByLastName("Smith", UserNameView.class);
List<UserSummary> summaries = userRepository.findByLastName("Smith", UserSummary.class);
```

### ✅ Open Projections - Computed Properties with SpEL

```java
public interface UserFullNameView {
    String getFirstName();
    String getLastName();
    
    // Computed Property Using SpEL
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
    
    // Transform to Uppercase
    @Value("#{target.email.toUpperCase()}")
    String getEmailUpperCase();
}

// Usage
List<UserFullNameView> users = userRepository.findBy();
users.forEach(u -> System.out.println(u.getFullName()));
```

### ✅ Nested Projections - Join and Project

```java
@Entity
public class Order {
    @Id
    private Long id;
    private BigDecimal total;
    
    @ManyToOne
    private User user;
}

// Nested Projection
public interface OrderWithUserView {
    Long getId();
    BigDecimal getTotal();
    UserNameView getUser();  // Nested Projection!
}

// Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<OrderWithUserView> findByTotalGreaterThan(BigDecimal amount);
}

// Generated SQL - JOIN with Selected Columns Only
// SELECT o.id, o.total, u.first_name, u.last_name, u.email
// FROM orders o JOIN users u ON o.user_id = u.id
// WHERE o.total > ?
```

### ✅ Projection with Aggregations

```java
public interface CategorySalesView {
    String getCategory();
    Long getTotalOrders();
    BigDecimal getTotalRevenue();
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o.category AS category, " +
           "COUNT(o) AS totalOrders, " +
           "SUM(o.total) AS totalRevenue " +
           "FROM Order o " +
           "GROUP BY o.category")
    List<CategorySalesView> getSalesByCategory();
}
```

### ✅ Native Query with Projection

```java
public interface MonthlyRevenueView {
    Integer getMonth();
    Integer getYear();
    BigDecimal getRevenue();
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query(value = "SELECT MONTH(order_date) AS month, " +
                   "YEAR(order_date) AS year, " +
                   "SUM(total) AS revenue " +
                   "FROM orders " +
                   "GROUP BY YEAR(order_date), MONTH(order_date)",
           nativeQuery = true)
    List<MonthlyRevenueView> getMonthlyRevenue();
}
```

### ✅ Projection Performance Comparison

```java
// ❌ Full Entity - 500ms Query, 50MB Memory
@GetMapping("/users-full")
public List<User> getFullUsers() {
    return userRepository.findAll();  // Loads Everything!
}

// ✅ Projection - 120ms Query, 5MB Memory
@GetMapping("/users-summary")
public List<UserSummary> getUserSummaries() {
    return userRepository.findBy();  // Only Selected Fields
}
```

### ✅ Combining Projections with Filtering

```java
public interface ActiveUserView {
    String getFirstName();
    String getEmail();
    boolean isActive();
}

public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT u.firstName, u.email, u.active " +
           "FROM User u WHERE u.active = true AND u.lastLoginDate > :since")
    List<ActiveUserView> findRecentActiveUsers(@Param("since") LocalDate since);
}
```

### ❌ Common Pitfall - N+1 with Lazy Collections

```java
// Projection Interface
public interface UserWithOrdersView {
    String getFirstName();
    List<Order> getOrders();  // Lazy Collection!
}

// ❌ Bad - N+1 Queries
List<UserWithOrdersView> users = userRepository.findAll();
users.forEach(u -> {
    u.getOrders().size();  // Separate Query Per User!
});

// ✅ Fix - Use @EntityGraph or JOIN FETCH
public interface UserRepository extends JpaRepository<User, Long> {
    @EntityGraph(attributePaths = "orders")
    List<UserWithOrdersView> findAll();
}
```

### ✅ Real-World Example - Dashboard API

```java
// Dashboard Summary Projection
public record DashboardSummary(
    long totalUsers,
    long activeUsers,
    BigDecimal totalRevenue,
    long pendingOrders
) {}

@Service
public class DashboardService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    public DashboardSummary getDashboard() {
        long totalUsers = userRepository.count();
        long activeUsers = userRepository.countByActive(true);
        
        BigDecimal revenue = orderRepository.getTotalRevenue();
        long pending = orderRepository.countByStatus(OrderStatus.PENDING);
        
        return new DashboardSummary(totalUsers, activeUsers, revenue, pending);
    }
}

// Repository Methods
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT SUM(o.total) FROM Order o WHERE o.status = 'COMPLETED'")
    BigDecimal getTotalRevenue();
    
    long countByStatus(OrderStatus status);
}
```

### ✅ Projection Best Practices

```java
// ✅ Use Interface Projections for Simple Cases
public interface UserEmailView {
    String getEmail();
}

// ✅ Use Class/Record Projections for Complex Logic
public record OrderSummary(
    Long orderId,
    String customerName,
    BigDecimal total,
    LocalDate orderDate
) {
    public boolean isRecent() {
        return orderDate.isAfter(LocalDate.now().minusDays(30));
    }
}

// ✅ Use Dynamic Projections for Flexible APIs
@GetMapping("/users")
public <T> List<T> getUsers(
        @RequestParam(required = false) String projection) {
    
    Class<T> type = switch (projection) {
        case "summary" -> (Class<T>) UserSummary.class;
        case "full" -> (Class<T>) User.class;
        default -> (Class<T>) UserNameView.class;
    };
    
    return userRepository.findBy(type);
}
```

## 💡 Why This Matters

Projections reduce database load by fetching only required columns. Interface-based projections use Spring-generated proxies - clean and simple. Class-based projections require constructor expressions in JPQL - more verbose. Record-based projections (Java 16+) combine brevity with immutability. Dynamic projections allow runtime type selection - flexible APIs. Open projections use SpEL for computed properties - adds minimal overhead. Nested projections fetch related entities selectively - avoid N+1 queries. Always prefer projections over loading full entities when you don't need all fields.

## 🎯 Key Takeaway

Use interface projections for read-only views with minimal fields. Use class/record projections for DTOs with business logic. Dynamic projections enable flexible APIs with runtime type selection. Projections dramatically improve performance - 5-10x faster for large entities. Combine with @EntityGraph to avoid N+1 queries on collections.

---

**Tags:** `#Java` `#JavaWisdom` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDataJPA` `#Projections` `#Performance` `#JPA`
