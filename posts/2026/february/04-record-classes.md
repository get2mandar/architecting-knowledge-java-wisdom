# Post #4: Record Classes - More Than Boilerplate Killers

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** February 15, 2026  
**Topic:** Java Records, Domain Modeling, Validation

---

## The Problem

Records aren't just DTOs with less typing. They're design tools.

## Code Example

### ❌ Traditional Thinking - Just Replacing DTOs
```java
public record UserDTO(String name, String email) {}
```

### ✅ Architect's Thinking - Domain Modeling
```java
public record EmailAddress(String value) {
    public EmailAddress {
        if (!value.matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
            throw new IllegalArgumentException("Invalid email");
        }
    }
}

public record UserRegistration(
    String name,
    EmailAddress email,  // Type-safe, always valid
    LocalDate registeredOn
) {
    // Compact constructor for invariants
    public UserRegistration {
        if (registeredOn.isAfter(LocalDate.now())) {
            throw new IllegalArgumentException("Cannot register in future");
        }
    }
    
    // Derived data
    public boolean isRecentlyRegistered() {
        return ChronoUnit.DAYS.between(registeredOn, LocalDate.now()) <= 7;
    }
}
```

## 💡 Why This Matters

Records with compact constructors enforce invariants at construction time. Invalid states become impossible, not just unlikely. Your domain rules live in your types.

## 🎯 Key Takeaway

Records + validation = self-defending data. Make illegal states unrepresentable in your domain model.

---

**Tags:** `#Java` `#JavaWisdom` `#Records` `#ModernJava` `#CleanCode` `#DomainDrivenDesign`
