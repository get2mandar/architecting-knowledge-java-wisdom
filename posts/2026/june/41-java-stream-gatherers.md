# Post #41: Java Streams - Gatherers (Custom Stream Operations)

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** June 24, 2026
**Topic:** Java Streams, Gatherers, Custom Intermediate Operations, Java 22+

---

## The Problem

Streams had a gap. You could transform (filter, map) or terminate (collect, reduce). But intermediate operations with state? You were stuck.

## Code Example

### ❌ Before Gatherers - Manual Batching Hell

```java
// Group Orders Into Batches of 100 for Bulk Insert
List<Order> orders = getOrders();  // 50,000 Orders

List<List<Order>> batches = new ArrayList<>();
for (int i = 0; i < orders.size(); i += 100) {
    batches.add(orders.subList(i, Math.min(i + 100, orders.size())));
}

for (List<Order> batch : batches) {
    database.insertBatch(batch);
}

// Problems:
// 1. Imperative Loop - Not Functional
// 2. Index Arithmetic - Error-Prone
// 3. Can't Chain with Stream Operations
// 4. Not Lazy - Creates All Batches in Memory
```

### ❌ Collectors - Terminates Stream

```java
// Collectors End the Pipeline!
List<Order> orders = getOrders();

Map<Integer, List<Order>> batches = orders.stream()
    .collect(Collectors.groupingBy(
        order -> order.getId() / 100  // Grouping Key
    ));

// Problem: Stream Dies After collect()
// Can't Continue with More Operations
```

### ✅ Gatherers - Stateful Intermediate Operations (Java 22+)

```java
// Group Into Fixed-Size Batches
List<Order> orders = getOrders();

orders.stream()
    .gather(Gatherers.windowFixed(100))  // Intermediate!
    .forEach(database::insertBatch);

// Benefits:
// 1. Declarative - No Index Arithmetic
// 2. Lazy - Processes One Batch at a Time
// 3. Chainable - Can Continue with More Operations
// 4. Clean API - Just Like filter() and map()
```

### ✅ Built-In Gatherers - Pre-Made Operations

```java
// 1. windowFixed() - Fixed-Size Batches
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9);
numbers.stream()
    .gather(Gatherers.windowFixed(3))
    .forEach(batch -> System.out.println(batch));
// Output: [1, 2, 3], [4, 5, 6], [7, 8, 9]

// 2. windowSliding() - Sliding Window
numbers.stream()
    .gather(Gatherers.windowSliding(2))
    .forEach(window -> System.out.println(window));
// Output: [1, 2], [2, 3], [3, 4], [4, 5], ...

// 3. fold() - Reduction-Like Transformation
String result = Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.fold(
        () -> "",  // Initial Value
        (acc, num) -> acc + num  // Accumulator
    ))
    .findFirst()
    .orElse("");
// Output: "12345"

// 4. scan() - Prefix Scan (Emit Each Intermediate)
Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.scan(
        () -> 0,  // Initial
        (acc, num) -> acc + num  // Running Sum
    ))
    .forEach(System.out::println);
// Output: 1, 3, 6, 10, 15

// 5. mapConcurrent() - Parallel Mapping
userIds.stream()
    .gather(Gatherers.mapConcurrent(10, id -> {
        // Fetch User from API in Parallel
        return userService.fetchUser(id);
    }))
    .forEach(user -> System.out.println(user));
```

### ✅ Real-World Example - Batch Processing Pipeline

```java
@Service
public class OrderProcessor {
    
    public void processMassiveOrderList(List<Order> orders) {
        orders.stream()
            .filter(order -> order.getStatus() == PENDING)  // Filter
            .gather(Gatherers.windowFixed(100))  // Batch Into 100
            .peek(batch -> logger.info("Processing batch of {}", batch.size()))
            .map(this::enrichOrders)  // Enrich Each Order
            .gather(Gatherers.windowFixed(50))  // Sub-Batch for DB Inserts
            .forEach(batch -> database.insertOrdersInTransaction(batch));
    }
    
    private List<Order> enrichOrders(List<Order> batch) {
        return batch.stream()
            .peek(order -> order.setEnrichedAt(Instant.now()))
            .collect(Collectors.toList());
    }
}
```

### ✅ Custom Gatherer - Rolling Average

```java
// Calculate Rolling Average of Last 3 Numbers
public static Gatherer<Integer, ?, Integer> rollingAverage(int windowSize) {
    return Gatherer.ofSequential(
        () -> new LinkedList<Integer>(),  // State: Ring Buffer
        (buffer, element, downstream) -> {
            ((LinkedList<Integer>) buffer).addLast(element);
            
            if (((LinkedList<Integer>) buffer).size() > windowSize) {
                ((LinkedList<Integer>) buffer).removeFirst();
            }
            
            if (((LinkedList<Integer>) buffer).size() == windowSize) {
                int avg = ((LinkedList<Integer>) buffer).stream()
                    .mapToInt(Integer::intValue)
                    .sum() / windowSize;
                downstream.push(avg);
            }
        }
    );
}

// Usage
Stream.of(10, 20, 30, 40, 50)
    .gather(rollingAverage(3))
    .forEach(System.out::println);
// Output: 20 (10+20+30)/3, 30 (20+30+40)/3, 40 (30+40+50)/3
```

### ✅ Custom Gatherer - Deduplication

```java
// Remove Consecutive Duplicates
public static <T> Gatherer<T, ?, T> distinctConsecutive() {
    return Gatherer.ofSequential(
        () -> new Object[] {null},  // State: Previous Element
        (state, element, downstream) -> {
            if (!element.equals(state[0])) {
                state[0] = element;
                downstream.push(element);
            }
        }
    );
}

// Usage
Stream.of(1, 1, 2, 2, 2, 3, 3, 1)
    .gather(distinctConsecutive())
    .forEach(System.out::println);
// Output: 1, 2, 3, 1
```

### ✅ Custom Gatherer - Conditional Batching

```java
// Batch Orders Until Total Amount Exceeds Threshold
public static Gatherer<Order, ?, List<Order>> batchByTotal(BigDecimal threshold) {
    return Gatherer.ofSequential(
        ArrayList::new,  // State: Current Batch
        (batch, order, downstream) -> {
            batch.add(order);
            
            BigDecimal total = batch.stream()
                .map(Order::getAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);
            
            if (total.compareTo(threshold) >= 0) {
                downstream.push(new ArrayList<>(batch));
                batch.clear();
            }
        },
        (batch, downstream) -> {
            // Finisher: Emit Remaining Orders
            if (!batch.isEmpty()) {
                downstream.push(batch);
            }
        }
    );
}

// Usage
orders.stream()
    .gather(batchByTotal(new BigDecimal("10000")))
    .forEach(batch -> database.insertBatch(batch));
```

### ✅ Combining Gatherers - Complex Pipelines

```java
users.stream()
    .filter(user -> user.isActive())  // Filter: Intermediate
    .gather(Gatherers.windowFixed(50))  // Batch: Intermediate Gatherer
    .gather(Gatherers.mapConcurrent(5, batch -> {  // Async Map: Gatherer
        return enrichUserBatch(batch);
    }))
    .map(enrichedBatch -> buildReport(enrichedBatch))  // Transform: Intermediate
    .forEach(report -> publishReport(report));  // Terminal
```

### ❌ When NOT to Use Gatherers

```java
// ❌ Simple Transformations - Use Regular Operators
numbers.stream()
    .gather(Gatherers.mapConcurrent(10, n -> n * 2))  // Wrong!
    .forEach(System.out::println);

// ✅ Right - Use map()
numbers.stream()
    .map(n -> n * 2)
    .forEach(System.out::println);

// ❌ Terminal Operation - Wrong Place for Gatherer
// Gatherers are Intermediate Only!
// Can't Use: .gather(...).collect(...)
```

## 💡 Why This Matters

Gatherers fill the gap between lazy intermediate operations and stateful terminal operations. Unlike collectors which terminate the stream, gatherers maintain pipeline laziness. Built-in gatherers (windowFixed, scan, fold) solve common patterns without custom code. Custom gatherers enable complex stateful transformations while maintaining functional style. mapConcurrent provides controlled parallelism with virtual threads - no thread pool configuration needed. Gatherers compose with standard Stream operations - filter, map, sorted still work.

## 🎯 Key Takeaway

Use Gatherers for batching, windowing, and stateful intermediate transformations. windowFixed() for fixed-size batches, windowSliding() for overlapping windows. scan() for prefix accumulation, fold() for ordered reduction. Create custom gatherers for domain-specific logic. Gatherers enable powerful streaming pipelines that were impossible before Java 22.

---

**Tags:** `#Java` `#JavaWisdom` `#StreamAPI` `#Gatherers` `#Java22` `#FunctionalProgramming` `#Streams` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#ServerSide` `#JavaCommunity` `#DevCommunity`
