# Post #36: @Transactional Propagation - When Transactions Nest

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** June 7, 2026  
**Topic:** Spring @Transactional, Propagation Levels, Transaction Management

---

## The Problem

Nested method calls with @Transactional can behave unexpectedly. Propagation controls how.

## Code Example

### ❌ Assuming All Calls Share One Transaction

```java
@Service
public class OrderService {
    
    @Autowired
    private AuditService auditService;
    
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        
        auditService.logOrder(order);  // What Transaction?
        
        if (fraudDetected(order)) {
            throw new FraudException();  // Rollback Both?
        }
    }
}

@Service
public class AuditService {
    
    @Transactional
    public void logOrder(Order order) {
        auditRepository.save(new AuditLog(order));
    }
}

// Question: If Fraud Detected, Does Audit Log Rollback Too?
// Answer: Depends on Propagation!
```

### ✅ Understanding Propagation Levels

```java
/*
REQUIRED (Default)
  - Join existing transaction or create new one
  - Most common, safest default

REQUIRES_NEW
  - Always create new transaction
  - Suspend existing transaction

NESTED
  - Create savepoint within existing transaction
  - Partial rollback possible

SUPPORTS
  - Join transaction if exists, run non-transactional otherwise
  
MANDATORY
  - Require existing transaction, throw exception if none
  
NOT_SUPPORTED
  - Always run non-transactionally, suspend existing
  
NEVER
  - Never run in transaction, throw exception if exists
*/
```

### ✅ REQUIRED - Default Behavior

```java
@Service
public class OrderService {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Transactional  // Transaction 1 Starts
    public void placeOrder(Order order) {
        orderRepository.save(order);
        
        inventoryService.reduceStock(order.getItems());  // Joins Transaction 1
        
        // If Exception Here - Everything Rolls Back!
    }
}

@Service
public class InventoryService {
    
    @Transactional  // Joins Existing Transaction
    public void reduceStock(List<Item> items) {
        items.forEach(item -> {
            Inventory inv = inventoryRepository.findById(item.getId()).orElseThrow();
            inv.reduce(item.getQuantity());
            inventoryRepository.save(inv);
        });
    }
}

// Both Methods Share Same Transaction
// Exception in Either Method Rolls Back Everything
```

### ✅ REQUIRES_NEW - Independent Transaction

```java
@Service
public class OrderService {
    
    @Autowired
    private AuditService auditService;
    
    @Transactional  // Transaction 1
    public void placeOrder(Order order) {
        orderRepository.save(order);
        
        auditService.logOrder(order);  // Transaction 2 (Independent)
        
        if (fraudDetected(order)) {
            throw new FraudException();  // Transaction 1 Rolls Back
        }
    }
}

@Service
public class AuditService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrder(Order order) {
        auditRepository.save(new AuditLog(order));
        // Transaction 2 Commits Immediately!
    }
}

// Audit Log Persists Even If Order Rolls Back
// Useful for Logging, Metrics, Non-Critical Operations
```

### ⚠️ REQUIRES_NEW - Connection Pool Warning

```java
@Service
public class BatchProcessor {
    
    @Transactional
    public void processBatch(List<Item> items) {
        for (Item item : items) {
            processItem(item);  // Opens New Connection Each Time!
        }
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processItem(Item item) {
        // Each Call Gets Separate Connection
    }
}

// Problem: Loop of 1000 Items = 1000 Database Connections!
// Risk: Connection Pool Exhaustion
// Solution: Batch Processing or Rethink Design
```

### ✅ NESTED - Savepoint Within Transaction

```java
@Service
public class OrderService {
    
    @Autowired
    private NotificationService notificationService;
    
    @Transactional  // Outer Transaction
    public void placeOrder(Order order) {
        orderRepository.save(order);
        
        try {
            notificationService.sendEmail(order);  // Nested Transaction
        } catch (Exception e) {
            // Nested Transaction Rolls Back to Savepoint
            // Outer Transaction Continues!
            logger.warn("Email failed, order still placed", e);
        }
    }
}

@Service
public class NotificationService {
    
    @Transactional(propagation = Propagation.NESTED)
    public void sendEmail(Order order) {
        emailRepository.save(new EmailLog(order));
        emailSender.send(order);  // May Fail
    }
}

// If Email Fails: Rollback to Savepoint
// Order Still Saved - Partial Rollback!
```

### ⚠️ NESTED - Database Support Required

```java
// NESTED Only Works With:
// - DataSourceTransactionManager (JDBC)
// - JpaTransactionManager (With JDBC Connections)

// Does NOT Work With:
// - Hibernate Without JDBC Connection
// - Some JTA Transaction Managers

@Service
public class ServiceWithNested {
    
    @Transactional(propagation = Propagation.NESTED)
    public void nestedMethod() {
        // May Throw Exception if Not Supported!
    }
}

// Check Transaction Manager Capabilities First
```

### ✅ SUPPORTS - Optional Transaction

```java
@Service
public class ReportService {
    
    @Transactional(propagation = Propagation.SUPPORTS)
    public Report generateReport(Long orderId) {
        // Joins Transaction if Called Within One
        // Runs Non-Transactionally if Called Standalone
        Order order = orderRepository.findById(orderId).orElseThrow();
        return new Report(order);
    }
}

// Called from @Transactional Method - Joins Transaction
orderService.processOrder(order);  // @Transactional
reportService.generateReport(order.getId());  // Joins

// Called Directly - No Transaction
reportService.generateReport(1L);  // No Transaction
```

### ✅ MANDATORY - Require Existing Transaction

```java
@Service
public class InventoryService {
    
    @Transactional(propagation = Propagation.MANDATORY)
    public void reduceStock(List<Item> items) {
        // Must Be Called Within Existing Transaction!
        items.forEach(item -> {
            Inventory inv = inventoryRepository.findById(item.getId()).orElseThrow();
            inv.reduce(item.getQuantity());
            inventoryRepository.save(inv);
        });
    }
}

// ✅ Called from @Transactional - Works
@Transactional
public void placeOrder(Order order) {
    inventoryService.reduceStock(order.getItems());  // OK
}

// ❌ Called Without Transaction - Exception!
inventoryService.reduceStock(items);  // IllegalTransactionStateException
```

### ✅ NOT_SUPPORTED - Always Non-Transactional

```java
@Service
public class MetricsService {
    
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void recordMetric(String eventName) {
        // Suspends Existing Transaction
        // Runs Without Transaction
        metricsRepository.save(new Metric(eventName));
    }
}

// Useful for Performance Metrics, Logging
// Where Transaction Overhead Not Needed
```

### ✅ NEVER - Prohibit Transactions

```java
@Service
public class CacheService {
    
    @Transactional(propagation = Propagation.NEVER)
    public void updateCache(String key, Object value) {
        // Must NOT Be Called Within Transaction
        cache.put(key, value);
    }
}

// ❌ Called from @Transactional - Exception!
@Transactional
public void method() {
    cacheService.updateCache("key", "value");  // IllegalTransactionStateException
}

// ✅ Called Without Transaction - Works
cacheService.updateCache("key", "value");  // OK
```

### ✅ Real-World Example - Order Processing

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    private final AuditService auditService;
    
    @Transactional  // Outer Transaction (REQUIRED)
    public void placeOrder(Order order) {
        // Save Order - Part of Main Transaction
        orderRepository.save(order);
        
        // Reduce Stock - Joins Main Transaction (REQUIRED)
        inventoryService.reduceStock(order.getItems());
        
        // Process Payment - Joins Main Transaction (REQUIRED)
        paymentService.charge(order.getTotal());
        
        // Log Audit - Independent Transaction (REQUIRES_NEW)
        // Persists Even If Order Fails
        auditService.logOrderAttempt(order);
        
        // Send Notification - Nested Transaction (NESTED)
        // Failure Doesn't Affect Order
        try {
            notificationService.sendConfirmation(order);
        } catch (Exception e) {
            logger.warn("Notification failed", e);
        }
    }
}

@Service
public class AuditService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logOrderAttempt(Order order) {
        auditRepository.save(new AuditLog(
            order.getId(),
            "ORDER_ATTEMPTED",
            Instant.now()
        ));
    }
}

@Service
public class NotificationService {
    
    @Transactional(propagation = Propagation.NESTED)
    public void sendConfirmation(Order order) {
        emailRepository.save(new EmailLog(order));
        emailSender.send(order);
    }
}
```

### ✅ Propagation Decision Matrix

```java
/*
Use REQUIRED When:
  - Default behavior is correct
  - Operations must succeed/fail together
  - Most common use case

Use REQUIRES_NEW When:
  - Operation must commit regardless of caller
  - Audit logs, metrics, independent records
  - Warning: Connection pool impact

Use NESTED When:
  - Partial rollback acceptable
  - Optional sub-operations
  - Requires savepoint support

Use SUPPORTS When:
  - Read-only operations
  - Optional transaction participation
  - Flexible execution context

Use MANDATORY When:
  - Method must run in transaction
  - Enforce transactional boundary
  - Fail-fast if misconfigured

Use NOT_SUPPORTED When:
  - Performance-critical operations
  - Transaction overhead unnecessary
  - Read-only caching operations

Use NEVER When:
  - Explicitly prohibit transactions
  - Enforce non-transactional execution
  - Rare use case
*/
```

### ⚠️ Self-Invocation Doesn't Work

```java
@Service
public class OrderService {
    
    @Transactional
    public void outerMethod() {
        this.innerMethod();  // Propagation IGNORED!
        // Spring Proxy Not Invoked
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() {
        // REQUIRES_NEW Never Applied!
    }
}

// ✅ Fix: Inject Self or Use Separate Service
@Service
public class OrderService {
    
    @Autowired
    private OrderService self;  // Inject Proxy
    
    @Transactional
    public void outerMethod() {
        self.innerMethod();  // Propagation Works!
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() {
        // REQUIRES_NEW Applied Correctly
    }
}
```

## 💡 Why This Matters

Propagation controls transaction boundaries in nested method calls. REQUIRED (default) joins existing or creates new - safe for most cases. REQUIRES_NEW creates independent transaction - use for audit logs, beware connection pool. NESTED creates savepoint for partial rollback - requires database support. SUPPORTS optional participation - useful for read operations. MANDATORY enforces transaction exists - fail-fast design. NOT_SUPPORTED suspends transaction - performance optimization. NEVER prohibits transaction - rare but explicit. Self-invocation bypasses proxy - inject self or use separate service.

## 🎯 Key Takeaway

Understand propagation to control transaction boundaries correctly. Use REQUIRED for typical operations requiring atomicity. Use REQUIRES_NEW sparingly for independent operations. Use NESTED for optional sub-operations with partial rollback. Always test nested transaction behavior in production scenarios.

---

**Tags:** `#Java` `#JavaWisdom` `#Spring` `#Transactional` `#DatabaseTransactions` `#Propagation` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava`
