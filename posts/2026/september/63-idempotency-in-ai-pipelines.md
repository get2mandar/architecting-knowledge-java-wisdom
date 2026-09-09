# Post #63: Idempotency in AI Pipelines - Designing for Safe Retries
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** September 9, 2026  
**Topic:** Idempotency, Safe Retries, Feature Pipelines, Replay Safety
 
---
 
## The Problem
 
Your feature pipeline computes user's lifetime purchase amount. Halfway through 50M records, it crashes. You restart. Does it recompute all 50M (wasting resources)? Continue from checkpoint (risky)? With proper idempotency, restart safely without duplicating work or corrupting results.
 
## Code Example
 
### ❌ Non-Idempotent Operations - Accumulation Trap
 
```java
// Accumulating Updates = Non-Idempotent
@Service
public class NonIdempotentFeatures {
    
    @Scheduled(cron = "0 0 * * *")  // Daily
    public void computeUserFeatures() {
        List<User> users = userRepository.findAll();
        
        for (User user : users) {
            // Read Feature (Stale)
            Feature feature = featureRepository.findById(user.getId())
                .orElse(new Feature(user.getId()));
            
            // Compute Increment
            int purchasesToday = purchaseRepository
                .countByUserIdAndDateToday(user.getId());
            
            // Accumulate (NOT IDEMPOTENT!)
            feature.setTotalPurchases(
                feature.getTotalPurchases() + purchasesToday
            );
            
            featureRepository.save(feature);
        }
    }
    
    // Failure Scenario:
    // 1. Process 25M Users (Total Purchases = 100)
    // 2. Crash
    // 3. Restart
    // 4. Reprocess All 50M Users
    // 5. First 25M: Feature Incremented AGAIN
    // 6. Result: Total Purchases = 200 (Should Be 100)
    // 7. Features Corrupted!
}
 
// Why Non-Idempotent?
// Operation: purchase_count = purchase_count + today_purchases
// Not Idempotent: Depends on Previous State
// Rerun: Different Result
```
 
### ✅ Solution 1: Partition Overwrite - Idempotent By Design
 
```java
/*
PARTITION OVERWRITE STRATEGY:
  - Delete Existing Partition
  - Recompute Fresh Data
  - Write Back (Atomic)
  - Running Twice = Same Result!
*/
 
@Service
public class IdempotentPartitionOverwrite {
    
    @Scheduled(cron = "0 0 * * *")  // Daily
    public void computeUserFeaturesIdempotent() {
        LocalDate today = LocalDate.now();
        
        // Step 1: Delete Today's Partition (Safe, Atomic Operation)
        featureRepository.deleteByComputedDate(today);
        
        // Step 2: Recompute All Features Fresh
        // (No Dependency on Previous State)
        List<User> users = userRepository.findAll();
        
        for (User user : users) {
            Feature feature = new Feature(user.getId());
            
            // Compute from Scratch (Not Increment!)
            int totalPurchases = purchaseRepository
                .countByUserId(user.getId());
            
            double avgOrderValue = purchaseRepository
                .avgAmountByUserId(user.getId());
            
            int daysActive = userRepository
                .daysSinceSignup(user.getId());
            
            // Set Fresh Values (Not Add)
            feature.setTotalPurchases(totalPurchases);
            feature.setAvgOrderValue(avgOrderValue);
            feature.setDaysActive(daysActive);
            feature.setComputedDate(today);
            
            featureRepository.save(feature);
        }
        
        // Step 3: Log Completion
        logger.info("Feature Computation Complete for {}", today);
    }
    
    // Idempotency Guarantee:
    // - Run 1: Delete Partition → Compute Fresh → Write
    // - Run 2 (Restart): Delete Partition (Already Empty) → Compute Fresh → Write
    // = IDENTICAL RESULT
    
    // Failure Scenarios:
    // 1. Crash During Computation → Restart: Partition Empty, Recompute
    // 2. Crash During Write → Restart: Recompute (Atomic)
    // 3. Partial Write → Restart: Delete + Recompute (Idempotent)
}
 
// Implementation Note: Use Atomic REPLACE PARTITION
// SQL: REPLACE INTO feature_table PARTITION (computed_date='2026-09-08')
// OR:  DELETE FROM features WHERE computed_date='2026-09-08';
//      INSERT INTO features SELECT ... WHERE computed_date='2026-09-08';
```
 
### ✅ Solution 2: Idempotency Keys - Track Processed Operations
 
```java
@Service
public class IdempotencyKeyTracking {
    
    public record FeatureOperation(
        String idempotencyKey,       // Unique Per Operation
        String userId,
        String operationType,        // PURCHASE, RETURN, REFUND
        Map<String, Object> data,
        LocalDateTime timestamp
    ) {}
    
    @Entity
    @Table(name = "feature_operations")
    public static class ProcessedOperation {
        @Id
        @Column(unique = true)
        private String idempotencyKey;
        
        private String userId;
        private LocalDateTime processedAt;
        private String resultJson;  // Cache Result
    }
    
    @Transactional
    public void processFeatureOperation(FeatureOperation operation) {
        // Step 1: Check If Already Processed
        Optional<ProcessedOperation> existing = 
            operationRepository.findById(operation.idempotencyKey);
        
        if (existing.isPresent()) {
            logger.info("Operation Already Processed: {}", 
                operation.idempotencyKey);
            // Replay Cached Result
            String cachedResult = existing.get().getResultJson();
            publishResult(cachedResult);
            return;  // Idempotent!
        }
        
        // Step 2: Process Operation (First Time)
        String result = computeFeatureUpdate(operation);
        
        // Step 3: Save Result (Atomic with Feature Update)
        Feature feature = featureRepository.findById(operation.userId)
            .orElse(new Feature(operation.userId));
        
        applyFeatureUpdate(feature, operation);
        featureRepository.save(feature);
        
        // Step 4: Log Processed (Same Transaction)
        ProcessedOperation processed = new ProcessedOperation();
        processed.setIdempotencyKey(operation.idempotencyKey);
        processed.setUserId(operation.userId);
        processed.setProcessedAt(LocalDateTime.now());
        processed.setResultJson(result);
        
        operationRepository.save(processed);
        
        // Publish Result
        publishResult(result);
    }
    
    private String computeFeatureUpdate(FeatureOperation operation) {
        return switch (operation.operationType()) {
            case "PURCHASE" -> {
                int amount = (int) operation.data().get("amount");
                yield "purchase_added:" + amount;
            }
            case "RETURN" -> {
                int amount = (int) operation.data().get("amount");
                yield "purchase_removed:" + amount;
            }
            default -> "unknown";
        };
    }
    
    private void applyFeatureUpdate(Feature feature, FeatureOperation op) {
        switch (op.operationType()) {
            case "PURCHASE" -> {
                int amount = (int) op.data().get("amount");
                feature.setPurchaseCount(feature.getPurchaseCount() + 1);
                feature.setTotalSpent(feature.getTotalSpent() + amount);
            }
            case "RETURN" -> {
                int amount = (int) op.data().get("amount");
                feature.setTotalSpent(feature.getTotalSpent() - amount);
            }
        }
    }
}
 
// Idempotency Key Generation:
// Option 1: Use Event ID (Already Unique)
// idempotencyKey = event.eventId
//
// Option 2: Generate from Operation Details
// idempotencyKey = SHA256("purchase:" + userId + ":" + timestamp + ":" + amount)
//
// Option 3: Request ID (API Context)
// idempotencyKey = request.getHeader("Idempotency-Key")
```
 
### ✅ Solution 3: Checkpointing - Resume From State
 
```java
@Service
public class CheckpointedBatchProcessing {
    
    @Entity
    @Table(name = "processing_checkpoints")
    public static class Checkpoint {
        @Id
        private String batchId;
        
        private Integer lastProcessedIndex;
        private Long totalRecords;
        private LocalDateTime createdAt;
        private LocalDateTime updatedAt;
        private Boolean completed;
    }
    
    @Transactional
    public void processBatchIdempotent(String batchId, List<User> allUsers) {
        // Step 1: Load or Create Checkpoint
        Checkpoint checkpoint = checkpointRepository
            .findById(batchId)
            .orElse(new Checkpoint());
        
        checkpoint.setBatchId(batchId);
        checkpoint.setTotalRecords((long) allUsers.size());
        checkpoint.setCreatedAt(LocalDateTime.now());
        
        int startIndex = checkpoint.getLastProcessedIndex() == null ? 
            0 : checkpoint.getLastProcessedIndex() + 1;
        
        logger.info("Resuming Batch {} from Index {}", batchId, startIndex);
        
        // Step 2: Process from Checkpoint
        for (int i = startIndex; i < allUsers.size(); i++) {
            User user = allUsers.get(i);
            
            try {
                // Compute Features
                Feature feature = computeFeatureIdempotent(user);
                featureRepository.save(feature);
                
                // Update Checkpoint After Each Record (For Recovery)
                checkpoint.setLastProcessedIndex(i);
                checkpoint.setUpdatedAt(LocalDateTime.now());
                checkpointRepository.save(checkpoint);
                
                if ((i + 1) % 10000 == 0) {
                    logger.info("Progress: {}/{}", i + 1, allUsers.size());
                }
            } catch (Exception e) {
                logger.error("Error Processing User {}: {}", user.getId(), e);
                // Don't Update Checkpoint - Will Retry This Record
                throw e;
            }
        }
        
        // Step 3: Mark Complete
        checkpoint.setCompleted(true);
        checkpoint.setUpdatedAt(LocalDateTime.now());
        checkpointRepository.save(checkpoint);
        
        logger.info("Batch {} Completed", batchId);
    }
    
    private Feature computeFeatureIdempotent(User user) {
        // Compute Fresh (Not Accumulate)
        Feature feature = new Feature(user.getId());
        
        int purchases = purchaseRepository.countByUserId(user.getId());
        double avgValue = purchaseRepository.avgByUserId(user.getId());
        
        feature.setPurchaseCount(purchases);
        feature.setAvgValue(avgValue);
        
        return feature;
    }
}
 
// Failure Recovery:
// 1. Process 25M Users
// 2. Crash at User 25M000001
// 3. Restart: Load Checkpoint
// 4. lastProcessedIndex = 25000000
// 5. Resume from 25000001
// 6. No Duplicate Processing (Already Saved)
// 7. No Lost Work (Checkpoints Safe)
```
 
### ✅ Real-World Example - Daily Feature Computation
 
```java
@Service
public class DailyFeatureComputationPipeline {
    
    @Scheduled(cron = "0 0 * * *")  // Daily at Midnight
    public void computeDailyFeatures() {
        LocalDate today = LocalDate.now();
        String batchId = "daily-features-" + today;
        
        // Idempotent Batch Processing
        List<User> allUsers = userRepository.findAll();
        
        // Pattern 1: Partition Overwrite
        // Delete today's partition, recompute fresh
        deletePartition(today);
        
        // Pattern 2: Checkpointing
        // Resume from last checkpoint if crashed
        processBatchWithCheckpoints(batchId, allUsers);
        
        // Pattern 3: Idempotency Keys
        // Track each operation, replay if needed
        for (User user : allUsers) {
            String idempotencyKey = "feature:" + user.getId() + ":" + today;
            
            processWithIdempotencyKey(idempotencyKey, user);
        }
    }
    
    private void deletePartition(LocalDate date) {
        // DELETE FROM features WHERE computed_date = ?
        featureRepository.deleteByComputedDate(date);
    }
    
    private void processBatchWithCheckpoints(String batchId, List<User> users) {
        // Load/Create Checkpoint, Resume from Last Position
        // See CheckpointedBatchProcessing
    }
    
    private void processWithIdempotencyKey(String key, User user) {
        // Track Operation, Allow Replay
        // See IdempotencyKeyTracking
    }
}
 
// Production Guarantees:
// - Restart Mid-Process? → Resume from Checkpoint
// - Query Yesterday's Features? → Partition Overwrite (Fresh)
// - Duplicate API Call? → Idempotency Key (Replay)
// = Feature Pipeline RESILIENT TO FAILURE
```
 
## 💡 Why This Matters
 
Idempotent operations ensure that running them multiple times produces the same result as running once—critical in distributed systems where retries are inevitable. Partition overwrite patterns delete and recompute entire partitions atomically, making the operation naturally idempotent. Idempotency keys track processed operations and replay results on retry. Checkpointing allows batch jobs to resume from failure without reprocessing completed work. Together, these patterns make ML pipelines resilient to failure without losing data or corrupting features.
 
## 🎯 Key Takeaway
 
Design ML pipelines to be idempotent: compute fresh rather than accumulate. Use partition overwrite for batch jobs. Implement idempotency keys for streaming operations. Add checkpointing for resumable batch processing. A safe pipeline can be rerun 10 times and produce the same result—that's idempotency.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#Idempotency` `#Pipelines` `#DataEngineering` `#BatchProcessing` `#Reliability` `#SafeRetries` `#MLOps` `#SpringBoot` `#DistributedSystems` `#Checkpointing` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
