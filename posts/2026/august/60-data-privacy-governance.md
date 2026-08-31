# Post #60: Data Privacy and Governance in ML Systems - Compliance and Ethics

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** August 30, 2026  
**Topic:** Data Privacy, GDPR, Audit Trails, Model Lineage, Compliance

---

## The Problem

Your model trained on customer data. GDPR says customers can request deletion. Do you know where their PII lives? In 100 tables? 5 systems? Good luck finding it all. Without governance, you're building non-compliant systems that could cost millions in fines.

## Code Example

### ❌ No Governance - Compliance Risk

```java
// No Tracking = No Compliance
@Service
public class UngovermedMLSystem {
    
    public void buildFraudModel() {
        // Load Customer Data (From Somewhere)
        DataFrame customerData = spark.read()
            .parquet("s3://data/customers/2026-01-01/");
        
        // Add Behavioral Features (From Somewhere)
        DataFrame behavioralData = spark.read()
            .parquet("s3://data/behavioral/");
        
        // Join Data (SQL at Random Path)
        DataFrame features = customerData.join(behavioralData);
        
        // Train Model (Notebook Script)
        Model model = train(features);
        
        // Deploy (Manual Upload)
        model.save("s3://models/fraud_detector_v1");
        
        // Problems When GDPR Request Comes:
        // Q: Where is customer_id: 12345 data?
        // A: ¯\_(ツ)_/¯
        // - Is it in customerData? YES
        // - Is it in behavioralData? YES
        // - Is it in the trained model? MAYBE
        // - Is it in predictions cache? UNKNOWN
        // - Is it in logs? PROBABLY
        // - Is it in backups? YES
        //
        // Can't Delete = GDPR Violation = €20 Million Fine
    }
}
```

### ✅ Solution 1: Data Lineage - Track Data Flow

```java
/*
DATA LINEAGE:
  - Document Where Data Comes From
  - Track How Data Flows Through Systems
  - Field-Level Granularity (Not Just Tables)
  - Answer: "Where is customer_id 12345?"
*/

@Configuration
public class DataLineageTracking {
    
    @Bean
    public LineageTracer lineageTracer() {
        return new LineageTracer();
    }
}

@Service
public class TrackedMLPipeline {
    
    @Autowired
    private LineageTracer lineage;
    
    public void buildFraudModel() {
        // Step 1: Load Raw Data (Track Source)
        DataFrame rawCustomers = spark.read()
            .parquet("s3://data/raw/customers/");
        
        lineage.recordSource("raw_customers",
            "s3://data/raw/customers/",
            columns = Set.of("customer_id", "age", "email", "phone"));
        
        // Step 2: Load Behavioral Data
        DataFrame rawBehavioral = spark.read()
            .parquet("s3://data/raw/behavioral/");
        
        lineage.recordSource("raw_behavioral",
            "s3://data/raw/behavioral/",
            columns = Set.of("customer_id", "transaction_count", "avg_amount"));
        
        // Step 3: Transform Data (Track Lineage)
        DataFrame features = rawCustomers.join(rawBehavioral, "customer_id")
            .select(
                col("customer_id"),      // Source: raw_customers
                col("age"),              // Source: raw_customers
                col("transaction_count"), // Source: raw_behavioral
                col("avg_amount")        // Source: raw_behavioral
            );
        
        lineage.recordTransform("features",
            inputs = Set.of("raw_customers", "raw_behavioral"),
            columns = Set.of("customer_id", "age", "transaction_count", "avg_amount"));
        
        // Step 4: Train Model (Track Data Version)
        Model model = train(features);
        
        lineage.recordModel("fraud_detector_v1",
            trainingData = "features",
            version = "v1",
            trainDate = LocalDate.now());
        
        // Step 5: Deploy Model
        model.save("s3://models/fraud_detector_v1");
        
        // Step 6: Make Predictions
        DataFrame predictions = model.predict(features);
        
        lineage.recordPrediction("fraud_predictions",
            model = "fraud_detector_v1",
            inputData = "features");
    }
    
    // Query: Where is customer_id 12345?
    public void handleGDPRDeletionRequest(String customerId) {
        // Lineage System Answers:
        Set<String> locations = lineage.findDataLocations(customerId);
        
        // Result:
        // - s3://data/raw/customers/ (DELETE)
        // - s3://data/raw/behavioral/ (DELETE)
        // - s3://features/ (DERIVED - DELETE)
        // - fraud_detector_v1 Model (RETRAIN - Customer Influence Removed)
        // - s3://predictions/ (DELETE)
        // - Elasticsearch Logs (REDACT)
        
        for (String location : locations) {
            deleteCustomerData(customerId, location);
        }
        
        // GDPR Compliant Deletion
        logger.info("Deleted all data for customer {}", customerId);
    }
}
```

### ✅ Solution 2: PII Detection and Redaction

```java
@Service
public class PIIDetection {
    
    public class PIIColumn {
        String columnName;
        String dataType;
        String piiType;  // EMAIL, PHONE, SSN, CREDIT_CARD, etc.
        Boolean requiresEncryption;
    }
    
    public List<PIIColumn> detectPII(DataFrame df) {
        List<PIIColumn> piiColumns = new ArrayList<>();
        
        // Regex Patterns for Common PII
        Pattern emailPattern = Pattern.compile(
            "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}"
        );
        Pattern phonePattern = Pattern.compile(
            "\\d{3}[-.]?\\d{3}[-.]?\\d{4}"
        );
        Pattern ssnPattern = Pattern.compile(
            "\\d{3}-\\d{2}-\\d{4}"
        );
        
        // Scan Each Column
        for (String column : df.columns()) {
            List<String> samples = df.select(column)
                .limit(1000)
                .collect();
            
            for (String sample : samples) {
                if (emailPattern.matcher(sample).matches()) {
                    piiColumns.add(new PIIColumn(
                        column, "String", "EMAIL", true
                    ));
                    break;
                }
                if (phonePattern.matcher(sample).matches()) {
                    piiColumns.add(new PIIColumn(
                        column, "String", "PHONE", true
                    ));
                    break;
                }
                if (ssnPattern.matcher(sample).matches()) {
                    piiColumns.add(new PIIColumn(
                        column, "String", "SSN", true
                    ));
                    break;
                }
            }
        }
        
        return piiColumns;
    }
    
    // Redact PII Before Logging
    public String redactPII(String text, List<PIIColumn> piiColumns) {
        // Replace EMAIL with [REDACTED_EMAIL]
        text = text.replaceAll(
            "[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}",
            "[REDACTED_EMAIL]"
        );
        
        // Replace PHONE with [REDACTED_PHONE]
        text = text.replaceAll(
            "\\d{3}[-.]?\\d{3}[-.]?\\d{4}",
            "[REDACTED_PHONE]"
        );
        
        return text;
    }
}
```

### ✅ Solution 3: Audit Trails - Immutable Logs

```java
@Entity
@Table(name = "audit_log")
public class AuditLog {
    
    @Id
    private String eventId;
    
    @Column(nullable = false)
    private LocalDateTime timestamp;
    
    @Column(nullable = false)
    private String action;  // READ, WRITE, DELETE, TRAIN, PREDICT
    
    @Column(nullable = false)
    private String user;  // Who Did It?
    
    @Column(nullable = false)
    private String resource;  // What Data?
    
    @Column(nullable = false)
    private String details;  // How Many Records? Which Model?
    
    @Column(nullable = false)
    private Boolean success;
    
    // GDPR Compliant - Never Update After Creation
    // Only Append New Records
}

@Service
public class AuditTrail {
    
    @Autowired
    private AuditLogRepository auditRepository;
    
    // Log Every Data Access
    public void logModelPrediction(
        String userId,
        String modelVersion,
        int recordsProcessed
    ) {
        AuditLog log = new AuditLog();
        log.setEventId(UUID.randomUUID().toString());
        log.setTimestamp(LocalDateTime.now());
        log.setAction("PREDICT");
        log.setUser(userId);
        log.setResource(modelVersion);
        log.setDetails("Processed " + recordsProcessed + " records");
        log.setSuccess(true);
        
        auditRepository.save(log);  // Append Only
    }
    
    // Query: Who Accessed Customer Data?
    public List<AuditLog> queryDataAccess(String customerId) {
        return auditRepository.findByResource(customerId)
            .stream()
            .sorted(Comparator.comparing(AuditLog::getTimestamp))
            .collect(Collectors.toList());
    }
}
```

### ✅ Solution 4: Model Cards - Documentation

```java
@Service
public class ModelCard {
    
    public record DocumentedModel(
        String modelId,
        String version,
        LocalDate trainDate,
        
        // Training Data
        String trainingDataSource,
        Integer trainingRecordCount,
        Set<String> trainingColumnsUsed,
        Set<String> piiColumnsUsed,
        
        // Model Performance
        Double accuracy,
        Double precision,
        Double recall,
        Double f1Score,
        
        // Limitations
        String limitations,  // "Biased against X group"
        
        // Ethical Considerations
        String ethicsStatement,
        
        // Compliance
        String gdprJustification,  // "Legitimate Interest per GDPR 6(1)(f)"
        String dpiaRequired,
        
        // Responsible Use
        String intendedUse,
        String prohibitedUses
    ) {}
    
    public void documentModel(String modelId) {
        DocumentedModel card = new DocumentedModel(
            modelId = "fraud_detector_v2",
            version = "2.1.0",
            trainDate = LocalDate.now().minusMonths(3),
            
            trainingDataSource = "Customer Transactions 2025-2026",
            trainingRecordCount = 500_000,
            trainingColumnsUsed = Set.of(
                "transaction_amount", "merchant_category", 
                "user_age", "transaction_frequency"
            ),
            piiColumnsUsed = Set.of("user_age"),  // Only Age, Not Email/Phone
            
            accuracy = 0.945,
            precision = 0.87,
            recall = 0.82,
            f1Score = 0.845,
            
            limitations = "Model Shows 3% Higher FPR for Customers Age 18-25",
            
            ethicsStatement = "Model Trained on Balanced Dataset. Regular Audits for Bias.",
            
            gdprJustification = "Legitimate Interest: Fraud Prevention (GDPR 6(1)(f)). DPIA Completed.",
            dpiaRequired = true,
            
            intendedUse = "Real-Time Fraud Detection for Credit Card Transactions",
            prohibitedUses = "Must NOT Use for: Loan Decisions, Employment Screening, or Profiling"
        );
        
        modelRegistry.save(card);
    }
}
```

## 💡 Why This Matters

The European Central Bank's May 2024 guide explicitly calls out attribute-level data lineage as a supervisory concern. GDPR right-to-erasure requires identifying every column across every system that holds a given individual's data. The EU AI Act (effective August 2027) demands comprehensive technical documentation and automatic logging for high-risk AI systems. Organizations across sectors show a "big gap between AI adoption and compliance preparedness," with many unaware of applicable regulations.

## 🎯 Key Takeaway

Build lineage from day one—field-level, not table-level. Detect and redact PII before logging. Maintain append-only audit trails. Document models with ethics and compliance statements. Treat privacy as architecture, not afterthought. The cost of compliance now << cost of fines later.

---

**Tags:** `#Java` `#JavaWisdom` `#DataPrivacy` `#GDPR` `#DataGovernance` `#Compliance` `#ModelLineage` `#AuditTrail` `#PII` `#DataProtection` `#SpringBoot` `#MLOps` `#Ethics` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity` `#SoftwareEngineering`
