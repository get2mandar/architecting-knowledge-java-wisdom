# Post #10: The @Async Annotation That Does Nothing

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 8, 2026  
**Topic:** Spring @Async, Asynchronous Programming

---

## The Problem

Your method isn't async. Spring is ignoring you. Here's why.

## Code Example

### ❌ Forgot to Enable Async - Annotation Does Nothing
```java
@SpringBootApplication
// Missing @EnableAsync!
public class Application { }

@Service
public class EmailService {
    @Async  // Does NOTHING without @EnableAsync
    public void sendEmail(String to, String message) {
        // Runs Synchronously!
    }
}
```

### ❌ Self-Invocation - Proxy Bypass
```java
@Service
public class OrderService {
    
    public void placeOrder(Order order) {
        this.sendNotification(order);  // NOT Async!
    }
    
    @Async
    public void sendNotification(Order order) {
        // Runs Synchronously - Same Class Call
    }
}
```

### ✅ Enable Async + Separate Service + Proper Return Type
```java
@SpringBootApplication
@EnableAsync
public class Application { }

@Service
@RequiredArgsConstructor
public class OrderService {
    private final NotificationService notificationService;
    
    public void placeOrder(Order order) {
        saveOrder(order);
        notificationService.sendNotification(order);  // Async Works!
    }
}

@Service
public class NotificationService {
    
    @Async
    public CompletableFuture<Void> sendNotification(Order order) {
        try {
            emailClient.send(order.getEmail(), "Order confirmed");
            return CompletableFuture.completedFuture(null);
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

## 💡 Why This Matters

Spring's @Async works through proxies, just like @Transactional. Without @EnableAsync, the annotation is ignored. Self-invocation bypasses the proxy. Return CompletableFuture for proper error handling - void methods swallow exceptions silently.

## 🎯 Key Takeaway

Three requirements for @Async: @EnableAsync in config, call from different bean, return CompletableFuture for errors. Miss any one, you're running synchronously.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringFramework` `#AsyncProgramming` `#SpringBoot` `#Concurrency`
