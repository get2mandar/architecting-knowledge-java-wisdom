# Post #39: Observability - Beyond Logs (Metrics, Traces, and Distributed Tracing)

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** June 17, 2026  
**Topic:** Observability, OpenTelemetry, Distributed Tracing, Micrometer

---

## The Problem

Logs tell you what happened. Metrics tell you something is wrong. Traces tell you where and why.

## Code Example

### ❌ Logging Only - Incomplete Picture

```java
@Service
public class OrderService {
    
    private final Logger logger = LoggerFactory.getLogger(OrderService.class);
    
    public Order processOrder(Order order) {
        logger.info("Processing order: {}", order.getId());
        
        try {
            inventoryService.reduceStock(order.getItems());
            paymentService.charge(order.getTotal());
            
            logger.info("Order processed successfully: {}", order.getId());
            return order;
        } catch (Exception e) {
            logger.error("Order processing failed", e);
            throw e;
        }
    }
}

// Problems:
// 1. Where Does Latency Come From? No Visibility
// 2. Logs Are Single-Service Only - No Cross-Service View
// 3. No Correlation Between Log Lines - Which Logs Belong to Same Request?
// 4. No Quantitative Data - Just String Messages
```

### ✅ The Three Pillars of Observability

```java
/*
Pillar 1: LOGS
  - Timestamped messages from application
  - Answers: What happened?
  - Example: "Order 123 created by user john@example.com"

Pillar 2: METRICS
  - Quantitative measurements (numbers)
  - Answers: What is the health? Trends?
  - Example: p99 latency 250ms, error rate 0.5%, requests/sec 1200

Pillar 3: TRACES
  - Request flow across services with timing
  - Answers: Where is latency? What went wrong?
  - Example: GET /order -> OrderService (10ms) -> PaymentService (2s) -> Database (1.9s)
*/
```

### ✅ Micrometer - Metrics First

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistry meterRegistry() {
        return new SimpleMeterRegistry();  // Or PrometheusMeterRegistry
    }
}

@Service
public class OrderService {
    
    private final MeterRegistry meterRegistry;
    private final Counter ordersPlaced;
    private final Timer orderProcessingTime;
    
    public OrderService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.ordersPlaced = Counter.builder("orders.placed")
            .description("Total orders placed")
            .tag("currency", "USD")
            .register(meterRegistry);
        
        this.orderProcessingTime = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .publishPercentiles(0.5, 0.95, 0.99)  // p50, p95, p99
            .register(meterRegistry);
    }
    
    public Order processOrder(Order order) {
        return orderProcessingTime.record(() -> {
            // Process Order
            Order saved = orderRepository.save(order);
            ordersPlaced.increment();
            return saved;
        });
    }
}

// GET /actuator/metrics/orders.processing.time
// Shows: Count, Mean, Max, and Percentiles (p50, p95, p99)
```

### ✅ OpenTelemetry - Distributed Tracing

```java
// pom.xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>

// application.properties
management.tracing.sampling.probability=1.0  // 100% Sampling in Dev
management.otlp.tracing.endpoint=http://localhost:4317  // Jaeger or Tempo
```

### ✅ Automatic Instrumentation - Spans

```java
@Service
public class OrderService {
    
    public Order processOrder(Order order) {
        // Micrometer Tracing Auto-Creates Span
        // Span Name: processOrder
        // Parent Trace ID: Propagated from Request Header
        
        inventoryService.reduceStock(order.getItems());  // Child Span
        paymentService.charge(order.getTotal());  // Child Span
        
        return order;
    }
}

// Generated Trace Structure
// [Trace ID: abc123]
//   ├─ Span: HTTP GET /orders
//   │  └─ Span: OrderService.processOrder (10ms)
//   │     ├─ Span: InventoryService.reduceStock (3ms)
//   │     ├─ Span: PaymentService.charge (2000ms) ← SLOW!
//   │     └─ Span: Save Order (5ms)
```

### ✅ Custom Spans - Deep Insights

```java
@Service
public class OrderService {
    
    private final Tracer tracer;
    
    public Order processOrder(Order order) {
        // Create Custom Span
        Span processingSpan = tracer.spanBuilder("order_processing")
            .addAttribute("orderId", order.getId())
            .addAttribute("userId", order.getUserId())
            .addAttribute("total", order.getTotal())
            .startSpan();
        
        try (Scope scope = processingSpan.makeCurrent()) {
            // Operations Inside This Scope Are Children of processingSpan
            
            Span inventorySpan = tracer.spanBuilder("reduce_stock")
                .addAttribute("itemCount", order.getItems().size())
                .startSpan();
            
            try (Scope inventoryScope = inventorySpan.makeCurrent()) {
                inventoryService.reduceStock(order.getItems());
            } finally {
                inventorySpan.end();
            }
            
            return order;
        } finally {
            processingSpan.end();
        }
    }
}
```

### ✅ Trace Context Propagation - Cross-Service Tracing

```java
// Service A: REST Controller
@RestController
public class OrderController {
    
    @PostMapping("/orders")
    public ResponseEntity<Order> create(@RequestBody OrderRequest request,
                                       @RequestHeader(HttpHeaders.TRACEPARENT) String traceParent) {
        // W3C Trace Context Header: traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
        // ├─ Version: 00
        // ├─ Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736
        // ├─ Span ID: 00f067aa0ba902b7
        // └─ Trace Flags: 01 (sampled)
        
        Order order = orderService.processOrder(request);
        return ResponseEntity.created(URI.create("/orders/" + order.getId()))
            .body(order);
    }
}

// Service B: Calls Service C
@Service
public class OrderService {
    
    private final WebClient webClient;
    
    public void processPayment(Order order) {
        // WebClient Automatically Propagates Trace Context!
        webClient.post()
            .uri("http://payment-service/charge")
            .bodyValue(order)
            .retrieve()
            .bodyToMono(PaymentResult.class)
            .block();
        
        // Trace: OrderService -> PaymentService automatically linked!
    }
}
```

### ✅ Correlation Between Logs and Traces

```java
@Configuration
public class LoggingConfig {
    
    @Bean
    public LoggingEventListener loggingEventListener() {
        return new LoggingEventListener();
    }
}

@Slf4j
@Service
public class OrderService {
    
    public Order processOrder(Order order) {
        // MDC Automatically Populated from Trace Context
        String traceId = MDC.get("traceId");    // From OpenTelemetry
        String spanId = MDC.get("spanId");
        
        logger.info("Processing order: {} [Trace: {} Span: {}]",
            order.getId(), traceId, spanId);
        
        // Log Now Searchable by Trace ID in Log Aggregation System
        // E.g., in ELK: GET /_search?q=traceId:abc123
    }
}

// application.properties
logging.pattern.console=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg [%X{traceId}/%X{spanId}]%n
```

### ✅ Sampling - Control Overhead

```java
// application.properties

// 100% Sampling (Development)
management.tracing.sampling.probability=1.0

// 10% Sampling (Production - Reduce Cost)
management.tracing.sampling.probability=0.1

// Adaptive Sampling (Conditional)
@Bean
public Sampler sampler() {
    return Sampler.create((traceId, spanContext, spanName, spanKind, 
                          attributes, parentContext) -> {
        // Sample All Errors
        if (spanName.contains("error")) {
            return SamplingDecision.RECORD_AND_SAMPLE;
        }
        
        // Sample Slow Requests (if duration > 1s)
        if (attributes.containsKey("duration") && 
            ((Long) attributes.get("duration")) > 1000) {
            return SamplingDecision.RECORD_AND_SAMPLE;
        }
        
        // Drop Others
        return SamplingDecision.DROP;
    });
}
```

### ✅ Backends - Where Traces Go

```java
// Jaeger (Popular, OpenTelemetry Native)
// - Real-time trace visualization
// - Service dependency map
// - Latency analysis

// Tempo (Grafana-native)
// - Simple deployment
// - Integrates with Grafana dashboards
// - Cost-effective

// DataDog, New Relic, Honeycomb
// - Commercial solutions
// - Advanced analytics
// - Integration with other observability tools

// application.properties
management.otlp.tracing.endpoint=http://localhost:4317  // OTLP Protocol
management.otlp.tracing.protocol=grpc
```

### ✅ Real-World Example - Microservices Gateway

```java
@RestController
@Slf4j
public class GatewayController {
    
    private final OrderService orderService;
    private final MeterRegistry meterRegistry;
    private final Tracer tracer;
    
    @PostMapping("/api/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        // Metrics: Count Requests
        Counter.builder("gateway.requests")
            .tag("endpoint", "/api/orders")
            .register(meterRegistry)
            .increment();
        
        // Tracing: Custom Span
        Span span = tracer.spanBuilder("gateway_create_order")
            .addAttribute("user.id", request.getUserId())
            .addAttribute("order.total", request.getTotal())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Logs: Include Trace Context (Auto-Populated via MDC)
            log.info("Creating order for user: {}", request.getUserId());
            
            Order order = orderService.processOrder(request);
            
            log.info("Order created: {} [Trace: {}]", 
                order.getId(), MDC.get("traceId"));
            
            return ResponseEntity.created(URI.create("/orders/" + order.getId()))
                .body(order);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}

// Dashboard in Jaeger:
// Shows Trace: GatewayController -> OrderService -> PaymentService
// Each Span Shows Duration, Attributes, Logs
// Error Highlighted with Exception Details
```

## 💡 Why This Matters

Observability combines logs, metrics, and traces for complete system visibility. Micrometer provides metrics abstraction - counters, timers, gauges. OpenTelemetry provides distributed tracing - spans, trace context propagation. Sampling reduces overhead in production - adaptive sampling only traces important requests. W3C Trace Context standard enables cross-service tracing automatically. Span attributes provide rich context - userId, orderId, durations. Correlation IDs (Trace ID) link logs, metrics, and traces together. Backend systems (Jaeger, Tempo, DataDog) store and visualize traces.

## 🎯 Key Takeaway

Add Micrometer for metrics (count, time, measure). Add OpenTelemetry for distributed tracing. Configure sampling strategically - 100% in dev, < 10% in production. Use trace IDs in logs via MDC - links logs with traces. Visualize traces in Jaeger or Tempo - find latency bottlenecks. Observability is not optional - it's essential for production systems.

---

**Tags:** `#Java` `#JavaWisdom` `#Observability` `#OpenTelemetry` `#Micrometer` `#DistributedTracing` `#Monitoring` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity`
