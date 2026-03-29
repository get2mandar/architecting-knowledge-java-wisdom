# Post #16: Logging - The Signal in the Noise

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** March 29, 2026  
**Topic:** Structured Logging, MDC, Observability

---

## The Problem

Your logs are useless in production. Here's how to fix them.

## Code Example

### ❌ Logging Noise - No Context, Wrong Level
```java
logger.info("Processing...");
logger.debug("Value is: " + expensiveComputation());  // Always Computed!
logger.error("Something failed");  // What Failed? Where? Why?
```

### ❌ String Concatenation - Performance Killer
```java
logger.info("User " + userId + " ordered " + items.size() + " items");
// String Built Even if INFO is Disabled
```

### ✅ Structured Logging with MDC - Contextual Information
```java
@Component
public class RequestFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);
        MDC.put("userId", extractUserId(request));
        
        try {
            logger.info("Request started");
            chain.doFilter(request, response);
            logger.info("Request completed");
        } finally {
            MDC.clear();  // Critical! Prevent Thread Pool Leaks
        }
    }
}

// Now Every Log in the Request Thread Includes requestId and userId
// Output: [requestId=abc-123] [userId=456] Order processing started
```

### ✅ Parameterized Logging - Lazy Evaluation
```java
logger.debug("Processing order: {}", order);  // toString() only if DEBUG Enabled
logger.info("User {} ordered {} items", userId, items.size());
```

### ✅ Meaningful Log Levels - Know What to Look For
```java
logger.error("Payment gateway timeout for order {}", orderId, exception);  // Production Alert
logger.warn("Inventory low for product {}: {} remaining", productId, count);  // Investigation Needed
logger.info("Order {} completed in {}ms", orderId, duration);  // Business Metrics
logger.debug("Request payload: {}", sanitize(payload));  // Development Only
```

### ✅ JSON Structured Logging - Machine-Readable
```java
// logback.xml with Logstash Encoder
logger.info("Order completed", 
    keyValue("orderId", order.getId()),
    keyValue("amount", order.getTotal()),
    keyValue("status", "SUCCESS")
);

// Output: {"timestamp":"2026-03-29T10:23:45","level":"INFO",
//         "requestId":"abc-123","userId":"456",
//         "orderId":"789","amount":99.99,"status":"SUCCESS",
//         "message":"Order completed"}
```

## 💡 Why This Matters

MDC (Mapped Diagnostic Context) attaches context to every log in a thread automatically. In production with 1000 concurrent requests, you can filter all logs for a single request by requestId. Parameterized logging defers string building until needed - massive performance win. JSON structured logs let you query by field in Elasticsearch/Splunk. Always clear MDC in finally blocks or you leak context between requests in thread pools.

## 🎯 Key Takeaway

Stop logging strings. Log structured data with MDC for automatic context, use parameterized messages for performance, choose levels deliberately. Production logs should be grep-able by correlation ID and parseable as JSON.

---

**Tags:** `#Java` `#JavaWisdom` `#Logging` `#Observability` `#StructuredLogging` `#MDC`
