# Post #62: Exactly-Once Semantics in Streaming Features - Avoiding Duplicate Processing

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** September 6, 2026  
**Topic:** Exactly-Once Semantics, Idempotent Producers, Transactions, Deduplication

---

## The Problem

A feature pipeline processes 100K transactions per day. Your Kafka broker crashes. Producer retries the message. Message gets processed twice. Features are counted twice. Your model sees double the user activity. Prediction: completely wrong. This is the exactly-once problem—deliver messages without loss, without duplication.

## Code Example

### ❌ At-Least-Once - Silent Duplication

```java
// Default Kafka Producer = At-Least-Once
@Service
public class AtLeastOnceProducer {
    
    @Autowired
    private KafkaTemplate<String, FeatureEvent> kafkaTemplate;
    
    public void sendFeatureUpdate(String userId, int purchaseAmount) {
        // Producer Sends Message
        kafkaTemplate.send("features", userId, 
            new FeatureEvent(userId, purchaseAmount, Instant.now())
        ).addCallback(
            success -> logger.info("Sent!"),
            error -> {
                // Network Error? Retry Automatically
                // Message Might Already Be on Broker!
                logger.warn("Retry", error);
                
                // Problem: Automatic Retry Without Idempotency
                // Result: Duplicate Message on Broker
            }
        );
    }
}

// Broker Crashes Scenario:
// 1. Producer: Send "purchase: $100"
// 2. Broker: Received (In Memory, Not Yet Flushed)
// 3. Broker: Crashes Before Persisting
// 4. Producer Timeout: Never Got ACK
// 5. Producer Auto-Retry: Send Again
// 6. Broker Recovers: Has Old Replica Without Message
// 7. Message Processed Twice
//
// Consumer Sees:
// Message 1: purchase: $100
// Message 2: purchase: $100 (Duplicate!)
// Result: Feature = 200 (Should Be 100)
```

### ✅ Solution 1: Idempotent Producer - Broker-Side Deduplication

```java
/*
IDEMPOTENT PRODUCER (Kafka 0.11+):
  - Producer ID (PID) + Sequence Number Per Message
  - Broker Detects Duplicates: Same PID + Same Sequence
  - Rejects Duplicate Automatically
  - Works at Broker Level (Transparent to Consumer)
*/

@Configuration
public class IdempotentProducerConfig {
    
    @Bean
    public ProducerFactory<String, FeatureEvent> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        
        // Enable Idempotence
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        
        // Required Settings for Idempotence
        props.put(ProducerConfig.ACKS_CONFIG, "all");  // Wait for All Replicas
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);  // Unlimited Retries
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, 
            StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, 
            JsonSerializer.class);
        
        return new DefaultKafkaProducerFactory<>(props);
    }
    
    @Bean
    public KafkaTemplate<String, FeatureEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}

@Service
public class IdempotentProducer {
    
    @Autowired
    private KafkaTemplate<String, FeatureEvent> kafkaTemplate;
    
    public void sendFeatureUpdate(String userId, int purchaseAmount) {
        FeatureEvent event = new FeatureEvent(
            userId,
            purchaseAmount,
            Instant.now()
        );
        
        // With Idempotent Producer:
        // 1. Producer Assigns Unique Producer ID (PID)
        // 2. Message Gets Sequence Number
        // 3. Broker Deduplicates Based on PID + Sequence
        // 4. Retry Safe!
        
        kafkaTemplate.send("features", userId, event)
            .addCallback(
                success -> logger.info("Sent with PID Deduplication"),
                error -> {
                    // Retry is Safe
                    // Broker Will Reject Duplicate
                    logger.warn("Will Retry Safely", error);
                }
            );
    }
}

// How It Works:
// Producer ID: pid-123
// Message 1: Sequence 1 -> Broker Accepts
// Message 2: Sequence 2 -> Broker Accepts
// Retry Message 1: Sequence 1 -> Broker Rejects (Duplicate!)
// Result: Message Processed Exactly Once on Broker
```

### ✅ Solution 2: Transactional Producer - Atomic Multi-Partition

```java
/*
TRANSACTIONAL PRODUCER:
  - Idempotent Producer + Atomicity Across Partitions
  - Read-Process-Write Must Be Atomic
  - Two-Phase Commit Between Consumer Offset + Output
  - Exactly-Once End-to-End
*/

@Service
public class TransactionalProducer {
    
    @Autowired
    private KafkaTemplate<String, FeatureEvent> kafkaTemplate;
    
    @Transactional
    public void processAndPublish(String userId, int purchaseAmount) {
        // This Entire Method Is Atomic
        // Either All Succeed or All Fail
        
        // Step 1: Save to Database (Same Transaction)
        FeatureUpdate update = new FeatureUpdate(
            userId,
            purchaseAmount,
            LocalDateTime.now()
        );
        featureRepository.save(update);
        
        // Step 2: Publish to Kafka (Same Transaction)
        // If Database Fails: Kafka Write Rolled Back
        // If Kafka Fails: Database Write Rolled Back
        kafkaTemplate.send("features", userId, 
            new FeatureEvent(userId, purchaseAmount, Instant.now())
        );
        
        // Both Succeed or Both Fail
        // No Partial State!
    }
}

// Producer Configuration for Transactions
@Configuration
public class TransactionalProducerConfig {
    
    @Bean
    public ProducerFactory<String, FeatureEvent> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        
        // Enable Transactions
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "feature-pipeline-1");
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        
        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

### ✅ Solution 3: Consumer-Side Deduplication - Idempotency Key

```java
/*
CONSUMER IDEMPOTENCY:
  - Deduplicate at Consumer (When Broker Dedup Not Enough)
  - Use Idempotency Key = Unique ID Per Operation
  - Database Upsert with Unique Constraint
  - Replay-Safe Processing
*/

@Service
public class IdempotentConsumer {
    
    public record FeatureEvent(
        String eventId,        // Unique ID
        String userId,
        int purchaseAmount,
        Instant timestamp
    ) {}
    
    @KafkaListener(topics = "features", groupId = "feature-group")
    public void processFeatureEvent(FeatureEvent event) {
        // Idempotency Key = eventId
        // Same eventId Always Produces Same Result
        
        featureService.processIdempotently(
            event.eventId,
            event.userId,
            event.purchaseAmount
        );
    }
}

@Service
public class IdempotentFeatureService {
    
    @Transactional
    public void processIdempotently(
        String idempotencyKey,
        String userId,
        int purchaseAmount
    ) {
        // Check: Has This Been Processed?
        Optional<ProcessedEvent> existing = 
            processedEventRepository.findByIdempotencyKey(idempotencyKey);
        
        if (existing.isPresent()) {
            // Already Processed: Return Cached Result
            logger.info("Duplicate Detected: {}", idempotencyKey);
            return;  // Replay Safe!
        }
        
        // Step 1: Process (Update Feature Table)
        Feature feature = featureRepository
            .findById(userId)
            .orElse(new Feature(userId));
        
        feature.setPurchaseCount(feature.getPurchaseCount() + 1);
        feature.setTotalSpent(feature.getTotalSpent() + purchaseAmount);
        featureRepository.save(feature);
        
        // Step 2: Log As Processed (Idempotency Guarantee)
        ProcessedEvent record = new ProcessedEvent();
        record.setIdempotencyKey(idempotencyKey);
        record.setUserId(userId);
        record.setProcessedAt(LocalDateTime.now());
        
        processedEventRepository.save(record);
        
        // Step 3: Publish Downstream Event
        eventBus.publish(new FeatureUpdateEvent(userId, feature));
    }
}

// Database Schema for Idempotency:
// CREATE TABLE processed_events (
//     idempotency_key VARCHAR(255) UNIQUE NOT NULL,  -- Unique Constraint!
//     user_id VARCHAR(255),
//     processed_at TIMESTAMP,
//     PRIMARY KEY (idempotency_key)
// );
//
// Duplicate Attempt:
// INSERT INTO processed_events (idempotency_key, user_id, processed_at)
// VALUES ('event-123', 'user-456', NOW())
// ON CONFLICT (idempotency_key) DO NOTHING;  -- Silently Ignore Duplicate
//
// Result: Operation Idempotent
```

### ✅ Real-World Example - Financial Transaction Pipeline

```java
@Service
public class FinancialFeaturePipeline {
    
    public record MoneyTransferEvent(
        String transferId,        // Idempotency Key
        String fromUserId,
        String toUserId,
        BigDecimal amount,
        Instant timestamp
    ) {}
    
    @Transactional
    public void processMoneyTransfer(MoneyTransferEvent event) {
        // Step 1: Check If Already Processed (Idempotency)
        Optional<ProcessedTransfer> existing = 
            transferRepository.findByTransferId(event.transferId);
        
        if (existing.isPresent()) {
            logger.warn("Duplicate Transfer: {} (Already Processed)", 
                event.transferId);
            return;  // Safe Replay
        }
        
        // Step 2: Update Features (Within Transaction)
        Feature fromFeatures = featureRepository.findById(event.fromUserId)
            .orElse(new Feature(event.fromUserId));
        Feature toFeatures = featureRepository.findById(event.toUserId)
            .orElse(new Feature(event.toUserId));
        
        // Update Features
        fromFeatures.setTotalSent(fromFeatures.getTotalSent().add(event.amount));
        fromFeatures.setTransactionCount(fromFeatures.getTransactionCount() + 1);
        
        toFeatures.setTotalReceived(toFeatures.getTotalReceived().add(event.amount));
        toFeatures.setReceivedCount(toFeatures.getReceivedCount() + 1);
        
        featureRepository.saveAll(List.of(fromFeatures, toFeatures));
        
        // Step 3: Log As Processed (Atomic with Feature Update)
        ProcessedTransfer processed = new ProcessedTransfer();
        processed.setTransferId(event.transferId);
        processed.setProcessedAt(LocalDateTime.now());
        transferRepository.save(processed);
        
        // Step 4: Publish To Downstream (Still in Same Transaction)
        kafkaTemplate.send("transfer-completed", event.transferId,
            new TransferCompletedEvent(event.fromUserId, event.toUserId, event.amount)
        );
        
        // All 4 Steps Atomic
        // Failure Anywhere: Entire Transaction Rolls Back
        // Retry: Previous Results Replayed
    }
    
    // Exactly-Once Guarantee:
    // - Broker: Deduplicates at Producer
    // - Consumer: Deduplicates at Processing
    // - Feature Store: Idempotent Updates
    // - Downstream: Receives Message Exactly Once
    // = Exactly-Once Pipeline!
}

// Failure Scenarios Handled:
// 1. Producer Crash Before Send -> Message Lost (Retry Recovers)
// 2. Producer Crash After Send -> Message Duplicated (Idempotent Producer Dedup)
// 3. Consumer Crash Before Process -> Message Reprocessed (Idempotent Consumer)
// 4. Consumer Crash After Process -> Message Not Reprocessed (Offset Committed)
// 5. Feature Store Crash -> Update Retried (Unique Constraint on Idempotency Key)
// 
// All Scenarios Result in: EXACTLY-ONCE Delivery
```

### ✅ Comparing Delivery Guarantees

```java
/*
AT-MOST-ONCE: acks=0
  - Send, Don't Wait for Confirmation
  - Throughput: High
  - Latency: Low
  - Data Loss: Possible
  - Duplicates: No
  - Use Case: Metrics, Logs, Non-Critical Events

AT-LEAST-ONCE: acks=1 (Broker Received)
  - Send, Wait for Broker ACK
  - Throughput: Medium
  - Latency: Medium
  - Data Loss: No
  - Duplicates: Possible
  - Use Case: Most Applications (With Deduplication)

EXACTLY-ONCE: acks=all + Idempotence + Transactions
  - Send, Wait for All Replicas + Idempotent + Transactional
  - Throughput: Low (3-5ms latency + Coordination)
  - Latency: High
  - Data Loss: No
  - Duplicates: No
  - Use Case: Financial Transactions, Billing, Critical Operations
*/

public class DeliveryGuaranteeComparison {
    
    // Scenario: Process 1 Million Purchase Events
    
    // At-Most-Once: ~100K Lost Events (1% Loss)
    // - Throughput: 500K events/sec
    // - Cost: Low
    // - Risk: High
    
    // At-Least-Once: 0 Lost, 50K Duplicate Events
    // - Throughput: 250K events/sec
    // - Cost: Medium (Deduplication Overhead)
    // - Risk: Medium (Duplicates Must Be Handled)
    
    // Exactly-Once: 0 Lost, 0 Duplicates
    // - Throughput: 80K events/sec (10% Cost)
    // - Cost: High (Transactions, Coordination)
    // - Risk: Low
    
    // Choice:
    // - Logs/Metrics -> At-Most-Once
    // - Event Analytics -> At-Least-Once + Dedup
    // - Financial -> Exactly-Once
}
```

## 💡 Why This Matters

Exactly-once semantics is the highest data integrity guarantee in streaming, essential for financial, billing, and compliance-driven applications. Kafka achieves exactly-once through idempotent producers (sequence-number deduplication at broker) and transactions (atomic multi-partition writes). End-to-end exactly-once requires coordination across source (readable), processor (transactional), and sink (idempotent or transactional). The cost is 2-5ms additional latency and 10-20% throughput reduction—worthwhile for financial systems, not for analytics.

## 🎯 Key Takeaway

Use exactly-once semantics for critical operations (money, billing, features affecting pricing). At-least-once with idempotency keys is production-standard. At-most-once only for non-critical telemetry. Enable idempotent producers by default. Implement consumer-side deduplication with unique constraints. Feature pipelines should use at least at-least-once with deduplication.

---

**Tags:** `#Java` `#JavaWisdom` `#ExactlyOnceSemantics` `#Kafka` `#Idempotence` `#Streaming` `#DataIntegrity` `#Transactions` `#SpringBoot` `#DistributedSystems` `#MessageQueues` `#Reliability` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
```
