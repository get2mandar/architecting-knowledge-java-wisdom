# Post #74: Streaming Feature Computation - Apache Kafka and Flink for ML
 
**Series:** Architecting Knowledge - Java Wisdom Series   
**Published:** October 18, 2026   
**Topic:** Stream Processing, Apache Kafka, Apache Flink, Windowed Aggregation   
 
---
 
## The Problem
 
Fraud detection needs "transactions in the last 5 minutes" as a feature. Computing that with a batch job means running a query every few minutes against a growing table—expensive, and always a few minutes stale by the time it's used. The feature that matters most for catching fraud in the act is exactly the one batch processing is worst at delivering: something fresh, continuously updated, computed as events actually arrive.
 
## Code Example
 
### ❌ Batch Polling - Stale by Design
 
```java
// Scheduled Job, Not a Real Feature Pipeline
@Scheduled(fixedRate = 60000)  // Every 60 Seconds
public class PollingFeatureJob {
 
    public void updateTransactionCounts() {
        // Full Table Scan, Every Minute, Growing More Expensive Over Time
        Map<String, Integer> counts = database.query(
            "SELECT user_id, COUNT(*) FROM transactions " +
            "WHERE timestamp > NOW() - INTERVAL 5 MINUTE GROUP BY user_id"
        );
        counts.forEach((userId, count) -> featureStore.write("txn_count_5m", userId, count));
    }
}
 
// Failure Scenario:
// 1. Fraud Ring Makes 20 Transactions in 90 Seconds
// 2. Batch Job Runs Every 60 Seconds - Feature Is Up to 59s Stale
// 3. By the Time the Count Updates, Damage Is Already Done
// 4. Query Cost Grows as Transaction Volume Grows
```
 
### ✅ Solution 1: Kafka as the Event Backbone
 
```java
/*
KAFKA CONSUMER: Reacts to Each Transaction Event as It Arrives
No Polling, No Fixed Interval - Driven by Actual Event Flow
*/
@KafkaListener(topics = "transaction-events", groupId = "feature-computation")
public void onTransactionEvent(TransactionEvent event) {
    // Fired the Moment the Event Lands, Not on a Timer
    featureUpdater.applyEvent(event);
}
 
@Service
public class FeatureUpdater {
    public void applyEvent(TransactionEvent event) {
        // Incrementally Update Rolling State Instead of Recomputing From Scratch
        windowedCounter.increment(event.getUserId(), event.getTimestamp());
    }
}
```
 
### ✅ Solution 2: Flink Windowed Aggregation - Correct, Continuous Windows
 
```java
/*
FLINK KEYED WINDOW:
  - Keyed By User ID
  - Tumbling or Sliding 5-Minute Window
  - Emits an Updated Count Continuously as Events Arrive
*/
DataStream<TransactionEvent> transactions = env
    .fromSource(kafkaSource, WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10)),
        "transaction-source");
 
DataStream<FeatureUpdate> transactionCounts = transactions
    .keyBy(TransactionEvent::getUserId)
    .window(SlidingEventTimeWindows.of(Time.minutes(5), Time.seconds(30)))
    .aggregate(new CountAggregator());
 
transactionCounts.addSink(new FeatureStoreSink("txn_count_5m"));
 
// Result: Feature Value Refreshes Every 30 Seconds,
// Always Reflecting a True Trailing 5-Minute Window
```
 
### ✅ Solution 3: Handling Late and Out-of-Order Events
 
```java
public class CountAggregator implements AggregateFunction<TransactionEvent, Long, Long> {
 
    @Override
    public Long createAccumulator() {
        return 0L;
    }
 
    @Override
    public Long add(TransactionEvent event, Long accumulator) {
        // Watermark Strategy (Set on the Stream) Already Handles
        // Events Arriving Up to 10 Seconds Out of Order
        return accumulator + 1;
    }
 
    @Override
    public Long getResult(Long accumulator) {
        return accumulator;
    }
 
    @Override
    public Long merge(Long a, Long b) {
        return a + b;
    }
}
 
// Late Events Beyond the Watermark Bound Are Routed to a Side Output,
// Logged for Investigation Rather Than Silently Dropped or Corrupting the Window
```
 
## Why This Matters
 
Batch polling forces a trade-off between freshness and cost: poll more often and the query gets more expensive, poll less often and the feature gets staler. Kafka removes the polling model entirely—computation is triggered by the event itself, so latency is bounded by processing time, not by a schedule. Flink's windowed aggregation handles the hard part correctly: maintaining a continuously updated 5-minute window without recomputing from scratch on every event, and doing so using event time rather than arrival time, which matters when events arrive out of order across a distributed system. Watermarks give you an explicit, tunable answer to "how late is too late," instead of silently getting wrong answers from unordered data.
 
## Key Takeaway
 
Time-sensitive features like rolling counts and rates belong in a streaming pipeline, not a scheduled batch job. Use Kafka to decouple event production from feature computation, triggered by arrival rather than a timer. Use Flink's windowing to maintain correct, continuously updated aggregates, and set an explicit watermark strategy so late-arriving events are handled deliberately, not accidentally.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#ApacheKafka` `#ApacheFlink` `#StreamProcessing` `#MLOps` `#FeatureStore` `#SpringBoot` `#DistributedSystems` `#Microservices` `#Reliability` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
