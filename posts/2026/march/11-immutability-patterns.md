# Post #11: Immutability Patterns - Defense by Design

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 11, 2026  
**Topic:** Immutability, Defensive Copying, Thread Safety

---

## The Problem

Mutable state is a bug waiting to happen. Here's how to defend against it.

## Code Example

### ❌ Looks Immutable, but Leaks Mutable State
```java
public final class UserProfile {
    private final String name;
    private final List<String> roles;
    
    public UserProfile(String name, List<String> roles) {
        this.name = name;
        this.roles = roles;  // Direct Reference!
    }
    
    public List<String> getRoles() {
        return roles;  // Leaking Internal State!
    }
}

// Usage - the "immutable" Object Just Got Mutated
UserProfile profile = new UserProfile("Alice", new ArrayList<>(List.of("USER")));
profile.getRoles().add("ADMIN");  // Oops! Modified After Construction
```

### ❌ Defensive Copy Only On Get - Still Broken
```java
public final class UserProfile {
    private final List<String> roles;
    
    public UserProfile(List<String> roles) {
        this.roles = roles;  // Still Direct Reference!
    }
    
    public List<String> getRoles() {
        return new ArrayList<>(roles);  // Copy On Get
    }
}

// Usage - Still Mutable!
List<String> rolesList = new ArrayList<>(List.of("USER"));
UserProfile profile = new UserProfile(rolesList);
rolesList.add("ADMIN");  // External Modification!
```

### ✅ Defensive Copy in Constructor + Unmodifiable View
```java
public final class UserProfile {
    private final String name;
    private final List<String> roles;
    
    public UserProfile(String name, List<String> roles) {
        this.name = name;
        this.roles = new ArrayList<>(roles);  // Defensive Copy
    }
    
    public List<String> getRoles() {
        return Collections.unmodifiableList(roles);  // Immutable View
    }
}
```

### ✅ Modern Approach - Use Immutable Collections
```java
public final class UserProfile {
    private final String name;
    private final List<String> roles;
    
    public UserProfile(String name, List<String> roles) {
        this.name = name;
        this.roles = List.copyOf(roles);  // Immutable Copy (Java 10+)
    }
    
    public List<String> getRoles() {
        return roles;  // Already Immutable, Safe to Return
    }
}
```

## 💡 Why This Matters

Final fields don't guarantee immutability when they hold references to mutable objects. The reference is final, but the object it points to can still change. Defensive copying in the constructor protects against external modification after construction. Unmodifiable wrappers or immutable collections prevent modification through getters. True immutability requires both.

## 🎯 Key Takeaway

Immutability is about object state, not just final fields. Copy mutable parameters in constructors, return unmodifiable views from getters. Better yet, use List.copyOf() and immutable collections by default.

---

**Tags:** `#Java` `#JavaWisdom` `#Immutability` `#ThreadSafety` `#DesignPatterns` `#CleanCode`
