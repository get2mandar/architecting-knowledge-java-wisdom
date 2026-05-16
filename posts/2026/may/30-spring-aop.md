# Post #30: Spring AOP - Aspects Without the Mystery

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** May 17, 2026  
**Topic:** Spring AOP, Aspects, Pointcuts, Advice

---

## The Problem

Cross-cutting concerns pollute your code. AOP separates them cleanly.

## Code Example

### ❌ Without AOP - Logging Scattered Everywhere

```java
@Service
public class UserService {
    
    public User createUser(User user) {
        logger.info("Creating user: " + user.getEmail());  // Logging
        long start = System.currentTimeMillis();  // Performance
        
        try {
            User saved = userRepository.save(user);
            
            long duration = System.currentTimeMillis() - start;
            logger.info("User created in " + duration + "ms");  // Logging
            
            return saved;
        } catch (Exception e) {
            logger.error("Failed to create user", e);  // Logging
            throw e;
        }
    }
    
    public void deleteUser(Long id) {
        logger.info("Deleting user: " + id);  // Logging
        long start = System.currentTimeMillis();  // Performance
        
        // Same Boilerplate Everywhere!
    }
}
```

### ✅ With AOP - Clean Separation of Concerns

```java
@Service
public class UserService {
    
    public User createUser(User user) {
        return userRepository.save(user);
        // Business Logic Only - No Logging/Performance Code!
    }
    
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
        // Clean and Focused
    }
}

@Aspect
@Component
public class LoggingAspect {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        logger.info("Calling: " + joinPoint.getSignature().getName());
    }
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object measurePerformance(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long duration = System.currentTimeMillis() - start;
        logger.info(pjp.getSignature().getName() + " took " + duration + "ms");
        return result;
    }
}
```

### ✅ Understanding AOP Terminology

```java
/*
Aspect - Module that encapsulates cross-cutting logic
    @Aspect class with advice methods

Join Point - Point in execution where aspect can be applied
    Method execution, exception handling, etc.

Pointcut - Expression matching join points
    execution(* com.example.service.*.*(..))

Advice - Action taken at a join point
    @Before, @After, @Around, @AfterReturning, @AfterThrowing

Weaving - Linking aspects with target objects
    Runtime proxies (Spring AOP) or compile-time (AspectJ)
*/
```

### ✅ Pointcut Expressions - Matching Methods

```java
@Aspect
@Component
public class PointcutExamples {
    
    // All Methods in Service Package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}
    
    // All Public Methods
    @Pointcut("execution(public * *(..))")
    public void publicMethods() {}
    
    // Methods Starting with 'get'
    @Pointcut("execution(* get*(..))")
    public void getters() {}
    
    // Methods with @Transactional
    @Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
    public void transactionalMethods() {}
    
    // All Methods in Classes with @Service
    @Pointcut("@within(org.springframework.stereotype.Service)")
    public void serviceClasses() {}
    
    // Combine Pointcuts
    @Pointcut("serviceMethods() && publicMethods()")
    public void publicServiceMethods() {}
}
```

### ✅ @Before Advice - Execute Before Method

```java
@Aspect
@Component
public class SecurityAspect {
    
    @Before("execution(* com.example.service.*.delete*(..))")
    public void checkPermissions(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        
        logger.info("Security check before: " + methodName);
        
        if (!securityService.hasDeletePermission()) {
            throw new AccessDeniedException("No delete permission");
        }
    }
}
```

### ✅ @After Advice - Execute After Method (Finally)

```java
@Aspect
@Component
public class AuditAspect {
    
    @After("execution(* com.example.service.*.*(..))")
    public void auditMethodCall(JoinPoint joinPoint) {
        String method = joinPoint.getSignature().getName();
        String user = getCurrentUser();
        
        auditLog.record(user, method, Instant.now());
        // Runs Whether Method Succeeds or Fails
    }
}
```

### ✅ @AfterReturning - Execute After Successful Return

```java
@Aspect
@Component
public class CacheAspect {
    
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.find*(..))",
        returning = "result"
    )
    public void cacheResult(JoinPoint joinPoint, Object result) {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        
        String cacheKey = generateKey(methodName, args);
        cache.put(cacheKey, result);
        
        logger.info("Cached result for: " + methodName);
    }
}
```

### ✅ @AfterThrowing - Execute After Exception

```java
@Aspect
@Component
public class ErrorHandlingAspect {
    
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "exception"
    )
    public void handleException(JoinPoint joinPoint, Exception exception) {
        String method = joinPoint.getSignature().getName();
        
        logger.error("Exception in " + method, exception);
        
        notificationService.alertOps(method, exception);
    }
}
```

### ✅ @Around Advice - Most Powerful, Full Control

```java
@Aspect
@Component
public class PerformanceAspect {
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object measureExecutionTime(ProceedingJoinPoint pjp) throws Throwable {
        String methodName = pjp.getSignature().getName();
        
        logger.info("Starting: " + methodName);
        long start = System.nanoTime();
        
        try {
            Object result = pjp.proceed();  // Execute Target Method
            
            long duration = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
            logger.info(methodName + " completed in " + duration + "ms");
            
            return result;
        } catch (Exception e) {
            logger.error(methodName + " failed after " + 
                TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start) + "ms");
            throw e;
        }
    }
}
```

### ✅ Accessing Method Arguments

```java
@Aspect
@Component
public class ValidationAspect {
    
    @Before("execution(* com.example.service.UserService.createUser(..)) && args(user)")
    public void validateUser(User user) {
        if (user.getEmail() == null || !user.getEmail().contains("@")) {
            throw new ValidationException("Invalid email");
        }
        
        logger.info("Validated user: " + user.getEmail());
    }
}
```

### ✅ Custom Annotation-Based Pointcuts

```java
// Define Custom Annotation
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Timed {
}

// Apply to Methods
@Service
public class ProductService {
    
    @Timed
    public List<Product> findAllProducts() {
        return productRepository.findAll();
    }
}

// Aspect Matching Annotation
@Aspect
@Component
public class TimedAspect {
    
    @Around("@annotation(com.example.annotation.Timed)")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        long duration = System.currentTimeMillis() - start;
        
        logger.info(pjp.getSignature().getName() + " took " + duration + "ms");
        return result;
    }
}
```

### ✅ Ordering Multiple Aspects

```java
@Aspect
@Component
@Order(1)  // Runs First
public class SecurityAspect {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void checkSecurity() {
        // Security Checks Run First
    }
}

@Aspect
@Component
@Order(2)  // Runs Second
public class LoggingAspect {
    
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore() {
        // Logging Runs After Security
    }
}
```

### ✅ Real-World Example - Retry Logic

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Retry {
    int maxAttempts() default 3;
    long delay() default 1000;
}

@Aspect
@Component
public class RetryAspect {
    
    @Around("@annotation(retry)")
    public Object retry(ProceedingJoinPoint pjp, Retry retry) throws Throwable {
        int attempts = 0;
        Exception lastException = null;
        
        while (attempts < retry.maxAttempts()) {
            try {
                return pjp.proceed();
            } catch (Exception e) {
                lastException = e;
                attempts++;
                
                logger.warn("Attempt " + attempts + " failed: " + e.getMessage());
                
                if (attempts < retry.maxAttempts()) {
                    Thread.sleep(retry.delay());
                }
            }
        }
        
        throw new RetryExhaustedException("Failed after " + attempts + " attempts", lastException);
    }
}

// Usage
@Service
public class PaymentService {
    
    @Retry(maxAttempts = 5, delay = 2000)
    public void processPayment(Payment payment) {
        externalPaymentGateway.charge(payment);
    }
}
```

### ❌ Common Pitfall - Self-Invocation Doesn't Work

```java
@Service
public class UserService {
    
    public void publicMethod() {
        this.privateMethod();  // AOP Won't Apply!
        // Spring Proxy Not Invoked
    }
    
    @Timed
    private void privateMethod() {
        // @Timed Aspect Never Runs
    }
}

// ✅ Fix: Call Through Proxy or Make Public
@Service
public class UserService {
    
    @Autowired
    private UserService self;  // Inject Proxy
    
    public void publicMethod() {
        self.privateMethod();  // AOP Applies!
    }
    
    @Timed
    public void privateMethod() {
        // Now Works
    }
}
```

## 💡 Why This Matters

AOP separates cross-cutting concerns (logging, security, transactions, caching) from business logic. Spring AOP uses runtime proxies - only works on public methods of Spring beans. Pointcut expressions define where advice applies - execution, annotation, within, args. @Before runs before method, @After runs finally, @Around wraps method with full control. @AfterReturning captures return value, @AfterThrowing captures exceptions. Use @Order to control aspect execution order. Self-invocation bypasses AOP - methods must be called from outside the class. Custom annotations make AOP more readable and reusable.

## 🎯 Key Takeaway

Use AOP for cross-cutting concerns that pollute multiple classes. @Around is most powerful but use simpler advice types when possible. Define reusable pointcuts with @Pointcut annotation. Custom annotations make aspects more maintainable. Remember: Spring AOP is proxy-based - it only intercepts external calls.

---

**Tags:** `#Java` `#JavaWisdom` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringAOP` `#AOP` `#CrossCuttingConcerns` `#Spring`
