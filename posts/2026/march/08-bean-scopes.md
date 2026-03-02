# Post #8: Spring Bean Scopes - The Lifecycle You Ignore

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 1, 2026  
**Topic:** Spring Bean Scopes, Dependency Injection

---

## The Problem

Why your prototype bean acts like a singleton. The lifecycle trap explained.

## Code Example

### ❌ Prototype Injection into Singleton - Doesn't Work as Expected

```java
@Service  // Singleton by Default
public class OrderProcessor {
    
    @Autowired
    private OrderContext orderContext;  // Prototype Bean
    
    public void processOrder(Order order) {
        orderContext.setOrder(order);  // SAME INSTANCE every time!
        // All orders share the same context
    }
}

@Component
@Scope("prototype")
public class OrderContext {
    private Order order;
}
```

### ✅ ObjectProvider - Clean and Modern

```java
@Service
@RequiredArgsConstructor
public class OrderProcessor {
    
    private final ObjectProvider<OrderContext> contextProvider;
    
    public void processOrder(Order order) {
        OrderContext context = contextProvider.getObject();  // Fresh Instance!
        context.setOrder(order);
        context.process();
    }
}
```

### ✅ Request Scope for Web Apps
```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, 
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String requestId;
    // One instance per HTTP request
}
```

## 💡 Why This Matters

Spring creates prototype beans at injection time, not at usage time. When a singleton holds a reference to a prototype, that reference is set ONCE during initialization. You get the same "prototype" instance forever. ObjectProvider solves this by deferring bean creation until you actually call getObject().

## 🎯 Key Takeaway

Prototype beans in singleton containers are a trap. Use ObjectProvider for programmatic control or scoped proxies for declarative injection. Understand your bean lifecycle before choosing a scope.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringFramework` `#BeanScopes` `#DependencyInjection` `#SpringBoot`
