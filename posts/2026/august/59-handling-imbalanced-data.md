# Post #59: Handling Imbalanced Data - Sampling, Weighting, and Rebalancing

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** August 26, 2026  
**Topic:** Imbalanced Data, SMOTE, Cost-Sensitive Learning, Ensemble Methods

---

## The Problem

Your fraud detection model reports 99% accuracy. But in production, it catches zero fraud. Why? Your training data was 99% legitimate transactions. The model learned: "Always predict legitimate." This is the imbalanced data problem—and it destroys production models silently.

## Code Example

### ❌ Naive Approach - Accuracy Theater

```java
// Training Data: 99% Legitimate, 1% Fraud
public class ImbalancedFraudModel {
    
    public void trainModel(List<Transaction> allTransactions) {
        // Naive Split: 80/20 (But Ratio Unchanged!)
        List<Transaction> training = allTransactions.subList(0, 8000);  // ~1% Fraud
        List<Transaction> testing = allTransactions.subList(8000, 10000);  // ~1% Fraud
        
        // Train Model
        model.fit(training);
        
        // Evaluate
        double accuracy = model.evaluate(testing);
        System.out.println("Accuracy: " + accuracy);  // 99.2%!
        
        // Model Decision: 
        // If Always Predict "Legitimate":
        // - Correct: 9900 * 0.992 = 9820
        // - Accuracy: 9820 / 10000 = 98.2%
        
        // But in Production:
        // Fraud: 0% Detected (Model Never Predicted Fraud!)
    }
}

// Problem: Accuracy ≠ Performance
// High Accuracy with Zero Fraud Detection
// Precision = 0/1 (Never Predicts Fraud)
// Recall = 0/100 (Misses All Fraud)
// F1-Score = 0 (Disaster)
```

### ✅ Solution 1: Class Weighting - Penalize Fraud Misclassification

```java
/*
CLASS WEIGHTING:
  - Assign Higher Penalty to Minority Class Errors
  - Model Pays Attention to Fraud Cases
  - Balances Without Changing Data Size
*/

@Service
public class CostSensitiveLearning {
    
    public void trainWithClassWeights(List<Transaction> transactions) {
        // Calculate Class Weights
        long fraudCount = transactions.stream()
            .filter(tx -> tx.isFraud())
            .count();
        long legitCount = transactions.size() - fraudCount;
        
        // Weight = Total / (Class Count * 2)
        // This Makes Minority Class Heavier
        double fraudWeight = (double) transactions.size() / 
                            (fraudCount * 2);
        double legitWeight = (double) transactions.size() / 
                            (legitCount * 2);
        
        System.out.println("Fraud Weight: " + fraudWeight);  // ~50
        System.out.println("Legit Weight: " + legitWeight);   // ~0.5
        
        // Train with Weights
        // Model Loss = Sum(weight[i] * prediction_error[i])
        // Fraud Errors Cost 50x More Than Legit Errors!
        model.fit(transactions, 
            classWeights = Map.of(
                "fraud", fraudWeight,
                "legit", legitWeight
            )
        );
    }
}

// Gradient Boosting with Class Weights
@Service
public class WeightedGradientBoosting {
    
    public void trainXGBoost(DataFrame trainingData) {
        // XGBoost Supports Scale_pos_weight
        int fraudCount = (int) trainingData
            .filter("is_fraud = 1")
            .count();
        int legitCount = (int) trainingData
            .filter("is_fraud = 0")
            .count();
        
        double scalePosWeight = (double) legitCount / fraudCount;  // ~99
        
        XGBParams params = new XGBParams()
            .scale_pos_weight(scalePosWeight)
            .max_depth(6)
            .learning_rate(0.1);
        
        model.train(trainingData, params);
        // Result: Model Now Detects ~70% of Fraud
    }
}
```

### ✅ Solution 2: Resampling - SMOTE Synthetic Minority Oversampling

```java
/*
SMOTE (Synthetic Minority OverSampling Technique):
  - Generate Synthetic Fraud Cases
  - Use KNN to Create Realistic Synthetic Samples
  - Interpolate Between Existing Fraud Cases
*/

@Service
public class SMOTEResampling {
    
    public List<Transaction> applySMOTE(
        List<Transaction> imbalancedTransactions,
        int samplingRatio = 100  // Generate Until 1:1 Ratio
    ) {
        // Separate Classes
        List<Transaction> fraudCases = imbalancedTransactions.stream()
            .filter(Transaction::isFraud)
            .collect(Collectors.toList());
        
        List<Transaction> legitCases = imbalancedTransactions.stream()
            .filter(tx -> !tx.isFraud())
            .collect(Collectors.toList());
        
        // Generate Synthetic Fraud Cases
        List<Transaction> syntheticFraud = new ArrayList<>();
        
        for (Transaction fraudTx : fraudCases) {
            // Find 5 Nearest Neighbors (KNN)
            List<Transaction> neighbors = findNearestNeighbors(
                fraudTx, 
                fraudCases, 
                k = 5
            );
            
            // Randomly Select Neighbor
            Transaction neighbor = neighbors.get(random(0, 5));
            
            // Interpolate Between Original and Neighbor
            // New Fraud = Original + Random(0,1) * (Neighbor - Original)
            Transaction syntheticCase = interpolate(fraudTx, neighbor);
            syntheticFraud.add(syntheticCase);
        }
        
        // Combine Original + Synthetic
        List<Transaction> balancedData = new ArrayList<>();
        balancedData.addAll(imbalancedTransactions);
        balancedData.addAll(syntheticFraud);  // ~1:1 Ratio Now
        
        return balancedData;
    }
    
    private Transaction interpolate(Transaction tx1, Transaction tx2) {
        double alpha = Math.random();  // 0.0 to 1.0
        
        return new Transaction()
            .setAmount(tx1.getAmount() + alpha * (tx2.getAmount() - tx1.getAmount()))
            .setMerchantType(tx1.getMerchantType())  // Categorical: Keep Original
            .setUserAge(tx1.getUserAge() + alpha * (tx2.getUserAge() - tx1.getUserAge()))
            .setIsFraud(true);  // Label as Fraud
    }
}

// Alternative: ADASYN (Adaptive Synthetic Sampling)
@Service
public class ADASYN {
    
    public List<Transaction> applyADASYN(List<Transaction> data) {
        // ADASYN Improves on SMOTE:
        // - Generates More Samples for Difficult Cases
        // - Less Samples for Easy Cases
        // - Better Boundary Definition
        
        List<Transaction> fraudCases = filterFraud(data);
        
        for (Transaction fraudTx : fraudCases) {
            // Calculate Difficulty Score
            int knnFraudNeighbors = countFraudNeighbors(fraudTx, data, k=10);
            double difficulty = 1.0 - (knnFraudNeighbors / 10.0);
            
            // Generate More Samples for Difficult Cases
            int syntheticSamplesToGenerate = (int) (difficulty * 5);
            
            for (int i = 0; i < syntheticSamplesToGenerate; i++) {
                Transaction neighbor = findRandomFraudNeighbor(fraudTx, data);
                Transaction synthetic = interpolate(fraudTx, neighbor);
                data.add(synthetic);
            }
        }
        
        return data;
    }
}
```

### ✅ Solution 3: Hybrid Approach - Over + Under Sampling

```java
@Service
public class HybridResampling {
    
    public List<Transaction> hybridBalance(List<Transaction> data) {
        // Step 1: Oversample Minority (Use SMOTE)
        List<Transaction> oversampled = applySMOTE(data, 100);
        
        // Step 2: Undersample Majority (Remove Random Legit Cases)
        List<Transaction> legit = oversampled.stream()
            .filter(tx -> !tx.isFraud())
            .collect(Collectors.toList());
        
        int targetLegitCount = oversampled.stream()
            .filter(Transaction::isFraud)
            .count() * 2;  // 2:1 Ratio
        
        Collections.shuffle(legit);  // Random Selection
        List<Transaction> undersampledLegit = 
            legit.subList(0, targetLegitCount);
        
        // Combine
        List<Transaction> balanced = new ArrayList<>();
        balanced.addAll(oversampled.stream()
            .filter(Transaction::isFraud)
            .collect(Collectors.toList()));
        balanced.addAll(undersampledLegit);
        
        return balanced;
    }
}
```

### ✅ Real-World Example - Credit Card Fraud Detection

```java
@Service
public class FraudDetectionPipeline {
    
    public record BalancingStrategy(
        String name,
        List<Transaction> trainingData,
        Double fraudRecall,    // % of Fraud Detected
        Double precision,      // % of Fraud Alerts That Are Real Fraud
        Double f1Score
    ) {}
    
    public void compareSamplingStrategies(List<Transaction> rawData) {
        // Strategy 1: No Resampling (Baseline)
        BalancingStrategy baseline = new BalancingStrategy(
            "Baseline (No Resampling)",
            rawData,
            0.35,  // Catches 35% of Fraud
            0.15,  // But False Alarm Rate High
            0.22
        );
        
        // Strategy 2: Class Weighting
        BalancingStrategy weighted = new BalancingStrategy(
            "Class Weighting",
            rawData,  // Same Data
            0.68,     // Catches 68% of Fraud
            0.42,     // Better Precision
            0.52
        );
        
        // Strategy 3: SMOTE
        List<Transaction> smo tedData = applySMOTE(rawData);
        BalancingStrategy smote = new BalancingStrategy(
            "SMOTE",
            smotedData,
            0.75,  // Catches 75% of Fraud
            0.58,  // Good Precision
            0.65
        );
        
        // Strategy 4: Hybrid (SMOTE + Class Weights)
        List<Transaction> hybridData = hybridBalance(rawData);
        BalancingStrategy hybrid = new BalancingStrategy(
            "Hybrid (SMOTE + Weights)",
            hybridData,
            0.82,  // Catches 82% of Fraud
            0.64,  // High Precision
            0.72
        );
        
        // Recommendation: Use Hybrid
        // - Recalls More Fraud (82%)
        // - Maintains Precision (64%)
        // - Best F1-Score (0.72)
    }
}

// Production Deployment:
// - Use Hybrid Resampling
// - Monitor Fraud Recall > 80%
// - Monitor False Positive Rate < 2%
// - Retrain Quarterly as Fraud Patterns Change
```

## 💡 Why This Matters

Data-level methods like undersampling, oversampling, and hybrid approaches modify the dataset itself to improve performance. SMOTE generates synthetic samples for the minority class, creating realistic interpolated examples. Cost-sensitive learning modifies algorithms to consider class imbalance by assigning misclassification costs. Ensemble methods like XGBoost with scale_pos_weight handle imbalance naturally through their training objective.

## 🎯 Key Takeaway

Never train on imbalanced data without handling it. Use class weighting first (simplest). Add SMOTE if you need more data. Combine both for production systems. Always optimize for business metrics (recall/precision), not accuracy. Test on realistic imbalanced test sets—not artificially balanced ones.

---

**Tags:** `#Java` `#JavaWisdom` `#ImbalancedData` `#SMOTE` `#CostSensitiveLearning` `#ClassWeighting` `#Resampling` `#MachineLearning` `#FraudDetection` `#SpringBoot` `#MLOps` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
