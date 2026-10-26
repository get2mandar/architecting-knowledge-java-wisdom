# Post #77: Model Registry and Artifact Management - Version Control for ML Experiments
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** October 28, 2026  
**Topic:** Model Registry, Artifact Management, Experiment Tracking, Reproducibility  
 
---
 
## The Problem
 
Production accuracy drops. Someone asks, "Which model is running, and how was it trained?" The artifact is a file called `fraud_model_final_v2_REAL.pkl` on a shared drive. Nobody remembers the training data snapshot, the hyperparameters, or the code commit behind it. Rebuilding it is guesswork. Code has Git; models need the same discipline, or every incident becomes archaeology.
 
## Code Example
 
### ❌ Models as Loose Files - No Lineage
 
```java
// Model Loaded From a Hard-Coded Path
@Service
public class FileBasedModelLoader {
 
    public FraudModel load() {
        return ModelSerializer.read(new File("/shared/models/fraud_model_final_v2_REAL.bin"));
        // Which Data? Which Code? Which Metrics? Unknown.
    }
}
 
// Failure Scenario:
// 1. File Overwritten Silently by Another Experiment
// 2. Production Behavior Changes With No Deployment Event
// 3. No Record of Training Data, Parameters, or Evaluation Results
// 4. Reproducing the Previous Model Is Not Possible
```
 
### ✅ Solution 1: Registry Entry - Metadata Travels With the Artifact
 
```java
/*
REGISTRY RECORD:
  - Immutable Artifact + Full Lineage + Lifecycle Stage
*/
public record ModelVersion(
    String modelName,
    int version,
    String artifactUri,        // Immutable Storage Location
    String gitCommit,          // Exact Training Code
    String datasetSnapshotId,  // Exact Training Data
    Map<String, String> hyperParameters,
    Map<String, Double> metrics,
    Stage stage                // STAGING, PRODUCTION, ARCHIVED
) {}
```
 
### ✅ Solution 2: Immutable Versions and Explicit Promotion
 
```java
@Service
public class ModelRegistryService {
 
    public ModelVersion register(ModelCandidate candidate) {
        // Every Registration Creates a New Version - Never Overwrites
        int next = repository.latestVersion(candidate.name()) + 1;
        return repository.save(candidate.toVersion(next, Stage.STAGING));
    }
 
    public void promote(String name, int version) {
        ModelVersion v = repository.get(name, version);
        if (v.metrics().get("auc") < MIN_AUC) {
            throw new PromotionRejectedException("Metrics Below Threshold");
        }
        repository.setStage(name, version, Stage.PRODUCTION);  // Auditable Transition
    }
}
```
 
### ✅ Solution 3: Serving Resolves by Stage, Not by Path
 
```java
@Service
public class RegistryBackedModelLoader {
 
    public FraudModel loadProduction() {
        ModelVersion v = registry.getByStage("fraud-model", Stage.PRODUCTION);
        logger.info("Loading fraud-model v{} (commit {})", v.version(), v.gitCommit());
        return ModelSerializer.read(artifactStore.open(v.artifactUri()));
    }
}
 
// Rollback = Change the Stage Pointer Back to Version N-1. No Rebuild Required.
```
 
## Why This Matters
 
A model is the product of code, data, and parameters, and losing any one of the three makes it irreproducible. A registry captures all three alongside the artifact and treats versions as immutable, so nothing changes without leaving a trail. Promotion gates add a quality check between experiment and production, and resolving models by stage rather than file path turns rollback into a pointer change. This is the same discipline Java teams already apply to libraries in an artifact repository, extended to models.
 
## Key Takeaway
 
Never ship an unnamed file. Register every model with its code commit, data snapshot, parameters, and metrics. Keep versions immutable, promote through explicit stages with gates, and let serving resolve by stage so rollback is instant and every production behavior is traceable.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#ModelRegistry` `#MLOps` `#ArtifactManagement` `#Reproducibility` `#ExperimentTracking` `#SpringBoot` `#DistributedSystems` `#Microservices` `#Reliability` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
