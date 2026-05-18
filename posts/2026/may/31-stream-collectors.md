# Post #31: Stream Collectors - Beyond toList()

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** May 20, 2026  
**Topic:** Java Stream API, Collectors, groupingBy, partitioningBy

---

## The Problem

toList() is just the beginning. Collectors unlock powerful aggregations.

## Code Example

### ❌ Manual Aggregation - Verbose and Error-Prone

```java
List<Order> orders = getOrders();

// Group by Customer Manually
Map<String, List<Order>> byCustomer = new HashMap<>();
for (Order order : orders) {
    byCustomer.computeIfAbsent(order.getCustomerId(), k -> new ArrayList<>())
        .add(order);
}

// Calculate Total Revenue Per Customer
Map<String, Double> revenueByCustomer = new HashMap<>();
for (Map.Entry<String, List<Order>> entry : byCustomer.entrySet()) {
    double total = 0;
    for (Order order : entry.getValue()) {
        total += order.getTotal();
    }
    revenueByCustomer.put(entry.getKey(), total);
}
```

### ✅ With Collectors - Declarative and Clean

```java
// Group by Customer
Map<String, List<Order>> byCustomer = orders.stream()
    .collect(Collectors.groupingBy(Order::getCustomerId));

// Calculate Total Revenue Per Customer
Map<String, Double> revenueByCustomer = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getCustomerId,
        Collectors.summingDouble(Order::getTotal)
    ));
```

### ✅ groupingBy - Group Elements by Key

```java
List<Product> products = getProducts();

// Simple Grouping by Category
Map<String, List<Product>> byCategory = products.stream()
    .collect(Collectors.groupingBy(Product::getCategory));

// Output: {Electronics=[...], Clothing=[...], Books=[...]}

// Group by Price Range
Map<String, List<Product>> byPriceRange = products.stream()
    .collect(Collectors.groupingBy(product -> {
        if (product.getPrice() < 50) return "Budget";
        if (product.getPrice() < 200) return "Mid-Range";
        return "Premium";
    }));
```

### ✅ Downstream Collectors - Aggregations Per Group

```java
// Count Products Per Category
Map<String, Long> countByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.counting()
    ));

// Average Price Per Category
Map<String, Double> avgPriceByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.averagingDouble(Product::getPrice)
    ));

// Sum Total Value Per Category
Map<String, Double> totalValueByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.summingDouble(Product::getPrice)
    ));
```

### ✅ Multi-Level Grouping - Nested Maps

```java
// Group by Category, Then by Brand
Map<String, Map<String, List<Product>>> byCategoryAndBrand = 
    products.stream()
        .collect(Collectors.groupingBy(
            Product::getCategory,
            Collectors.groupingBy(Product::getBrand)
        ));

// Output: {Electronics={Apple=[...], Samsung=[...]}, ...}

// Group by Region, Then Calculate Revenue Per Currency
Map<String, Map<String, Double>> revenueByRegionAndCurrency = 
    orders.stream()
        .collect(Collectors.groupingBy(
            Order::getRegion,
            Collectors.groupingBy(
                Order::getCurrency,
                Collectors.summingDouble(Order::getTotal)
            )
        ));
```

### ✅ partitioningBy - Split Into Two Groups

```java
// Partition Orders by Status (Paid vs Unpaid)
Map<Boolean, List<Order>> ordersByPayment = orders.stream()
    .collect(Collectors.partitioningBy(Order::isPaid));

// Access Groups
List<Order> paidOrders = ordersByPayment.get(true);
List<Order> unpaidOrders = ordersByPayment.get(false);

// Partition with Downstream Collector
Map<Boolean, Long> countByPaymentStatus = orders.stream()
    .collect(Collectors.partitioningBy(
        Order::isPaid,
        Collectors.counting()
    ));

// Output: {true=45, false=12}
```

### ✅ toMap - Custom Key-Value Mapping

```java
// Map Product ID to Product
Map<Long, Product> productMap = products.stream()
    .collect(Collectors.toMap(
        Product::getId,
        Function.identity()
    ));

// Map Product Name to Price
Map<String, Double> priceByName = products.stream()
    .collect(Collectors.toMap(
        Product::getName,
        Product::getPrice
    ));

// ❌ Duplicate Keys Throw Exception!
// Map<String, Product> byCategory = products.stream()
//     .collect(Collectors.toMap(Product::getCategory, Function.identity()));
// IllegalStateException: Duplicate Key!

// ✅ Handle Duplicates with Merge Function
Map<String, Product> byCategory = products.stream()
    .collect(Collectors.toMap(
        Product::getCategory,
        Function.identity(),
        (existing, replacement) -> existing  // Keep First
    ));
```

### ✅ mapping - Transform Before Collecting

```java
// Group Products by Category, Collect Only Names
Map<String, List<String>> namesByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.mapping(Product::getName, Collectors.toList())
    ));

// Group Orders by Customer, Collect Product Names
Map<String, Set<String>> productsByCustomer = orders.stream()
    .collect(Collectors.groupingBy(
        Order::getCustomerId,
        Collectors.flatMapping(
            order -> order.getItems().stream().map(Item::getProductName),
            Collectors.toSet()
        )
    ));
```

### ✅ collectingAndThen - Post-Process Result

```java
// Group by Category, Make Lists Immutable
Map<String, List<Product>> immutableByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.collectingAndThen(
            Collectors.toList(),
            Collections::unmodifiableList
        )
    ));

// Calculate Average Price, Round to 2 Decimals
Double avgPrice = products.stream()
    .collect(Collectors.collectingAndThen(
        Collectors.averagingDouble(Product::getPrice),
        avg -> Math.round(avg * 100.0) / 100.0
    ));
```

### ✅ joining - Concatenate Strings

```java
// Simple Join
String names = products.stream()
    .map(Product::getName)
    .collect(Collectors.joining(", "));

// With Prefix and Suffix
String namesList = products.stream()
    .map(Product::getName)
    .collect(Collectors.joining(", ", "[", "]"));

// Output: [Laptop, Mouse, Keyboard]
```

### ✅ summarizingDouble - Statistics in One Pass

```java
DoubleSummaryStatistics stats = products.stream()
    .collect(Collectors.summarizingDouble(Product::getPrice));

System.out.println("Count: " + stats.getCount());
System.out.println("Sum: " + stats.getSum());
System.out.println("Min: " + stats.getMin());
System.out.println("Max: " + stats.getMax());
System.out.println("Average: " + stats.getAverage());
```

### ✅ filtering - Filter Within Collector

```java
// Group Active Products by Category
Map<String, List<Product>> activeByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.filtering(
            Product::isActive,
            Collectors.toList()
        )
    ));

// Count Expensive Products Per Category (Price > 100)
Map<String, Long> expensiveCountByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        Collectors.filtering(
            p -> p.getPrice() > 100,
            Collectors.counting()
        )
    ));
```

### ✅ Real-World Example - Sales Report

```java
public class SalesReport {
    
    public Map<String, ReportData> generateReport(List<Order> orders) {
        return orders.stream()
            .filter(o -> o.getStatus() == OrderStatus.COMPLETED)
            .collect(Collectors.groupingBy(
                Order::getRegion,
                Collectors.collectingAndThen(
                    Collectors.toList(),
                    this::buildReportData
                )
            ));
    }
    
    private ReportData buildReportData(List<Order> orders) {
        double totalRevenue = orders.stream()
            .mapToDouble(Order::getTotal)
            .sum();
        
        long orderCount = orders.size();
        
        Set<String> uniqueCustomers = orders.stream()
            .map(Order::getCustomerId)
            .collect(Collectors.toSet());
        
        return new ReportData(totalRevenue, orderCount, uniqueCustomers.size());
    }
}

record ReportData(double revenue, long orders, long customers) {}
```

### ✅ Custom Collector - Advanced Use Case

```java
// Collect Top N Most Expensive Products Per Category
public static Collector<Product, ?, List<Product>> topNByPrice(int n) {
    return Collector.of(
        () -> new PriorityQueue<>(
            Comparator.comparingDouble(Product::getPrice).reversed()
        ),
        PriorityQueue::add,
        (queue1, queue2) -> {
            queue1.addAll(queue2);
            return queue1;
        },
        queue -> queue.stream()
            .limit(n)
            .collect(Collectors.toList())
    );
}

// Usage
Map<String, List<Product>> top3ByCategory = products.stream()
    .collect(Collectors.groupingBy(
        Product::getCategory,
        topNByPrice(3)
    ));
```

### ✅ teeing - Combine Two Collectors

```java
// Calculate Min and Max Price Simultaneously
record PriceRange(double min, double max) {}

PriceRange range = products.stream()
    .collect(Collectors.teeing(
        Collectors.minBy(Comparator.comparingDouble(Product::getPrice)),
        Collectors.maxBy(Comparator.comparingDouble(Product::getPrice)),
        (min, max) -> new PriceRange(
            min.map(Product::getPrice).orElse(0.0),
            max.map(Product::getPrice).orElse(0.0)
        )
    ));
```

## 💡 Why This Matters

Collectors transform streams into various data structures - Lists, Sets, Maps, summaries. groupingBy groups elements by classification function - returns Map<K, List<V>>. partitioningBy splits into two groups based on predicate - returns Map<Boolean, List<T>>. toMap creates custom key-value mappings - requires merge function for duplicate keys. Downstream collectors aggregate within groups (counting, summing, averaging). mapping and flatMapping transform elements before collecting. collectingAndThen applies final transformation to collected result. filtering filters elements within downstream collector (Java 9+).

## 🎯 Key Takeaway

Use groupingBy for multi-level aggregations instead of nested loops. partitioningBy for binary splits (active/inactive, valid/invalid). Always handle duplicate keys in toMap with merge function. Downstream collectors enable complex aggregations in single pass. Custom collectors solve specialized collection requirements.

---

**Tags:** `#Java` `#JavaWisdom` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#StreamAPI` `#Collectors` `#FunctionalProgramming` `#Java8`
