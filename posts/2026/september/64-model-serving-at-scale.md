# Post #64: Model Serving at Scale - Inference Latency Under Load

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** September 13, 2026  
**Topic:** Model Serving, Batching, Latency, Throughput, GPU Utilization

---

## The Problem

Your fraud detection model runs on one GPU. Serving 1000 requests per second. One request at a time: P99 latency = 2 seconds. GPU sits idle 90% of the time. Add batching: Process 32 requests together in same 100ms. Same GPU, 30x throughput, lower latency. This is inference at scale.

## Code Example

### ❌ Single-Request Inference - GPU Underutilization

```java
// One Request At A Time
@RestController
public class NaiveFraudDetection {
    
    @Autowired
    private ONNXModel fraudModel;  // Runs on GPU
    
    @PostMapping("/predict/fraud")
    public ResponseEntity<FraudScore> predictFraud(@RequestBody Transaction tx) {
        // Single Request Inference
        float[] features = extractFeatures(tx);
        float fraudScore = fraudModel.predict(features);
        
        return ResponseEntity.ok(new FraudScore(fraudScore));
    }
}

// Performance Characteristics:
// P50 Latency: 100ms (Model Inference: 50ms + Overhead: 50ms)
// P99 Latency: 2000ms (Queuing: 1950ms)
// GPU Utilization: ~10% (Most Time: Waiting for Requests)
// Throughput: ~10 Requests/Second

// Problem:
// - Model Execution: 50ms (Optimal)
// - GPU Overhead: Huge (Kernel Launch, Memory Transfer)
// - Queuing: Brutal (Each Request Waits for Previous)
// - Throughput: Terrible (~10 RPS)
//
// GPU Can Process 32 Transactions in Same Time as 1!
```

### ✅ Solution 1: Dynamic Batching - Accumulate and Process

```java
/*
DYNAMIC BATCHING:
  - Accumulate Requests into Batch
  - Wait Until: Max Batch Size OR Timeout
  - Process Entire Batch on GPU (Efficient!)
  - Return Results to All Requests
*/

@Service
public class DynamicBatchingPredictor {
    
    private final int MAX_BATCH_SIZE = 32;
    private final long BATCH_TIMEOUT_MS = 10;
    
    private final Queue<PredictionRequest> requestQueue = new ConcurrentLinkedQueue<>();
    private final BlockingQueue<PredictionResult> resultQueue = new LinkedBlockingQueue<>();
    private final ExecutorService batchProcessor = Executors.newSingleThreadExecutor();
    
    public record PredictionRequest(
        String requestId,
        float[] features,
        CompletableFuture<Float> resultFuture
    ) {}
    
    public record PredictionResult(
        String requestId,
        float score
    ) {}
    
    public DynamicBatchingPredictor(ONNXModel model) {
        // Start Batch Processor Thread
        batchProcessor.submit(() -> processBatchesInLoop(model));
    }
    
    public CompletableFuture<Float> predict(float[] features) {
        String requestId = UUID.randomUUID().toString();
        CompletableFuture<Float> resultFuture = new CompletableFuture<>();
        
        // Add to Queue
        requestQueue.add(new PredictionRequest(requestId, features, resultFuture));
        
        // Return Immediately (Latency = Queueing Only!)
        return resultFuture;
    }
    
    private void processBatchesInLoop(ONNXModel model) {
        long batchStartTime;
        
        while (true) {
            batchStartTime = System.currentTimeMillis();
            List<PredictionRequest> batch = new ArrayList<>();
            
            // Accumulate Requests: Max Size OR Timeout
            PredictionRequest first = null;
            try {
                first = requestQueue.poll(BATCH_TIMEOUT_MS, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                continue;
            }
            
            if (first == null) continue;
            
            batch.add(first);
            
            // Fill Batch to Max Size (Or Until Timeout)
            while (batch.size() < MAX_BATCH_SIZE && 
                   System.currentTimeMillis() - batchStartTime < BATCH_TIMEOUT_MS) {
                PredictionRequest req = requestQueue.poll();
                if (req != null) {
                    batch.add(req);
                }
            }
            
            // Process Entire Batch on GPU (Single Kernel Launch!)
            float[][] features = batch.stream()
                .map(r -> r.features)
                .toArray(float[][]::new);
            
            float[] scores = model.predictBatch(features);  // GPU Loves This!
            
            // Return Results to All Requests
            for (int i = 0; i < batch.size(); i++) {
                batch.get(i).resultFuture.complete(scores[i]);
            }
            
            logger.info("Processed Batch Size: {}", batch.size());
        }
    }
}

// Performance with Dynamic Batching:
// Batch Size: 32
// GPU Execution: 50ms (Model) + 5ms (Batch Overhead) = 55ms
// Throughput: 32 Requests / 55ms = ~580 RPS (50x Improvement!)
// P50 Latency: 5ms (Early Batch) + 55ms (GPU) = 60ms
// P99 Latency: 10ms (Timeout) + 55ms (GPU) = 65ms
//
// Compare to Single-Request:
// Single: P99 = 2000ms, Throughput = 10 RPS
// Batch:  P99 = 65ms, Throughput = 580 RPS
```

### ✅ Solution 2: Request Coalescence - Deduplication

```java
@Service
public class RequestCoalescingPredictor {
    
    // Cache In-Flight Predictions (Same Features = Reuse Result)
    private final ConcurrentHashMap<String, CompletableFuture<Float>> inflightRequests 
        = new ConcurrentHashMap<>();
    
    public CompletableFuture<Float> predictWithCoalescing(float[] features) {
        // Generate Hash of Features (Stable)
        String featureHash = hashFeatures(features);
        
        // Check If Already Computing
        if (inflightRequests.containsKey(featureHash)) {
            logger.info("Request Coalesced: {}", featureHash);
            return inflightRequests.get(featureHash);  // Return Existing Future
        }
        
        // New Request: Start Computation
        CompletableFuture<Float> resultFuture = new CompletableFuture<>();
        inflightRequests.put(featureHash, resultFuture);
        
        // Predict (From Batch Processor)
        predict(features).thenAccept(score -> {
            resultFuture.complete(score);
            inflightRequests.remove(featureHash);  // Clean Up
        });
        
        return resultFuture;
    }
    
    private String hashFeatures(float[] features) {
        return Arrays.hashCode(features) + "";
    }
}

// Scenario:
// Request A: Predict User 123's Features
// Request B: (100ms Later) Predict User 123's Features (Same!)
// Request A: Queued, Waiting for Batch
// Request B: Arrives, Detects A Is Computing Same
// Result: Both Get Same Answer, GPU Runs Once
//
// Benefit: Duplicate Requests Free (No GPU Cost)
```

### ✅ Solution 3: Multi-Model Serving - GPU Sharing

```java
@Service
public class MultiModelServer {
    
    public enum ModelType {
        FRAUD_DETECTOR_V1,
        FRAUD_DETECTOR_V2,
        CHURN_PREDICTOR,
        RECOMMENDATION_ENGINE
    }
    
    private final Map<ModelType, ONNXModel> modelCache = new ConcurrentHashMap<>();
    private final Map<ModelType, DynamicBatchingPredictor> batchProcessors = 
        new ConcurrentHashMap<>();
    
    public MultiModelServer() {
        // Load All Models on Startup
        modelCache.put(ModelType.FRAUD_DETECTOR_V1, 
            loadModel("fraud-v1.onnx"));  // 500MB
        modelCache.put(ModelType.FRAUD_DETECTOR_V2, 
            loadModel("fraud-v2.onnx"));  // 500MB
        modelCache.put(ModelType.CHURN_PREDICTOR, 
            loadModel("churn.onnx"));     // 300MB
        
        // GPU Memory: 8GB Total
        // Loaded Models: 1.3GB (Plenty of Space!)
        // Available for Batch Processing: 6.7GB
        
        // Create Batch Processors
        for (ModelType type : modelCache.keySet()) {
            batchProcessors.put(type, 
                new DynamicBatchingPredictor(modelCache.get(type)));
        }
    }
    
    public CompletableFuture<Float> predict(
        ModelType model,
        float[] features
    ) {
        return batchProcessors.get(model).predict(features);
    }
    
    // GPU Utilization:
    // Single Model: Batch 32 Transactions = 580 RPS
    // Multiple Models: Route Requests to Appropriate Model
    // Total GPU Throughput: ~1500 RPS (Multiple Models Batching Independently)
    // GPU Utilization: 95%+ (All Models Sharing Batches)
}

// Scheduling Strategy:
// Fraud V1: Batch Size 32 (Timeout 10ms)
// Fraud V2: Batch Size 16 (Slower Model, Smaller Batch)
// Churn:    Batch Size 64 (Fast Model, Larger Batch)
//
// Independent Queues: Each Model Accumulates Requests
// Concurrent Batching: Multiple Batches Run Simultaneously
// GPU Multiplexing: Switch Between Models Efficiently
```

### ✅ Real-World Example - Production Fraud Pipeline

```java
@Service
public class ProductionFraudInferenceServer {
    
    @RestController
    public class FraudDetectionController {
        
        @Autowired
        private MultiModelServer modelServer;
        
        @PostMapping("/fraud/predict")
        public CompletableFuture<FraudPrediction> predictFraud(
            @RequestBody Transaction transaction
        ) {
            // Extract Features
            float[] features = featureService.extractFeatures(transaction);
            
            // Get Prediction (Batched!)
            return modelServer
                .predict(ModelType.FRAUD_DETECTOR_V2, features)
                .thenApply(score -> new FraudPrediction(
                    transaction.getId(),
                    score,
                    score > 0.7 ? "HIGH_RISK" : "LOW_RISK"
                ));
        }
    }
    
    // Production Metrics:
    // P50 Latency: 45ms (Batch Accumulation: 15ms + GPU: 30ms)
    // P95 Latency: 60ms (Near Max Timeout)
    // P99 Latency: 70ms (Max Timeout)
    // Throughput: 500+ RPS Per Model
    // GPU Utilization: 90%+
    // Cost Per Prediction: $0.0001 (Amortized GPU)
    
    // vs Single Request:
    // P50 Latency: 50ms (Overhead: 40ms + GPU: 10ms)
    // P99 Latency: 2000ms (Queuing!)
    // Throughput: 10 RPS
    // GPU Utilization: 10%
    // Cost Per Prediction: $0.01 (Wasted GPU)
    //
    // Batching Savings: 100x Cost Reduction
}

// Operational Insights:
// - Batch Size Tuning: Larger = Higher Throughput, Higher Latency
// - Timeout Tuning: Longer = Better Batches, Higher P99 Latency
// - Model Selection: Choose Model Based on Required Accuracy
// - Fallback: If V2 Busy, Route to V1 (Slightly Lower Accuracy)
```

## 💡 Why This Matters

Processing model inference one request at a time severely underutilizes GPUs because kernel launch overhead and memory transfer dominate single-request latency. Batching multiple requests allows the GPU to amortize overhead—32 requests processed in nearly the same time as 1. Dynamic batching accumulates requests up to a max size or timeout, balancing latency (lower timeout) with throughput (larger batch). Multi-model serving shares GPU memory across multiple models, achieving 90%+ utilization. Production systems can achieve 50x throughput improvement and 100x cost reduction through proper batching architecture.

## 🎯 Key Takeaway

Never serve ML models one request at a time. Implement dynamic batching: accumulate requests, process batches on GPU. Balance batch size (throughput) vs timeout (latency). Share GPUs across multiple models. Proper batching turns expensive GPUs into efficient workhorses: 500+ RPS per GPU, 90%+ utilization, sub-100ms latency.

---

**Tags:** `#Java` `#JavaWisdom` `#ModelServing` `#InferencePipeline` `#GPU` `#Batching` `#Latency` `#Throughput` `#MLOps` `#Performance` `#SpringBoot` `#DistributedSystems` `#Optimization` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
