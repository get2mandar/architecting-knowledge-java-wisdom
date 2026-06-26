# Post #42: Message Queues - Event Streaming with Kafka

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** June 28, 2026  
**Topic:** Apache Kafka, Event Streaming, Message Queues, Distributed Systems

---

## The Problem

Traditional queues are point-to-point. Kafka is publish-subscribe. Scale from one system to a thousand without rewriting architecture.

## Code Example

### ❌ Without Kafka - Tightly Coupled Services

```java
// Service A: Order Service
@Service
public class OrderService {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private NotificationService notificationService;
    
    @Transactional
    public Order placeOrder(Order order) {
        // Service A Calls Service B Directly
        inventoryService.reduceStock(order.getItems());
        
        // Service A Calls Service C Directly
        paymentService.charge(order.getTotal());
        
        // Service A Calls Service D Directly
        notificationService.sendConfirmation(order);
        
        return order;
    }
}

// Problems:
// 1. All Services Must Be Running
// 2. If Notification Fails, Order Fails!
// 3. Tight Coupling - Can't Scale Independently
// 4. New Service Requires Code Change
```

### ✅ With Kafka - Loosely Coupled Event Streaming

```java
// Service A: Publishes Event
@Service
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Transactional
    public Order placeOrder(Order order) {
        // Save Order
        Order saved = orderRepository.save(order);
        
        // Publish Event - Fire and Forget!
        OrderEvent event = new OrderEvent(
            saved.getId(),
            saved.getCustomerId(),
            saved.getTotal(),
            Instant.now()
        );
        
        kafkaTemplate.send("orders", 
            saved.getId().toString(), 
            event);
        
        return saved;
    }
}

// Services B, C, D: Subscribe Independently
@Service
public class InventoryListener {
    
    @KafkaListener(topics = "orders", groupId = "inventory-group")
    public void handleOrderPlaced(OrderEvent event) {
        inventoryService.reduceStock(event.orderId());
    }
}

// Benefits:
// 1. Services Independent - Notification Down? Order Still Placed
// 2. New Service? Just Add Kafka Listener
// 3. Scales Automatically - Add Consumer Instances
// 4. Retries Built-In - Kafka Stores Messages
```

### ✅ Understanding Kafka Architecture

```java
/*
KAFKA COMPONENTS:

Topic: Logical Channel (like a folder)
  - "orders", "payments", "notifications"
  - Durable storage of events
  - Multiple partitions for parallelism

Partition: Physical Shard Within Topic
  - Ordered log of events
  - Each event has sequential offset
  - Replication for fault tolerance
  
Broker: Kafka Server
  - Cluster of brokers
  - Each holds partition replicas
  - Auto-rebalancing on failure

Producer: Publishes to Topics
  - OrderService sends order events
  - PaymentService sends payment events
  
Consumer: Subscribes to Topics
  - InventoryListener listens for orders
  - NotificationListener listens for orders
  - Multiple consumers = Parallel Processing

Consumer Group: Scaling Mechanism
  - "inventory-group", "notification-group"
  - Partitions distributed among consumers
  - Automatic rebalancing on scale
*/
```

### ✅ Spring Kafka Configuration

```java
// application.properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.JsonDeserializer
spring.kafka.consumer.group-id=order-service
spring.kafka.consumer.auto-offset-reset=earliest  // Start from beginning if no offset

@Configuration
public class KafkaConfig {
    
    @Bean
    public NewTopic ordersTopic() {
        return TopicBuilder.name("orders")
            .partitions(3)  // 3 Partitions for Parallelism
            .replicas(2)    // 2 Replicas for Fault Tolerance
            .build();
    }
}
```

### ✅ Kafka Producer - Publishing Events

```java
public record OrderEvent(
    String orderId,
    String customerId,
    BigDecimal total,
    Instant timestamp
) {}

@Service
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderPlaced(Order order) {
        // 1. Create Event
        OrderEvent event = new OrderEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotal(),
            Instant.now()
        );
        
        // 2. Send to Kafka
        // Key: orderId (Ensures Same Order Goes to Same Partition)
        // Value: Event Data
        kafkaTemplate.send("orders", order.getId(), event)
            .addCallback(
                // Success Callback
                result -> logger.info("Sent order event: " + order.getId()),
                // Error Callback
                ex -> logger.error("Failed to send order event", ex)
            );
    }
    
    // With Acknowledgment
    public void publishWithAcknowledgment(Order order) {
        OrderEvent event = new OrderEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotal(),
            Instant.now()
        );
        
        kafkaTemplate.send("orders", order.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    logger.info("Order published at offset: " + 
                        result.getRecordMetadata().offset());
                } else {
                    logger.error("Failed: " + ex.getMessage());
                }
            });
    }
}
```

### ✅ Kafka Consumer - Processing Events

```java
@Service
public class InventoryListener {
    
    // Simple Listener
    @KafkaListener(
        topics = "orders",
        groupId = "inventory-group"
    )
    public void handleOrderPlaced(OrderEvent event) {
        logger.info("Processing order: " + event.orderId());
        inventoryService.reduceStock(event.orderId());
    }
    
    // Listener with Error Handling
    @KafkaListener(
        topics = "orders",
        groupId = "inventory-group"
    )
    public void handleWithErrorHandling(OrderEvent event) {
        try {
            inventoryService.reduceStock(event.orderId());
        } catch (Exception e) {
            logger.error("Failed to process order", e);
            // Send to Dead Letter Topic
            kafkaTemplate.send("orders-dlq", event.orderId(), event);
        }
    }
}

@Service
public class NotificationListener {
    
    // Different Group = Receives Same Message
    @KafkaListener(
        topics = "orders",
        groupId = "notification-group"
    )
    public void sendNotification(OrderEvent event) {
        logger.info("Sending notification for order: " + event.orderId());
        emailService.send(event.customerId());
    }
}
```

### ✅ Consumer Groups - Scaling Pattern

```java
// application.properties
spring.kafka.consumer.group-id=analytics-group
spring.kafka.consumer.max-poll-records=10

@Service
public class AnalyticsListener {
    
    @KafkaListener(
        topics = "orders",
        groupId = "analytics-group",
        concurrency = "3"  // 3 Consumer Instances!
    )
    public void processAnalytics(OrderEvent event) {
        // If Topic Has 3 Partitions:
        // Consumer 1: Partition 0
        // Consumer 2: Partition 1
        // Consumer 3: Partition 2
        
        analyticsService.recordOrder(event);
    }
}

// Scale to 5 Instances?
// Just Deploy 2 More - Rebalancing Automatic!
// Kafka Distributes Load: 1-1-1-1-1 Per Partition
```

### ✅ Partitioning Strategy - Ordering Guarantee

```java
// Scenario: Process All Orders by Customer in Sequence

@Service
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderWithPartitionKey(Order order) {
        OrderEvent event = new OrderEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotal(),
            Instant.now()
        );
        
        // KEY DECISION: Customer ID as Partition Key
        // All Orders for Customer "john@example.com" Go to Same Partition
        // Guarantees Sequential Processing!
        kafkaTemplate.send(
            "orders",
            order.getCustomerId(),  // Partition Key!
            event
        );
    }
}

// Order Flow:
// Customer 1: P0 - Order1 -> Order2 -> Order3 (Sequential)
// Customer 2: P1 - Order4 -> Order5 (Sequential)
// Customer 3: P2 - Order6 -> Order7 -> Order8 (Sequential)
```

### ✅ Real-World Example - Order Processing Pipeline

```java
public record OrderPlacedEvent(
    String orderId,
    String customerId,
    List<String> items,
    BigDecimal total
) {}

@Service
public class OrderPublisher {
    
    @Autowired
    private KafkaTemplate<String, OrderPlacedEvent> kafkaTemplate;
    
    public void publishOrder(Order order) {
        OrderPlacedEvent event = new OrderPlacedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getItems(),
            order.getTotal()
        );
        
        kafkaTemplate.send("orders", order.getCustomerId(), event);
    }
}

@Service
public class InventoryService {
    
    @KafkaListener(
        topics = "orders",
        groupId = "inventory-group",
        concurrency = "2"
    )
    public void reserveInventory(OrderPlacedEvent event) {
        logger.info("Reserving inventory for order: " + event.orderId());
        // Reduce stock, send event if success
        kafkaTemplate.send("inventory-reserved", event.orderId(), 
            new InventoryReservedEvent(event.orderId()));
    }
}

@Service
public class PaymentService {
    
    @KafkaListener(
        topics = "orders",
        groupId = "payment-group",
        concurrency = "3"
    )
    public void processPayment(OrderPlacedEvent event) {
        logger.info("Processing payment for order: " + event.orderId());
        // Charge customer, send event if success
        kafkaTemplate.send("payment-processed", event.orderId(),
            new PaymentProcessedEvent(event.orderId()));
    }
}

@Service
public class NotificationService {
    
    @KafkaListener(
        topics = "orders",
        groupId = "notification-group"
    )
    public void sendConfirmation(OrderPlacedEvent event) {
        logger.info("Sending confirmation for order: " + event.orderId());
        emailService.sendOrderConfirmation(event.customerId());
    }
}
```

### ✅ Offset Management - Tracking Progress

```java
@Service
public class OrderProcessingService {
    
    @KafkaListener(
        topics = "orders",
        groupId = "order-processing-group"
    )
    public void processOrder(
        OrderEvent event,
        Acknowledgment acknowledgment  // Manual Ack
    ) {
        try {
            // Process Order
            orderService.process(event);
            
            // Commit Offset After Success
            acknowledgment.acknowledge();
            logger.info("Offset committed for order: " + event.orderId());
        } catch (Exception e) {
            logger.error("Failed to process order", e);
            // Don't Commit - Will Retry From Same Offset!
        }
    }
}

// application.properties
spring.kafka.consumer.enable-auto-commit=false  // Manual Control
spring.kafka.listener.ack-mode=manual
```

## 💡 Why This Matters

Kafka is publish-subscribe, not point-to-point - multiple consumers receive same event. Partitions enable parallelism - scale by adding consumer instances. Consumer groups coordinate work - automatic rebalancing on scale/failure. Messages persist - not deleted after consumption, enabling replays. Offsets track progress - exactly once processing semantics. Event ordering guaranteed within partition - use keys strategically. Spring Kafka abstracts complexity - @KafkaListener, KafkaTemplate.

## 🎯 Key Takeaway

Use Kafka for event-driven microservices - decouple services completely. One producer, many consumers - extend system without touching code. Partition by business key (customerId, orderId) - maintain ordering. Consumer groups scale automatically - add instances, work distributes. Kafka is the backbone of modern distributed systems.

---

**Tags:** `#Java` `#JavaWisdom` `#Kafka` `#MessageQueues` `#EventStreaming` `#Microservices` `#SpringBoot` `#DistributedSystems` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity` `#SoftwareEngineering` `#Architecture`
