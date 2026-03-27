# Post #14: The Repository Pattern Spring Hides

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 22, 2026  
**Topic:** Spring Data JPA, Repository Pattern, Custom Repositories

---

## The Problem

Spring Data magic is great. Until you need custom logic.

## Code Example

### ❌ Breaking the Abstraction - Service Does Too Much
```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final EntityManager entityManager;  // Leaked JPA!
    
    public List<UserDTO> findActiveUsersWithOrders() {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        // Complex Criteria Query Logic here...
        // This Belongs in the Repository Layer!
    }
}
```

### ❌ Repository Explosion - Too Many Interfaces
```java
public interface UserRepository extends JpaRepository<User, Long> { }
public interface UserSearchRepository extends JpaRepository<User, Long> { }
public interface UserReportRepository extends JpaRepository<User, Long> { }
// Three Repositories for the Same Entity?
```

### ✅ Custom Repository Fragment - Extend without Breaking
```java
public interface UserRepositoryCustom {
    List<User> findActiveUsersWithComplexCriteria(SearchParams params);
}

public class UserRepositoryImpl implements UserRepositoryCustom {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public List<User> findActiveUsersWithComplexCriteria(SearchParams params) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<User> query = cb.createQuery(User.class);
        Root<User> user = query.from(User.class);
        
        List<Predicate> predicates = new ArrayList<>();
        if (params.getName() != null) {
            predicates.add(cb.like(user.get("name"), "%" + params.getName() + "%"));
        }
        if (params.isActiveOnly()) {
            predicates.add(cb.equal(user.get("active"), true));
        }
        
        query.where(predicates.toArray(new Predicate[0]));
        return entityManager.createQuery(query).getResultList();
    }
}

// Repository Extends Both JpaRepository AND Custom Fragment
public interface UserRepository extends 
    JpaRepository<User, Long>, 
    UserRepositoryCustom {
    
    // Spring Data Query Methods
    List<User> findByActiveTrue();
    
    // Custom Method from UserRepositoryCustom
    List<User> findActiveUsersWithComplexCriteria(SearchParams params);
}
```

### ✅ Service Stays Clean - No JPA Leakage
```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    
    public List<UserDTO> searchUsers(SearchParams params) {
        return userRepository.findActiveUsersWithComplexCriteria(params)
            .stream()
            .map(this::toDTO)
            .collect(Collectors.toList());
    }
}
```

## 💡 Why This Matters

Spring Data's query derivation and @Query are powerful, but not sufficient for all scenarios. Custom repository fragments let you add complex logic using EntityManager or JDBC while maintaining the repository abstraction. Spring automatically detects implementations with the "Impl" suffix and merges them with your repository interface. Your service layer stays clean, unaware of JPA internals.

## 🎯 Key Takeaway

Don't leak JPA into your service layer. Create custom repository fragments for complex queries. Follow the pattern: define custom interface, implement with "Impl" suffix, extend from your main repository. Spring handles the rest.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringDataJPA` `#RepositoryPattern` `#CleanArchitecture` `#BestPractices`
