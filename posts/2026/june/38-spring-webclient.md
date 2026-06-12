# Post #38: Spring WebClient - Modern HTTP Client

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** June 14, 2026  
**Topic:** Spring WebClient, Reactive HTTP, Non-Blocking, Resilience

---

## The Problem

RestTemplate is deprecated. WebClient is the future. But it's async. Learn the transition.

## Code Example

### ❌ RestTemplate - Blocking and Deprecated

```java
// RestTemplate - Synchronous, Blocks Thread
@Service
public class UserService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public User fetchUser(Long id) {
        // Thread Blocks Until Response Arrives
        ResponseEntity<User> response = restTemplate.getForEntity(
            "http://api.example.com/users/{id}",
            User.class,
            id
        );
        return response.getBody();
    }
}

// Problems:
// 1. One Thread Per Request = Limited Concurrency
// 2. Memory Overhead = 1MB Per Thread
// 3. No Timeout Handling Built-In
// 4. Deprecated Since Spring 5!
```

### ✅ WebClient - Non-Blocking and Reactive

```java
// WebClient - Asynchronous, Non-Blocking
@Service
public class UserService {
    
    @Autowired
    private WebClient webClient;
    
    public Mono<User> fetchUser(Long id) {
        // Non-Blocking, Returns Mono (Promise)
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class);
    }
}

// Benefits:
// 1. One Thread Serves Multiple Requests
// 2. Lower Memory Footprint
// 3. Built-In Timeout Support
// 4. Modern Reactive API
```

### ✅ WebClient Configuration - Setup

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient() {
        // Basic WebClient
        return WebClient.create("http://api.example.com");
    }
    
    @Bean
    public WebClient advancedWebClient() {
        HttpClient httpClient = HttpClient.create()
            .responseTimeout(Duration.ofSeconds(5))  // Response Timeout
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)  // Connect Timeout
            .option(ChannelOption.SO_KEEPALIVE, true);  // Keep-Alive
        
        return WebClient.builder()
            .baseUrl("http://api.example.com")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.USER_AGENT, "MyApp/1.0")
            .build();
    }
}
```

### ✅ GET Request - Simple Retrieval

```java
@Service
public class ProductService {
    
    @Autowired
    private WebClient webClient;
    
    // Single Product
    public Mono<Product> getProduct(Long id) {
        return webClient.get()
            .uri("/products/{id}", id)
            .retrieve()
            .bodyToMono(Product.class)
            .timeout(Duration.ofSeconds(5));
    }
    
    // List of Products
    public Flux<Product> getAllProducts() {
        return webClient.get()
            .uri("/products")
            .retrieve()
            .bodyToFlux(Product.class)
            .timeout(Duration.ofSeconds(10));
    }
}

// Usage
productService.getProduct(1L)
    .subscribe(product -> System.out.println(product));

productService.getAllProducts()
    .subscribe(product -> System.out.println(product));
```

### ✅ POST Request - Creating Resources

```java
@Service
public class OrderService {
    
    @Autowired
    private WebClient webClient;
    
    public Mono<Order> createOrder(OrderRequest request) {
        return webClient.post()
            .uri("/orders")
            .contentType(MediaType.APPLICATION_JSON)
            .body(Mono.just(request), OrderRequest.class)  // Send Body
            .retrieve()
            .bodyToMono(Order.class)
            .timeout(Duration.ofSeconds(5));
    }
}

// Usage
orderService.createOrder(new OrderRequest("item123", 2))
    .subscribe(order -> System.out.println("Order: " + order.getId()));
```

### ✅ Error Handling - Graceful Failures

```java
@Service
public class UserService {
    
    @Autowired
    private WebClient webClient;
    
    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .onStatus(
                status -> status.is4xxClientError(),
                response -> response.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(
                        new UserNotFoundException("User not found: " + body)
                    ))
            )
            .onStatus(
                status -> status.is5xxServerError(),
                response -> response.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(
                        new ServerException("Server error: " + body)
                    ))
            )
            .bodyToMono(User.class)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(WebClientResponseException.class, ex -> {
                logger.error("WebClient error: " + ex.getResponseBodyAsString());
                return Mono.error(ex);
            })
            .doOnError(e -> logger.error("Request failed", e));
    }
}
```

### ✅ Retry Logic - Resilience

```java
@Service
public class OrderService {
    
    @Autowired
    private WebClient webClient;
    
    public Mono<Order> processOrder(OrderRequest request) {
        return webClient.post()
            .uri("/orders")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(Order.class)
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1))  // Max 3 Retries
                .filter(throwable -> isRetryable(throwable))
                .doBeforeRetry(signal -> logger.info("Retrying...")))
            .timeout(Duration.ofSeconds(5));
    }
    
    private boolean isRetryable(Throwable throwable) {
        if (throwable instanceof WebClientResponseException ex) {
            // Retry on 5xx, Not 4xx
            return ex.getStatusCode().is5xxServerError();
        }
        return throwable instanceof TimeoutException
            || throwable instanceof IOException;
    }
}
```

### ✅ Headers and Query Parameters

```java
@Service
public class ApiClient {
    
    @Autowired
    private WebClient webClient;
    
    public Mono<Response> search(String query, int page) {
        return webClient.get()
            .uri(uriBuilder -> uriBuilder
                .path("/search")
                .queryParam("q", query)
                .queryParam("page", page)
                .build()
            )
            .header("Authorization", "Bearer " + getToken())
            .header("X-Custom-Header", "value")
            .retrieve()
            .bodyToMono(Response.class);
    }
}
```

### ✅ Request/Response Interceptors

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient webClient() {
        return WebClient.builder()
            .baseUrl("http://api.example.com")
            .filter((request, next) -> {
                // Log Request
                logger.info("Request: {} {}", 
                    request.getHttpMethod(), 
                    request.getUrl());
                
                // Add Headers
                request.getHeaders().add("X-Trace-Id", generateTraceId());
                
                // Continue
                return next.exchange(request)
                    .doOnNext(response -> {
                        // Log Response
                        logger.info("Response: {}", response.getStatusCode());
                    });
            })
            .build();
    }
}
```

### ✅ Combining Multiple Requests - Orchestration

```java
@Service
public class OrderAssemblyService {
    
    @Autowired
    private WebClient webClient;
    
    public Mono<OrderDetails> getOrderDetails(Long orderId) {
        return webClient.get()
            .uri("/orders/{id}", orderId)
            .retrieve()
            .bodyToMono(Order.class)
            .flatMap(order -> 
                Mono.zip(
                    getUser(order.getUserId()),
                    getShippingInfo(order.getShippingId()),
                    getPaymentInfo(order.getPaymentId())
                ).map(tuple -> new OrderDetails(
                    order,
                    tuple.getT1(),  // User
                    tuple.getT2(),  // Shipping
                    tuple.getT3()   // Payment
                ))
            );
    }
    
    private Mono<User> getUser(Long userId) {
        return webClient.get()
            .uri("/users/{id}", userId)
            .retrieve()
            .bodyToMono(User.class);
    }
    
    private Mono<ShippingInfo> getShippingInfo(Long shippingId) {
        return webClient.get()
            .uri("/shipping/{id}", shippingId)
            .retrieve()
            .bodyToMono(ShippingInfo.class);
    }
    
    private Mono<PaymentInfo> getPaymentInfo(Long paymentId) {
        return webClient.get()
            .uri("/payments/{id}", paymentId)
            .retrieve()
            .bodyToMono(PaymentInfo.class);
    }
}

// Usage
orderAssemblyService.getOrderDetails(1L)
    .subscribe(details -> System.out.println(details));
```

### ✅ Streaming Responses - Server-Sent Events

```java
@RestController
public class NotificationController {
    
    @Autowired
    private WebClient webClient;
    
    @GetMapping("/notifications", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamNotifications() {
        return webClient.get()
            .uri("http://notification-service/stream")
            .retrieve()
            .bodyToFlux(String.class)
            .timeout(Duration.ofMinutes(5));  // Long-Lived Connection
    }
}
```

### ✅ Request Timeout Configuration - Production Ready

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    public WebClient productionWebClient() {
        HttpClient httpClient = HttpClient.create()
            // Connection Timeout
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
            // Response Timeout
            .responseTimeout(Duration.ofSeconds(5))
            // Keep-Alive
            .option(ChannelOption.SO_KEEPALIVE, true)
            // Connection Pool
            .connectionProvider(
                ConnectionProvider.builder("custom")
                    .maxConnections(500)
                    .maxIdleTime(Duration.ofMinutes(5))
                    .build()
            );
        
        return WebClient.builder()
            .baseUrl("http://api.example.com")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

### ✅ Real-World Example - Microservice Gateway

```java
@RestController
@RequiredArgsConstructor
public class GatewayController {
    
    private final WebClient webClient;
    
    @GetMapping("/api/user-profile/{userId}")
    public Mono<UserProfile> getUserProfile(@PathVariable Long userId) {
        return Mono.zip(
            // Call User Service
            webClient.get()
                .uri("http://user-service/users/{id}", userId)
                .retrieve()
                .bodyToMono(User.class)
                .timeout(Duration.ofSeconds(3)),
            
            // Call Order Service
            webClient.get()
                .uri("http://order-service/users/{id}/orders", userId)
                .retrieve()
                .bodyToFlux(Order.class)
                .collectList()
                .timeout(Duration.ofSeconds(3)),
            
            // Call Preference Service
            webClient.get()
                .uri("http://preference-service/users/{id}/prefs", userId)
                .retrieve()
                .bodyToMono(Preferences.class)
                .timeout(Duration.ofSeconds(3))
        ).map(tuple -> new UserProfile(
            tuple.getT1(),
            tuple.getT2(),
            tuple.getT3()
        )).timeout(Duration.ofSeconds(10))
         .onErrorResume(WebClientResponseException.class, ex -> {
             logger.error("Gateway error", ex);
             return Mono.error(new GatewayException());
         });
    }
}
```

## 💡 Why This Matters

WebClient is Spring's modern HTTP client for synchronous and asynchronous requests. Non-blocking architecture handles more requests with fewer threads - critical for microservices. Mono<T> represents single async value, Flux<T> represents stream. Built-in timeout, retry, and error handling - more robust than RestTemplate. Reactive composition with flatMap, zip, and operators - powerful async orchestration. Connection pooling and keep-alive - better resource management. Server-sent events and streaming - handles large data efficiently.

## 🎯 Key Takeaway

Migrate from RestTemplate to WebClient for new projects. Use Mono for single values, Flux for streams. Always set timeouts - prevent hanging requests. Use retryWhen for transient failures with backoff. Compose multiple requests with zip() or flatMap(). Handle errors explicitly with onStatus() and onErrorResume().

---

**Tags:** `#Java` `#JavaWisdom` `#SpringBoot` `#WebClient` `#Reactive` `#AsyncHttp` `#Microservices` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide`
