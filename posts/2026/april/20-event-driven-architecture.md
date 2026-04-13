# Post #20: Event-Driven Architecture in Spring

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** April 12, 2026  
**Topic:** Event-Driven Architecture, Spring Events, ApplicationEventPublisher

---

## The Problem

Tight coupling kills scalability. Events set you free.

## Code Example

### ❌ Tight Coupling - Service Calls Everything Directly

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final InventoryService inventoryService;
    private final EmailService emailService;
    private final NotificationService notificationService;
    private final AuditService auditService;
    
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        inventoryService.reduceStock(order.getItems());
        emailService.sendConfirmation(order.getCustomerEmail());
        notificationService.sendSMS(order.getCustomerPhone());
        auditService.logOrderPlaced(order);
        // Coupled to 4 Services! If Any Fails, Transaction Rolls Back
    }
}

// ❌ Synchronous and Blocking - All or Nothing
// Email Server Down? Order Fails. SMS Service Slow? Order Waits.
```

### ✅ Event-Driven - Decouple with Events

```java
public record OrderPlacedEvent(
    String orderId,
    List<OrderItem> items,
    String customerEmail,
    String customerPhone,
    Instant placedAt
) {}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        
        // Publish Event - Fire and Forget
        eventPublisher.publishEvent(new OrderPlacedEvent(
            order.getId(),
            order.getItems(),
            order.getCustomerEmail(),
            order.getCustomerPhone(),
            Instant.now()
        ));
        // Service Knows Nothing About Listeners!
    }
}
```

### ✅ Multiple Listeners - Independent Processing

```java
@Component
public class InventoryEventListener {
    
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Reduce Stock
        inventoryService.reduceStock(event.items());
    }
}

@Component
public class EmailEventListener {
    
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Send Email
        emailService.sendConfirmation(event.customerEmail());
    }
}

@Component
public class NotificationEventListener {
    
    @Async  // Process Asynchronously!
    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Send SMS
        notificationService.sendSMS(event.customerPhone());
    }
}
```

### ✅ Conditional Listeners - Only When Needed

```java
@Component
public class PremiumCustomerListener {
    
    @EventListener(condition = "#event.customerTier == 'PREMIUM'")
    public void handlePremiumOrder(OrderPlacedEvent event) {
        // Special Handling for Premium Customers
        loyaltyService.addBonusPoints(event.customerId());
    }
}
```

### ✅ Transaction-Bound Events - After Commit Only

```java
@Component
public class AuditEventListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Audit Only After Transaction Success
        auditService.logOrderPlaced(event.orderId());
    }
}
```

### ✅ Async Configuration - Enable Parallel Processing

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("event-");
        executor.initialize();
        return executor;
    }
}
```

## 💡 Why This Matters

ApplicationEventPublisher decouples producers from consumers. Publishers don't know who listens - zero coupling. Listeners can be added/removed without touching the publisher. Events are synchronous by default but @Async makes them non-blocking. @TransactionalEventListener ensures events fire only after commit - no phantom events on rollback. Use condition attribute for selective processing. Name events in past tense (OrderPlaced, not PlaceOrder) - they're facts, not commands.

## 🎯 Key Takeaway

Use events for cross-cutting concerns (audit, notifications, analytics). Keep event payloads immutable - use records. Publisher stays transactional, listeners process independently. Event-driven architecture trades immediate consistency for scalability and resilience.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringFramework` `#EventDriven` `#Microservices` `#Architecture`
