# Post #54: The Dual Write Problem in AI Feature Pipelines

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** August 09, 2026  
**Topic:** Dual Write Problem, Consistency, Transactional Outbox, Change Data Capture

---

## The Problem

You update a feature in your database. You also need to publish an event to Kafka. What if the database succeeds but Kafka fails? Or vice versa? Your system is now inconsistent. This is the dual write problem—and it's silent and expensive.

## Code Example

### ❌ Naive Dual Write - Inconsistency Risk

```java
// DANGEROUS: Two Separate Writes
@Service
public class FeatureUpdateService {
    
    @Autowired
    private Database database;
    
    @Autowired
    private KafkaTemplate<String, FeatureEvent> kafkaTemplate;
    
    public void updateUserFeature(String userId, int purchaseCount) {
        // Write 1: Update Database
        database.updateFeature(userId, "purchases_30d", purchaseCount);
        // ✓ Database Updated
        
        // Write 2: Publish Event to Kafka
        try {
            kafkaTemplate.send("feature-updates", userId, 
                new FeatureEvent(userId, "purchases_30d", purchaseCount));
        } catch (Exception e) {
            // ✗ Kafka Failed!
            // Database is Updated, but Kafka Never Got the Event
            logger.error("Kafka publish failed", e);
        }
    }
}

// Problem Scenario:
// 1. Database Successfully Updated: purchases_30d = 15
// 2. Kafka Send Fails (Network Issue, Full Broker)
// 3. Downstream Consumer Never Sees the Update
// 4. Feature Store Cache Out of Sync
// 5. Inconsistency: Silent, Undetected!
```

### ✅ Solution 1: Transactional Outbox Pattern

```java
/*
OUTBOX PATTERN:
  1. Write to Database (Business Entity) + Outbox Table (Single Transaction)
  2. Separate Process Polls Outbox Table
  3. Publishes Events to Kafka
  4. Marks as Published in Outbox
  
Guarantees: Either both succeed or both fail (Database Transaction)
*/

// Define Outbox Table Entity
@Entity
@Table(name = "feature_updates_outbox")
public class FeatureUpdateOutbox {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String userId;
    
    @Column(nullable = false)
    private String featureName;
    
    @Column(nullable = false)
    private Integer featureValue;
    
    @Column(nullable = false)
    private LocalDateTime createdAt;
    
    @Column
    private LocalDateTime publishedAt;  // NULL = Not Yet Published
    
    @Column
    private Boolean isPublished = false;
}

// Write to Database and Outbox Atomically
@Service
@Transactional  // Single Database Transaction!
public class OutboxFeatureUpdate {
    
    @Autowired
    private FeatureRepository featureRepository;
    
    @Autowired
    private FeatureUpdateOutboxRepository outboxRepository;
    
    public void updateUserFeature(String userId, int purchaseCount) {
        // Step 1: Update Feature in Database
        Feature feature = featureRepository.findById(userId).orElseThrow();
        feature.setPurchases30d(purchaseCount);
        featureRepository.save(feature);  // Persisted in Same Transaction
        
        // Step 2: Write to Outbox Table (Same Transaction!)
        FeatureUpdateOutbox outboxEntry = new FeatureUpdateOutbox();
        outboxEntry.setUserId(userId);
        outboxEntry.setFeatureName("purchases_30d");
        outboxEntry.setFeatureValue(purchaseCount);
        outboxEntry.setCreatedAt(LocalDateTime.now());
        
        outboxRepository.save(outboxEntry);  // Persisted in Same Transaction
        
        // Either Both Succeed or Both Fail
        // No Partial Success!
    }
}

// Separate Service: Outbox Relay (Publishes Events)
@Service
public class OutboxRelay {
    
    @Autowired
    private FeatureUpdateOutboxRepository outboxRepository;
    
    @Autowired
    private KafkaTemplate<String, FeatureEvent> kafkaTemplate;
    
    @Scheduled(fixedDelay = 5000)  // Poll Every 5 Seconds
    @Transactional
    public void relayOutboxEvents() {
        // Find Unpublished Events
        List<FeatureUpdateOutbox> unpublished = outboxRepository
            .findByIsPublishedFalse();
        
        for (FeatureUpdateOutbox entry : unpublished) {
            try {
                // Publish to Kafka
                kafkaTemplate.send("feature-updates", entry.getUserId(),
                    new FeatureEvent(
                        entry.getUserId(),
                        entry.getFeatureName(),
                        entry.getFeatureValue()
                    )).get();  // Wait for Confirmation
                
                // Mark as Published (Same Transaction)
                entry.setPublishedAt(LocalDateTime.now());
                entry.setIsPublished(true);
                outboxRepository.save(entry);
                
                logger.info("Published outbox entry: {}", entry.getId());
            } catch (Exception e) {
                // Don't Mark as Published - Will Retry Next Poll
                logger.error("Failed to publish event", e);
            }
        }
    }
}

// Benefits:
// ✅ Database and Kafka Stay Consistent
// ✅ No Data Loss
// ✅ Leverages Database Transactions
// ✅ At-Least-Once Delivery Guarantee
// Costs:
// ❌ Additional Outbox Table Maintenance
// ❌ Polling Latency (Seconds)
```

### ✅ Solution 2: Change Data Capture (CDC)

```java
/*
CDC PATTERN (Using Debezium):
  1. Read Database Transaction Log
  2. Capture Row Changes (Insert/Update/Delete)
  3. Stream to Kafka in Real-Time
  4. No Application Code Changes Needed
  
Guarantees: Captures Every Change, No Polling
*/

// Debezium Configuration (No Java Code)
// application.properties
debezium.connector.postgres.plugin=pgoutput
debezium.connector.postgres.slot.name=feature_updates_slot
debezium.connector.postgres.slot.drop.on.stop=false

// Java Consumer: Listen to CDC Events
@Component
public class CDCFeatureListener {
    
    @KafkaListener(topics = "cdc.features.feature_updates", 
        groupId = "cdc-consumer")
    public void processCDCEvent(String eventJson) throws JsonProcessingException {
        // Parse CDC Event from Debezium
        ObjectMapper mapper = new ObjectMapper();
        Map<String, Object> event = mapper.readValue(eventJson, Map.class);
        
        String operation = (String) event.get("op");  // INSERT, UPDATE, DELETE
        Map<String, Object> after = (Map<String, Object>) event.get("after");
        
        if ("u".equals(operation)) {  // UPDATE
            String userId = (String) after.get("user_id");
            Integer purchaseCount = (Integer) after.get("purchases_30d");
            
            // React to Change
            logger.info("Feature updated via CDC: {} -> {}", userId, purchaseCount);
            featureStore.invalidateCache(userId);
        }
    }
}

// Benefits:
// ✅ No Application Changes
// ✅ Real-Time CDC (Milliseconds)
// ✅ Captures All Changes
// ✅ Schema-Aware
// Costs:
// ❌ Additional Infrastructure (Debezium)
// ❌ Operational Complexity
// ❌ Database Overhead
```

### ✅ Solution 3: Event Sourcing

```java
/*
EVENT SOURCING PATTERN:
  1. Store Events (Not State)
  2. Derive Current State from Event Log
  3. Events Published to Kafka Automatically
  
Guarantees: Single Source of Truth = Event Log
*/

@Entity
@Table(name = "feature_events")
public class FeatureEvent {
    
    @Id
    @GeneratedValue
    private Long eventId;
    
    @Column(nullable = false)
    private String userId;
    
    @Column(nullable = false)
    private String eventType;  // FEATURE_UPDATED, FEATURE_CREATED
    
    @Column(nullable = false)
    @Convert(converter = JsonConverter.class)
    private Map<String, Object> eventData;
    
    @Column(nullable = false)
    private LocalDateTime timestamp;
    
    @Column(nullable = false)
    private Long aggregateVersion;  // For Versioning
}

@Service
public class EventSourcedFeatureStore {
    
    @Autowired
    private FeatureEventRepository eventRepository;
    
    @Autowired
    private KafkaTemplate<String, FeatureEvent> kafkaTemplate;
    
    @Transactional
    public void updateFeature(String userId, int purchaseCount) {
        // Step 1: Create Event
        FeatureEvent event = new FeatureEvent();
        event.setUserId(userId);
        event.setEventType("FEATURE_UPDATED");
        event.setEventData(Map.of("feature", "purchases_30d", "value", purchaseCount));
        event.setTimestamp(LocalDateTime.now());
        event.setAggregateVersion(getNextVersion(userId));
        
        // Step 2: Persist Event (Single Write!)
        eventRepository.save(event);
        
        // Step 3: Publish to Kafka (Via Transaction Listener)
        publishEventToKafka(event);
    }
    
    // Reconstruct State from Events
    public Map<String, Integer> getCurrentFeatures(String userId) {
        List<FeatureEvent> events = eventRepository
            .findByUserIdOrderByTimestamp(userId);
        
        Map<String, Integer> features = new HashMap<>();
        for (FeatureEvent event : events) {
            if ("FEATURE_UPDATED".equals(event.getEventType())) {
                features.put(
                    (String) event.getEventData().get("feature"),
                    (Integer) event.getEventData().get("value")
                );
            }
        }
        
        return features;
    }
}

// Benefits:
// ✅ True Dual Consistency (Events = Source of Truth)
// ✅ Audit Trail Built-In
// ✅ Time Travel (Replay History)
// Costs:
// ❌ Complex Event Handling
// ❌ Storage Growth (Full Event Log)
// ❌ Operational Complexity
```

### ✅ Real-World Example - Feature Pipeline

```java
@Service
public class ProductionFeatureUpdate {
    
    @Autowired
    private FeatureUpdateOutboxRepository outboxRepository;
    
    @Autowired
    private FeatureRepository featureRepository;
    
    @Transactional
    public void processFeatureUpdate(String userId, FeatureUpdate update) {
        // Atomic: Database + Outbox (Single Transaction)
        Feature feature = featureRepository.findById(userId).orElseThrow();
        
        // Update Feature
        feature.updateFromEvent(update);
        featureRepository.save(feature);
        
        // Write to Outbox
        FeatureUpdateOutbox outbox = new FeatureUpdateOutbox();
        outbox.setUserId(userId);
        outbox.setFeatureName(update.featureName());
        outbox.setFeatureValue(update.value());
        outbox.setCreatedAt(LocalDateTime.now());
        
        outboxRepository.save(outbox);
        
        // Either Both Succeed or Both Fail!
    }
}
```

## 💡 Why This Matters

The dual-write problem exists in distributed systems where atomic updates across multiple systems like databases and messaging systems are challenging, leading to potential inconsistencies. The transactional outbox pattern ensures atomicity by introducing an intermediary outbox table in the database. Without proper handling, data inconsistencies accumulate silently. Feature stores depend on consistency—one missed update breaks downstream models.

## 🎯 Key Takeaway

Never write to database and Kafka separately. Use Transactional Outbox for production simplicity. Use CDC when you can't modify application code. Use Event Sourcing when you need audit trails and time-travel. The dual write problem is architectural—solve it by design.

---

**Tags:** `#Java` `#JavaWisdom` `#DualWriteProblem` `#Consistency` `#DistributedSystems` `#Kafka` `#OutboxPattern` `#CDC` `#EventSourcing` `#SpringBoot` `#Architecture` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity` `#SoftwareEngineering`
