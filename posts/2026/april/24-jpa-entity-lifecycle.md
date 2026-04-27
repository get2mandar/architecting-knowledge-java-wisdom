# Post #24: JPA Entity Lifecycle - States You Should Know

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** April 26, 2026  
**Topic:** JPA, Hibernate, Entity Lifecycle, Persistence Context

---

## The Problem

Entities aren't just objects. They're state machines you need to understand.

## Code Example

### ❌ Common Misconception - It's Just CRUD

```java
// "I Just Save and Load Entities, Right?"
User user = new User();
userRepository.save(user);
// What State is This Entity In? Most Developers Don't Know
```

### ❌ Detached Entity Trap - Lost Changes

```java
@Service
public class UserService {
    
    @Transactional
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();
        // User Leaves Transaction as Detached!
    }
    
    public void updateUser(User user) {
        user.setName("Updated");  // Change Not Tracked!
        // No Transaction, No Persistence Context - Change Lost
    }
}
```

### ✅ Understanding the Four States

```java
/*
1. TRANSIENT (New)
   - Just Created with 'new'
   - No Primary Key
   - Not in Persistence Context
   - Not in Database

2. MANAGED (Persistent)
   - Tracked by Persistence Context
   - Changes Auto-Saved on Flush
   - Has Primary Key
   - Represents Database Row

3. DETACHED
   - Was Managed, Now Outside Context
   - Has Primary Key
   - Changes Not Tracked
   - Can Be Reattached

4. REMOVED
   - Marked for Deletion
   - Still in Context Until Flush
   - DELETE on Commit
*/
```

### ✅ Transient to Managed - Persist

```java
@Service
@Transactional
public class UserService {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    public void createUser() {
        User user = new User();  // TRANSIENT - No ID, Not Tracked
        user.setName("John");
        
        entityManager.persist(user);  // Now MANAGED
        
        // Automatic Flush Before Commit - INSERT Generated
        // User Now Has ID and Represents DB Row
    }
}
```

### ✅ Managed State - Dirty Checking Magic

```java
@Service
@Transactional
public class UserService {
    
    public void updateUser(Long id) {
        User user = entityManager.find(User.class, id);  // MANAGED
        
        user.setName("Updated");  // Change Tracked!
        user.setEmail("[email protected]");  // Also Tracked!
        
        // No Need to Call Save or Update!
        // Changes Auto-Flushed to DB Before Commit
    }
}
```

### ✅ Detached State - Transaction Boundary Issue

```java
@Service
public class UserService {
    
    @Transactional
    public User loadUser(Long id) {
        return entityManager.find(User.class, id);  // MANAGED Inside
        // Returns to Caller as DETACHED - Transaction Ends
    }
    
    public void updateDetachedUser() {
        User user = loadUser(1L);  // Now DETACHED
        
        user.setName("Changed");  // Change Not Tracked!
        
        // Wrong! Entity is Detached - No Auto-Save
        // Need to Merge or Save Explicitly
    }
}
```

### ✅ Reattaching Detached Entities - Merge

```java
@Service
@Transactional
public class UserService {
    
    public void updateDetached(User detachedUser) {
        // Option 1: Merge - Returns Managed Copy
        User managed = entityManager.merge(detachedUser);
        managed.setName("Updated");  // Now Tracked!
        
        // Option 2: Spring Data JPA Save
        userRepository.save(detachedUser);  // Handles Merge
    }
}
```

### ✅ Removed State - Entity Deletion

```java
@Service
@Transactional
public class UserService {
    
    public void deleteUser(Long id) {
        User user = entityManager.find(User.class, id);  // MANAGED
        
        entityManager.remove(user);  // Now REMOVED
        
        // Entity Still in Memory Until Flush
        // DELETE SQL Executed on Commit
    }
}
```

### ❌ Common Pitfall - LazyInitializationException

```java
@Service
public class UserService {
    
    @Transactional
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
    
    public void printOrders(Long userId) {
        User user = getUser(userId);  // DETACHED After Transaction
        
        user.getOrders().forEach(System.out::println);
        // LazyInitializationException! Session Closed
    }
}
```

### ✅ Solution - Fetch Within Transaction

```java
@Service
@Transactional
public class UserService {
    
    public void printOrders(Long userId) {
        User user = userRepository.findById(userId).orElseThrow();  // MANAGED
        
        user.getOrders().forEach(System.out::println);  // Works! Still in Transaction
        
        // Or Use @EntityGraph / JOIN FETCH
    }
}
```

### ✅ Checking Entity State - Debug Helper

```java
@Service
public class EntityStateChecker {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    public String checkState(Object entity) {
        if (!entityManager.contains(entity)) {
            return entity.getId() == null ? "TRANSIENT" : "DETACHED";
        }
        return "MANAGED";
    }
}
```

### ✅ Preventing Detachment - Keep Transaction Open

```java
// ❌ Wrong - Multiple Transactions
@Service
public class UserService {
    
    @Transactional
    public User load(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
    
    @Transactional
    public void update(User user) {  // User is Detached Here!
        userRepository.save(user);
    }
}

// ✅ Right - Single Transaction Scope
@Service
@Transactional
public class UserService {
    
    public void loadAndUpdate(Long id) {
        User user = userRepository.findById(id).orElseThrow();  // MANAGED
        user.setName("Updated");  // Still MANAGED - Auto-Saved
        // Single Transaction - No Detachment
    }
}
```

### ✅ Using @Transactional Correctly

```java
@Service
public class UserService {
    
    // Read-Only Transaction for Queries
    @Transactional(readOnly = true)
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
    
    // Write Transaction for Updates
    @Transactional
    public void updateUser(Long id, String name) {
        User user = userRepository.findById(id).orElseThrow();
        user.setName(name);  // Auto-Saved on Commit
    }
    
    // No Transaction - Entity Returns Detached
    public User getUserDetached(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

### ✅ Lifecycle Callbacks - Hook Into Transitions

```java
@Entity
public class User {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    @PrePersist
    protected void onCreate() {
        // Called Before INSERT
        System.out.println("About to Persist: " + name);
    }
    
    @PostPersist
    protected void afterCreate() {
        // Called After INSERT
        System.out.println("Persisted with ID: " + id);
    }
    
    @PreUpdate
    protected void onUpdate() {
        // Called Before UPDATE
        System.out.println("About to Update: " + name);
    }
    
    @PostUpdate
    protected void afterUpdate() {
        // Called After UPDATE
        System.out.println("Updated: " + name);
    }
    
    @PreRemove
    protected void onDelete() {
        // Called Before DELETE
        System.out.println("About to Remove: " + name);
    }
    
    @PostLoad
    protected void afterLoad() {
        // Called After Entity Loaded from DB
        System.out.println("Loaded: " + name);
    }
}
```

## 💡 Why This Matters

Understanding entity states prevents data loss and unexpected behavior. TRANSIENT entities have no ID and aren't tracked. MANAGED entities auto-save changes - no explicit save needed. DETACHED entities lose change tracking when transaction ends. Use merge() or save() to reattach detached entities. LazyInitializationException happens when accessing lazy collections outside transaction. Keep operations in single @Transactional scope to avoid detachment. EntityManager.contains() checks if entity is managed.

## 🎯 Key Takeaway

Entity lifecycle isn't optional knowledge - it's fundamental. Know when entities are managed vs detached. Keep transaction boundaries tight. Use @Transactional correctly. Dirty checking only works on managed entities. When in doubt, check the state.

---

**Tags:** `#Java` `#JavaWisdom` `#JPA` `#Hibernate` `#Persistence` `#EntityLifecycle`
