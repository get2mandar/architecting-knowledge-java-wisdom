# Post #34: Spring Boot Actuator - Production-Ready Endpoints

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** May 31, 2026  
**Topic:** Spring Boot Actuator, Monitoring, Health Checks, Metrics

---

## The Problem

Production apps need monitoring. Building it yourself takes weeks. Actuator gives it for free.

## Code Example

### ❌ Without Actuator - Manual Everything

```java
@RestController
public class MonitoringController {
    
    @GetMapping("/health")
    public Map<String, Object> health() {
        Map<String, Object> health = new HashMap<>();
        
        // Manual Database Check
        try {
            dataSource.getConnection().close();
            health.put("database", "UP");
        } catch (Exception e) {
            health.put("database", "DOWN");
        }
        
        // Manual Disk Space Check
        File root = new File("/");
        long freeSpace = root.getFreeSpace();
        health.put("diskSpace", freeSpace > 1_000_000_000 ? "UP" : "DOWN");
        
        // Dozens More Checks to Write!
        return health;
    }
}
```

### ✅ With Actuator - Built-In Monitoring

```java
// pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

// application.properties
management.endpoints.web.exposure.include=health,info,metrics

// That's It! Endpoints Available:
// GET /actuator/health
// GET /actuator/info
// GET /actuator/metrics
```

### ✅ Health Endpoint - Application Status

```java
// Default Health Endpoint
// GET /actuator/health
{
  "status": "UP"
}

// Detailed Health Info
// application.properties
management.endpoint.health.show-details=always

// Response
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500000000000,
        "free": 200000000000,
        "threshold": 10485760
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

### ✅ Custom Health Indicator - Domain-Specific Checks

```java
@Component
public class ExternalApiHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                "https://api.external.com/status",
                String.class
            );
            
            if (response.getStatusCode() == HttpStatus.OK) {
                return Health.up()
                    .withDetail("api", "External API is reachable")
                    .withDetail("responseTime", "120ms")
                    .build();
            } else {
                return Health.down()
                    .withDetail("error", "Non-200 response")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### ✅ Metrics Endpoint - Application Performance

```java
// GET /actuator/metrics
{
  "names": [
    "jvm.memory.used",
    "jvm.memory.max",
    "http.server.requests",
    "system.cpu.usage",
    "process.uptime",
    "db.pool.connections"
  ]
}

// GET /actuator/metrics/jvm.memory.used
{
  "name": "jvm.memory.used",
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 567123456
    }
  ],
  "availableTags": [
    {
      "tag": "area",
      "values": ["heap", "nonheap"]
    }
  ]
}

// GET /actuator/metrics/http.server.requests
{
  "name": "http.server.requests",
  "measurements": [
    {
      "statistic": "COUNT",
      "value": 12453
    },
    {
      "statistic": "TOTAL_TIME",
      "value": 245.3
    }
  ]
}
```

### ✅ Custom Metrics - Track Business Events

```java
@Service
public class OrderService {
    
    private final MeterRegistry meterRegistry;
    private final Counter orderCounter;
    private final Timer orderProcessingTimer;
    
    public OrderService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        
        this.orderCounter = Counter.builder("orders.placed")
            .description("Total orders placed")
            .tag("status", "success")
            .register(meterRegistry);
        
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .register(meterRegistry);
    }
    
    public void placeOrder(Order order) {
        orderProcessingTimer.record(() -> {
            // Process Order
            orderRepository.save(order);
            paymentService.charge(order);
            
            orderCounter.increment();
        });
    }
}

// GET /actuator/metrics/orders.placed
// GET /actuator/metrics/orders.processing.time
```

### ✅ Info Endpoint - Application Metadata

```java
// application.properties
info.app.name=Order Management System
info.app.version=2.5.0
info.app.description=Handles order processing
info.team.name=Platform Team

// GET /actuator/info
{
  "app": {
    "name": "Order Management System",
    "version": "2.5.0",
    "description": "Handles order processing"
  },
  "team": {
    "name": "Platform Team"
  }
}

// Add Build Info Automatically
// pom.xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <goals>
                        <goal>build-info</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>

// Now Info Includes Build Details
{
  "build": {
    "artifact": "order-service",
    "name": "order-service",
    "version": "2.5.0",
    "group": "com.example",
    "time": "2026-05-30T10:30:00.000Z"
  }
}
```

### ✅ Enabling and Exposing Endpoints

```java
// application.properties

// Expose Specific Endpoints
management.endpoints.web.exposure.include=health,info,metrics,prometheus

// Expose All Endpoints (Development Only!)
management.endpoints.web.exposure.include=*

// Exclude Specific Endpoints
management.endpoints.web.exposure.exclude=env,beans

// Enable Shutdown Endpoint (Disabled by Default)
management.endpoint.shutdown.enabled=true

// Change Base Path
management.endpoints.web.base-path=/admin

// Now: /admin/health instead of /actuator/health
```

### ✅ Kubernetes Health Probes

```java
// application.properties
management.endpoint.health.probes.enabled=true
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true

// Liveness Probe - Is App Running?
// GET /actuator/health/liveness
{
  "status": "UP"
}

// Readiness Probe - Can App Handle Traffic?
// GET /actuator/health/readiness
{
  "status": "UP"
}

// Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: order-service
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### ✅ Prometheus Integration - Metrics Export

```java
// pom.xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

// application.properties
management.endpoints.web.exposure.include=prometheus
management.metrics.export.prometheus.enabled=true

// GET /actuator/prometheus
# HELP jvm_memory_used_bytes Memory used
# TYPE jvm_memory_used_bytes gauge
jvm_memory_used_bytes{area="heap"} 5.67123456E8
jvm_memory_used_bytes{area="nonheap"} 1.23456789E8

# HELP http_server_requests_seconds HTTP requests
# TYPE http_server_requests_seconds summary
http_server_requests_seconds_count{method="GET",uri="/api/orders",status="200"} 1245
http_server_requests_seconds_sum{method="GET",uri="/api/orders",status="200"} 245.3
```

### ✅ Custom Endpoint - Business Specific Info

```java
@Component
@Endpoint(id = "sales")
public class SalesEndpoint {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @ReadOperation
    public Map<String, Object> sales() {
        long todaySales = orderRepository.countTodayOrders();
        BigDecimal todayRevenue = orderRepository.sumTodayRevenue();
        
        Map<String, Object> stats = new HashMap<>();
        stats.put("todayOrders", todaySales);
        stats.put("todayRevenue", todayRevenue);
        stats.put("timestamp", Instant.now());
        
        return stats;
    }
    
    @ReadOperation
    public Map<String, Object> salesByRegion(@Selector String region) {
        long orders = orderRepository.countByRegion(region);
        BigDecimal revenue = orderRepository.sumRevenueByRegion(region);
        
        Map<String, Object> stats = new HashMap<>();
        stats.put("region", region);
        stats.put("orders", orders);
        stats.put("revenue", revenue);
        
        return stats;
    }
}

// GET /actuator/sales
// GET /actuator/sales/us-west
```

### ✅ Securing Actuator Endpoints

```java
// pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

@Configuration
public class ActuatorSecurityConfig {
    
    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()
                .requestMatchers("/actuator/info").permitAll()
                .anyRequest().hasRole("ADMIN")
            )
            .httpBasic(withDefaults());
        
        return http.build();
    }
}

// application.properties
spring.security.user.name=admin
spring.security.user.password=secret
spring.security.user.roles=ADMIN
```

### ✅ Production Configuration - Separate Port

```java
// application-prod.properties

// Actuator on Different Port
management.server.port=8081
management.server.address=127.0.0.1  // Localhost Only!

// Minimal Exposure
management.endpoints.web.exposure.include=health,metrics,prometheus

// Show Health Details Only for Authorized
management.endpoint.health.show-details=when-authorized

// Disable Shutdown
management.endpoint.shutdown.enabled=false

// Now: localhost:8081/actuator/health (Internal Only)
//      localhost:8080/api/orders (Public)
```

### ✅ Environment Endpoint - Configuration Inspection

```java
// GET /actuator/env
{
  "activeProfiles": ["prod"],
  "propertySources": [
    {
      "name": "applicationConfig: [classpath:/application-prod.properties]",
      "properties": {
        "server.port": {
          "value": "8080"
        },
        "spring.datasource.url": {
          "value": "jdbc:mysql://prod-db:3306/orders"
        }
      }
    }
  ]
}

// Security Note: Mask Sensitive Properties!
// application.properties
management.endpoint.env.show-values=when-authorized
```

## 💡 Why This Matters

Actuator provides production-ready features out of the box - health checks, metrics, info. Health endpoint monitors application status - database, disk space, custom checks. Metrics endpoint exposes performance data - memory, CPU, HTTP requests, custom counters. Kubernetes probes use /actuator/health/liveness and /actuator/health/readiness. Prometheus scrapes /actuator/prometheus for metrics export. Custom endpoints expose business-specific monitoring data. Secure actuator endpoints in production - use separate port and authentication. MeterRegistry enables custom metrics - counters, timers, gauges.

## 🎯 Key Takeaway

Add spring-boot-starter-actuator for instant monitoring. Expose only needed endpoints in production. Use health probes for Kubernetes deployments. Integrate with Prometheus for metrics collection. Secure actuator endpoints - they expose sensitive data. Create custom health indicators for domain checks.

---

**Tags:** `#Java` `#JavaWisdom` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringBoot` `#Actuator` `#Monitoring` `#Production`
