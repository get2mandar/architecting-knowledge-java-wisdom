# Post #3: @Transactional - The Silent Failure

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** February 11, 2026  
**Topic:** Spring @Transactional, Proxy Mechanism

---

## The Problem

Your Spring transaction doesn't work, and you never noticed.

## Code Example

### ❌ This Transaction won't Rollback
```java
@Service
public class UserService {
    
    @Transactional
    public void updateUser(User user) {
        userRepository.save(user);
        this.sendEmail(user);  // Calling own method - NO proxy!
    }
    
    @Transactional
    private void sendEmail(User user) {
        // Exception here won't rollback the save!
    }
}
```

### ✅ Architect's Pattern
```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserService self;  // Self-injection
    
    @Transactional
    public void updateUser(User user) {
        userRepository.save(user);
        self.sendEmail(user);  // Goes through proxy ✓
    }
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void sendEmail(User user) {
        // Now part of the transaction
    }
}
```

## 💡 Why This Matters

Spring's @Transactional works through proxies. Internal method calls bypass the proxy. Your transaction boundaries aren't where you think they are.

## 🎯 Key Takeaway

Self-invocation is transaction's silent killer. Use self-injection or move the logic to a separate service to ensure proxy interception.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringFramework` `#Transactional` `#CleanCode` `#JavaDevelopment`
