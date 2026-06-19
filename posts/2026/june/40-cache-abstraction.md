# Post #40: Cache Abstraction - @Cacheable Done Right

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** June 21, 2026  
**Topic:** Spring Cache Abstraction, @Cacheable, @CacheEvict, Cache Managers

---

## The Problem

Manual caching is tedious and error-prone. Spring Cache Abstraction eliminates boilerplate.

## Code Example

### ❌ Manual Cache Management - Messy

```java
@Service
public class UserService {
    
    private final Map<Long, User> userCache = new ConcurrentHashMap<>();
    private final UserRepository userRepository;
    
    public User getUserById(Long id) {
        // Check Cache
        if (userCache.containsKey(id)) {
            return userCache.get(id);
        }
        
        // Load from Database
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        // Store in Cache
        userCache.put(id, user);
        
        return user;
    }
    
    public User updateUser(Long id, User updatedUser) {
        User user = userRepository.save(updatedUser);
        
        // Invalidate Cache
        userCache.remove(id);
        
        return user;
    }
}

// Problems:
// 1. Business Logic Mixed with Caching Logic
// 2. Cache Not Distributed - Doesn't Work in Clustered Environment
// 3. No TTL Support - Cache Grows Indefinitely
// 4. No Fallback to Different Cache Providers
```

### ✅ Spring Cache Abstraction - Declarative

```java
@Configuration
@EnableCaching
public class CacheConfig {
    // Enables @Cacheable, @CacheEvict, @CachePut
}

@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Cacheable(value = "users", key = "#id")
    public User getUserById(Long id) {
        // Automatic Cache Check BEFORE Method Execution
        // If Cache Hit: Return Cached Value
        // If Cache Miss: Execute Method, Cache Result
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        // Always Execute Method
        // Cache Result (for read later)
        return userRepository.save(user);
    }
    
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        // Remove from Cache
        userRepository.deleteById(id);
    }
}
```

### ✅ @Cacheable - Basic Caching

```java
@Service
public class ProductService {
    
    // Simple Caching
    @Cacheable(value = "products")
    public Product getProduct(Long id) {
        // Cache Key Auto-Generated from Method Arguments
        // Default Key: SimpleKeyGenerator (args)
        return productRepository.findById(id).orElseThrow();
    }
    
    // Custom Cache Key
    @Cacheable(value = "productsByCategory", key = "#category + ':' + #page")
    public List<Product> getProductsByCategory(String category, int page) {
        return productRepository.findByCategory(category, PageRequest.of(page, 20))
            .getContent();
    }
    
    // Conditional Caching
    @Cacheable(
        value = "productDetails",
        key = "#id",
        condition = "#id > 0",         // Only Cache if ID > 0
        unless = "#result == null"     // Don't Cache Null Results
    )
    public Product getProductDetail(Long id) {
        return productRepository.findById(id).orElse(null);
    }
}
```

### ✅ @CachePut - Always Execute and Update Cache

```java
@Service
public class OrderService {
    
    @CachePut(value = "orders", key = "#order.id")
    public Order saveOrder(Order order) {
        // Always Executes Method
        // Always Updates Cache
        // Useful for Updates
        return orderRepository.save(order);
    }
    
    // Problem: Cache Doesn't Help - Method Always Runs
    // Solution: Use @Cacheable for Reads, @CachePut for Writes
}

// Typical Pattern
@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#id")
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();  // Cache Hit on Reads
    }
    
    @CachePut(value = "users", key = "#user.id")
    public User updateUser(User user) {
        return userRepository.save(user);  // Update Database and Cache
    }
}
```

### ✅ @CacheEvict - Remove from Cache

```java
@Service
public class UserService {
    
    // Evict Single Entry
    @CacheEvict(value = "users", key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
    
    // Evict All Entries
    @CacheEvict(value = "users", allEntries = true)
    public void clearAllUsers() {
        // Clears Entire Cache
    }
    
    // Multiple Cache Operations
    @Caching(
        evict = {
            @CacheEvict(value = "users", key = "#id"),
            @CacheEvict(value = "userPermissions", key = "#id"),
            @CacheEvict(value = "userRoles", allEntries = true)
        }
    )
    public void deleteUserAndRelated(Long id) {
        userRepository.deleteById(id);
    }
}
```

### ✅ Cache Managers - Switching Providers

```java
// Default: In-Memory ConcurrentHashMap
@Configuration
@EnableCaching
public class CacheConfigDefault {
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("users", "products", "orders");
    }
}

// Caffeine: Fast In-Memory Cache (Local)
@Configuration
@EnableCaching
public class CacheConfigCaffeine {
    
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("users", "products");
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(10, TimeUnit.MINUTES)  // TTL: 10 Minutes
            .maximumSize(1000));  // Max 1000 Entries
        return cacheManager;
    }
}

// Redis: Distributed Cache (Across Instances)
// pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

// application.properties
spring.cache.type=redis
spring.redis.host=localhost
spring.redis.port=6379

// Automatically Uses RedisCacheManager!

@Service
public class UserService {
    
    @Cacheable(value = "users", key = "#id")  // Works with ANY Cache Manager!
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
}
```

### ✅ Cache Key Generation

```java
@Service
public class ReportService {
    
    // Default KeyGenerator: Uses All Parameters
    @Cacheable(value = "reports")
    public Report generateReport(Long userId, String type, int year) {
        // Key: [userId, type, year]
        return generateFromData(userId, type, year);
    }
    
    // Custom KeyGenerator
    @Bean
    public KeyGenerator customKeyGenerator() {
        return (target, method, params) -> {
            // Custom Logic
            return params[0].toString() + ":" + params[1].toString();
        };
    }
    
    @Cacheable(value = "reports", keyGenerator = "customKeyGenerator")
    public Report generateReport(Long userId, String type) {
        // Uses Custom KeyGenerator
        return generateFromData(userId, type);
    }
    
    // SpEL Expressions
    @Cacheable(
        value = "userReports",
        key = "T(java.lang.String).format('%s_%s_%d', #userId, #type, #year)"
    )
    public Report generateReportWithSpEL(Long userId, String type, int year) {
        return generateFromData(userId, type, year);
    }
}
```

### ✅ TTL and Eviction Policies

```java
@Configuration
@EnableCaching
public class CacheConfigWithTTL {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        Map<String, RedisCacheConfiguration> cacheConfigs = new HashMap<>();
        
        // Cache 1: 5 Minutes TTL
        cacheConfigs.put("users", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5)));
        
        // Cache 2: 1 Hour TTL
        cacheConfigs.put("reports", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1)));
        
        // Cache 3: No TTL (Explicit Eviction Only)
        cacheConfigs.put("config", RedisCacheConfiguration.defaultCacheConfig()
            .disableCachingNullValues());
        
        return RedisCacheManager.create(
            RedisCacheWriter.lockingRedisCacheWriter(connectionFactory),
            RedisCacheConfiguration.defaultCacheConfig(),
            cacheConfigs
        );
    }
}
```

### ✅ Conditional Caching - Smart Decisions

```java
@Service
public class UserService {
    
    @Cacheable(
        value = "users",
        key = "#id",
        condition = "#id > 0",                    // Only Cache if ID Valid
        unless = "#result == null"                // Don't Cache Null
    )
    public User getUser(Long id) {
        return userRepository.findById(id).orElse(null);
    }
    
    @Cacheable(
        value = "userProfile",
        key = "#userId",
        condition = "#cacheable == true"          // Cache Only if Parameter Allows
    )
    public UserProfile getUserProfile(Long userId, boolean cacheable) {
        return buildProfile(userId);
    }
    
    // Cache Only Large Results
    @Cacheable(
        value = "bulkReports",
        key = "#reportId",
        unless = "#result.size() < 100"           // Don't Cache Small Results
    )
    public List<ReportData> getBulkReportData(Long reportId) {
        return expensiveQuery(reportId);
    }
}
```

### ✅ Real-World Example - Multi-Tier Caching

```java
@Configuration
@EnableCaching
public class MultiTierCacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        // Layer 1: Local Caffeine Cache (Fast)
        CaffeineCacheManager localCache = new CaffeineCacheManager();
        localCache.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .maximumSize(500));
        
        // Layer 2: Distributed Redis Cache (Shared)
        RedisCacheManager distributedCache = RedisCacheManager.create(connectionFactory);
        
        // Composite: Try Local First, Fall Back to Redis
        return new CompositeCacheManager(localCache, distributedCache);
    }
}

@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        // Hit Order:
        // 1. Caffeine (Local) - 1μs
        // 2. Redis (Shared) - 5ms
        // 3. Database - 100ms
        return productRepository.findById(id).orElseThrow();
    }
}
```

### ✅ Monitoring Cache Hits and Misses

```java
@Configuration
@EnableCaching
public class CacheMonitoringConfig {
    
    @Bean
    public CacheManager cacheManager(MeterRegistry meterRegistry) {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager("users", "products");
        
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .recordStats());  // Enable Statistics!
        
        // Register Metrics
        for (String cacheName : cacheManager.getCacheNames()) {
            Cache cache = cacheManager.getCache(cacheName);
            new CacheMetricsCollector(cache, meterRegistry).bindTo(meterRegistry);
        }
        
        return cacheManager;
    }
}

// Metrics Available:
// - cache.hits (total, tags: cache name)
// - cache.misses (total)
// - cache.puts
// - cache.evictions
// - cache.size
```

## 💡 Why This Matters

Spring Cache Abstraction separates caching from business logic through annotations. @Cacheable checks cache before execution - most common for reads. @CachePut always executes and updates cache - for writes and updates. @CacheEvict removes entries - critical for invalidation. CacheManager abstraction allows switching providers without code changes - Caffeine, Redis, EhCache. TTL prevents unbounded cache growth - configurable per cache. Conditional caching via condition and unless parameters - smart caching decisions. SpEL expressions in cache keys - complex key generation. Multi-tier caching combines fast local cache with distributed redis.

## 🎯 Key Takeaway

Use @Cacheable for frequently accessed read data. Use @CachePut for updates to keep cache fresh. Always configure TTL - prevent infinite cache bloat. Start with Caffeine for single-instance, switch to Redis for distributed. Monitor cache hit rates - indicates effectiveness. Invalidate strategically - overly aggressive eviction defeats caching.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringBoot` `#Caching` `#Performance` `#Redis` `#CacheAbstraction` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity`
