# Post #17: Virtual Threads - Rethinking Concurrency

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** April 1, 2026  
**Topic:** Virtual Threads, Project Loom, Concurrency

---

## The Problem

Million threads. One JVM. Welcome to Project Loom.

## Code Example

### ❌ Traditional Threads - Expensive and Limited
```java
ExecutorService executor = Executors.newFixedThreadPool(200);

for (int i = 0; i < 10000; i++) {
    final int taskId = i;
    executor.submit(() -> {
        // Queued! Only 200 Run Concurrently
        fetchUserData(taskId);
    });
}
// 9800 Tasks Waiting in Queue, Eating Memory
```

### ❌ Pooling Virtual Threads - Missing the Point
```java
ExecutorService pool = Executors.newFixedThreadPool(200, 
    Thread.ofVirtual().factory());
// Don't Pool Virtual Threads! They're Cheap to Create
```

### ✅ Virtual Threads - Create Millions Without Pooling
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10000; i++) {
        final int taskId = i;
        executor.submit(() -> {
            fetchUserData(taskId);  // All Run Concurrently!
        });
    }
}
// 10,000 Virtual Threads, Minimal Memory Footprint
```

### ✅ Simple API - Just Like Regular Threads
```java
Thread.startVirtualThread(() -> {
    String data = fetchFromDatabase();  // Blocking is OK!
    processData(data);
});
```

### ✅ Spring Boot Integration - One Property
```properties
# application.properties
spring.threads.virtual.enabled=true

# Now Every Request Runs on a Virtual Thread Automatically!
```

### ❌ Virtual Thread Pinning - The Gotcha
```java
synchronized (lock) {
    Thread.sleep(1000);  // Pins Carrier Thread!
    // Other Virtual Threads Can't Use This Carrier
}
```

### ✅ Use ReentrantLock Instead
```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    Thread.sleep(1000);  // Virtual Thread Yields Properly ✓
} finally {
    lock.unlock();
}
```

## 💡 Why This Matters

Platform threads map 1:1 to OS threads - each consumes ~1MB stack memory. You can create maybe 4000-10000 before your JVM dies. Virtual threads are JVM-managed and stack memory is allocated dynamically on heap - you can create millions. When a virtual thread blocks on I/O, the JVM unmounts it from its carrier thread (platform thread), letting that carrier run other virtual threads. This makes blocking I/O efficient again - no more callback hell. BUT beware thread pinning: synchronized blocks prevent unmounting in Java 21-23 (fixed in 24). Use ReentrantLock for long blocking operations.

## 🎯 Key Takeaway

Virtual threads aren't faster - they're cheaper and more scalable. Perfect for I/O-bound workloads (web servers, microservices, API calls). Don't pool them, don't cache in ThreadLocal aggressively. Write simple blocking code, let Loom handle concurrency.

---

**Tags:** `#Java` `#JavaWisdom` `#VirtualThreads` `#ProjectLoom` `#Concurrency` `#Java21`
