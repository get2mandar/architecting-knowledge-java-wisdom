# Post #9: Spring Data JPA - The N+1 Query Killer

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 4, 2026  
**Topic:** Spring Data JPA, N+1 Query Problem, Performance

---

## The Problem

One query became 1000. Here's why.

## Code Example

### ❌ The Silent Killer - Lazy Loading Nightmare
```java
@Entity
public class User {
    @Id
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "user")  // Lazy by Default
    private List<Order> orders;
}

@Service
public class UserService {
    public List<UserDTO> getAllUsers() {
        List<User> users = userRepository.findAll();  // 1 Query
        return users.stream()
            .map(user -> new UserDTO(
                user.getName(),
                user.getOrders().size()  // N Additional Queries!
            ))
            .collect(Collectors.toList());
    }
}

// SQL executed: 1000 Users = 1001 Queries in Total!
// SELECT * FROM users;                    -- Query 1
// SELECT * FROM orders WHERE user_id = 1; -- Query 2
// SELECT * FROM orders WHERE user_id = 2; -- Query 3
// ... and so on
```

### ✅ EntityGraph - Fetch Exactly What You Need
```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @EntityGraph(attributePaths = "orders")
    @Query("SELECT DISTINCT u FROM User u")
    List<User> findAllWithOrders();
}

@Service
public class UserService {
    public List<UserDTO> getAllUsers() {
        List<User> users = userRepository.findAllWithOrders();  // 1 Query!
        return users.stream()
            .map(user -> new UserDTO(
                user.getName(),
                user.getOrders().size()
            ))
            .collect(Collectors.toList());
    }
}

// SQL executed: Just 1 Query with JOIN
// SELECT DISTINCT u.*, o.*
// FROM users u LEFT JOIN orders o ON o.user_id = u.id;
```

## 💡 Why This Matters

Lazy loading seems convenient until you iterate in a loop. Each access to a lazy collection triggers a separate query. With 1000 users, that's 1001 database round trips. EntityGraph tells JPA to fetch associations upfront using JOINs, converting N+1 queries into a single query.

## 🎯 Key Takeaway

Default lazy loading is a production time bomb. Use EntityGraph to fetch associations explicitly. Test with realistic data volumes - 10 records hide problems that 10,000 expose.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringDataJPA` `#Performance` `#DatabaseOptimization` `#NPlus1`
