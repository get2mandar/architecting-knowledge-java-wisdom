# Post #69: Model Versioning in Production - Blue-Green and Canary Deployments
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** September 30, 2026  
**Topic:** Model Versioning, Blue-Green Deployment, Canary Releases, Rollback Strategy  
 
---
 
## The Problem
 
Your fraud model team ships v2.3. It replaces v2.2 in one deployment: no comparison, no rollback path. Three hours later, v2.3 flags 40% of legitimate transactions as fraud. Support tickets flood in before anyone notices the model regression. Rolling back means redeploying the entire service, losing another 20 minutes. Overwriting a model in place is an architectural decision, not just a deployment step, and it removes your only safety net.
 
## Code Example
 
### ❌ Without Versioning - Overwrite in Place
 
```java
// Single Model Reference, No Rollback Path
@Service
public class UnversionedModelService {
 
    private FraudModel currentModel;
 
    public void deployModel(FraudModel newModel) {
        // Replaces Old Model Immediately
        this.currentModel = newModel;
        logger.info("Model Deployed");
    }
 
    public FraudScore predict(Transaction tx) {
        return currentModel.score(tx);  // No Idea Which Version Is Live
    }
}
 
// Failure Scenario:
// 1. v2.3 Deployed, Overwrites v2.2 Reference
// 2. v2.3 Has a Training Data Bug
// 3. False Positive Rate Jumps from 2% to 40%
// 4. No Record of Previous Model State
// 5. Rollback = Full Redeploy of v2.2 Artifact
// 6. 20+ Minutes of Bad Predictions in Production
```
 
### ✅ Solution 1: Blue-Green Deployment - Instant Rollback
 
```java
/*
BLUE-GREEN MODEL DEPLOYMENT:
  - Blue: Currently Serving Version
  - Green: New Version, Deployed but Not Yet Live
  - Switch: Atomic Traffic Cutover Between Them
*/
@Service
public class BlueGreenModelService {
 
    private final AtomicReference<FraudModel> activeModel = new AtomicReference<>();
    private FraudModel standbyModel;
 
    public void stageNewVersion(FraudModel candidate) {
        this.standbyModel = candidate;  // Green: Loaded, Not Serving
    }
 
    public void promoteStandby() {
        // Atomic Swap: Green Becomes Blue
        activeModel.set(standbyModel);
        logger.info("Promoted Model Version: {}", standbyModel.getVersion());
    }
 
    public void rollback(FraudModel previousModel) {
        // Instant Revert: No Redeploy Needed
        activeModel.set(previousModel);
        logger.warn("Rolled Back to Version: {}", previousModel.getVersion());
    }
 
    public FraudScore predict(Transaction tx) {
        return activeModel.get().score(tx);
    }
}
```
 
### ✅ Solution 2: Canary Release - Gradual Traffic Shift
 
```java
@Service
public class CanaryModelService {
 
    private final FraudModel stableModel;
    private final FraudModel canaryModel;
    private volatile int canaryPercentage = 5;  // Start Small
 
    public FraudScore predict(Transaction tx) {
        boolean useCanary = ThreadLocalRandom.current().nextInt(100) < canaryPercentage;
 
        FraudModel modelToUse = useCanary ? canaryModel : stableModel;
        FraudScore score = modelToUse.score(tx);
 
        // Tag Result for Comparison Dashboards
        metrics.recordPrediction(modelToUse.getVersion(), score);
        return score;
    }
 
    public void increaseCanaryTraffic(int newPercentage) {
        // Ramp: 5% -> 25% -> 50% -> 100%, Watched at Each Stage
        this.canaryPercentage = newPercentage;
        logger.info("Canary Traffic Now at {}%", newPercentage);
    }
}
 
// Ramp Schedule (Watched Against Error Budget):
// 5%  -> 30 Minutes -> Compare Precision/Recall
// 25% -> 1 Hour      -> Compare Latency, False Positive Rate
// 50% -> 2 Hours     -> Compare Business Metrics
// 100% -> Full Cutover, Old Version Kept Warm for Rollback
```
 
### ✅ Solution 3: Shadow Traffic - Zero-Risk Validation
 
```java
@Service
public class ShadowDeploymentService {
 
    private final FraudModel productionModel;
    private final FraudModel candidateModel;
 
    public FraudScore predict(Transaction tx) {
        // Production Result Is What Callers Receive
        FraudScore liveScore = productionModel.score(tx);
 
        // Candidate Runs Async, Result Never Returned to Caller
        CompletableFuture.runAsync(() -> {
            FraudScore shadowScore = candidateModel.score(tx);
            comparisonLog.record(tx.getId(), liveScore, shadowScore);
        });
 
        return liveScore;  // Caller Never Sees the Candidate
    }
}
 
// Value: Candidate Sees Real Traffic and Real Data Patterns
// Zero Impact on Users - Purely Observational Validation
```
 
## Why This Matters
 
Model versioning is what separates a reversible deployment from an incident. Blue-green gives you an atomic, instant rollback because the previous model version never disappears it just stops receiving traffic. Canary releases limit the blast radius of a bad model to a small traffic percentage, giving you comparison data before full commitment. Shadow traffic goes further: the candidate model runs against production data with zero user impact, so regressions surface before a single real prediction depends on it. None of these patterns are about the model itself they're about how much damage a bad model can do before someone notices.
 
## Key Takeaway
 
Never overwrite a production model in place. Keep the previous version addressable for instant rollback. Use canary releases to shift traffic gradually and compare metrics at each stage. Use shadow traffic to validate a candidate against real data before it ever serves a real prediction. Version everything the model artifact, the traffic split, and the rollback path.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#ModelVersioning` `#BlueGreenDeployment` `#CanaryRelease` `#MLOps` `#SpringBoot` `#Resilience` `#DistributedSystems` `#Microservices` `#Reliability` `#Observability` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
