# Post #57: Data Validation and Quality Checks - Preventing Bad Data

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** August 19, 2026  
**Topic:** Data Validation, Great Expectations, Schema Enforcement, Data Contracts

---

## The Problem

Bad data flows through your pipeline silently. A missing column here. A negative age there. Null values everywhere. By the time you notice, your model trained on garbage. Data quality issues are the leading cause of model failures—yet most teams have no validation layer.

## Code Example

### ❌ No Validation - Silent Failures

```java
// No Validation = Garbage In, Garbage Out
@Service
public class UnvalidatedDataPipeline {
    
    public void processTransactions(List<Transaction> rawTransactions) {
        // Assume Data Is Valid (Dangerous!)
        
        for (Transaction tx : rawTransactions) {
            // Process Directly
            // What If:
            // - tx.amount is NULL? → NullPointerException
            // - tx.amount is Negative? → Bad Feature
            // - tx.merchant_category is Empty String? → Data Corruption
            // - tx.timestamp is in Future? → Bad Data
            
            features.put("amount", tx.getAmount());  // No Validation!
            model.predict(features);
        }
    }
}

// Real-World Impact:
// 1. First 1000 transactions: Invalid
// 2. No Errors Thrown (Silently Accepted)
// 3. Model Trains on Garbage
// 4. Predictions Become Unreliable
// 5. Business Impacts Undetected Until Too Late
```

### ✅ Solution 1: Great Expectations - Test Data Like Code

```java
/*
GREAT EXPECTATIONS:
  - Define Expectations (Rules) Like Unit Tests
  - Validate Data Before Processing
  - Generate Automated Reports
  - Integrate with ETL Pipelines
*/

// Define Expectations Once
@Configuration
public class DataQualityExpectations {
    
    @Bean
    public ExpectationSuite transactionExpectations() {
        ExpectationSuite suite = new ExpectationSuite();
        
        // Schema Expectations
        suite.addExpectation("expect_table_columns_to_match_set",
            Set.of("transaction_id", "amount", "merchant_category", 
                   "timestamp", "user_id"));
        
        // Type Expectations
        suite.addExpectation("expect_column_values_to_be_of_type",
            "amount", "Float");
        suite.addExpectation("expect_column_values_to_be_of_type",
            "timestamp", "DateTime");
        
        // Null Expectations
        suite.addExpectation("expect_column_values_to_not_be_null",
            "transaction_id");
        suite.addExpectation("expect_column_values_to_not_be_null",
            "amount");
        
        // Value Range Expectations
        suite.addExpectation("expect_column_values_to_be_between",
            "amount", 0.0, 10000.0);  // Transactions Between $0-$10k
        
        // Business Logic Expectations
        suite.addExpectation("expect_column_values_to_not_be_in_set",
            "merchant_category", Set.of("", "UNKNOWN", "NULL"));
        
        // Uniqueness Expectations
        suite.addExpectation("expect_primary_key_column_values_to_be_unique",
            "transaction_id");
        
        return suite;
    }
}

// Validate Data in Pipeline
@Service
public class ValidatedDataPipeline {
    
    @Autowired
    private ExpectationSuite expectations;
    
    public void processTransactions(Dataset<Row> df) {
        // Validate Against Expectations
        ValidationResult result = expectations.validate(df);
        
        if (!result.isSuccessful()) {
            // Log Failures
            result.getFailures().forEach(failure -> 
                logger.error("Data Quality Check Failed: {} - {}",
                    failure.getExpectation(),
                    failure.getReason())
            );
            
            // Alert and Stop Processing
            alertOps("Data Quality Issues Detected");
            throw new DataQualityException("Invalid Data");
        }
        
        // Data is Guaranteed Valid
        // Safe to Process
        df.foreach(row -> processValidRow(row));
    }
}
```

### ✅ Schema Validation - Type Safety

```java
// Define Schema as Code
public class TransactionSchema {
    
    public static StructType getExpectedSchema() {
        return new StructType()
            .add("transaction_id", DataTypes.LongType, false)  // Not Null
            .add("amount", DataTypes.DoubleType, false)        // Not Null
            .add("merchant_category", DataTypes.StringType, false)
            .add("timestamp", DataTypes.TimestampType, false)
            .add("user_id", DataTypes.LongType, false);
    }
}

// Enforce Schema at Ingestion
@Service
public class SchemaValidation {
    
    public Dataset<Row> loadAndValidate(String path) {
        // Load Data
        Dataset<Row> df = spark.read()
            .schema(TransactionSchema.getExpectedSchema())
            .parquet(path);
        
        // Check Row Count
        long rowCount = df.count();
        if (rowCount == 0) {
            throw new DataQualityException("No Data Found");
        }
        
        // Check for Nulls in Required Columns
        df.select("transaction_id", "amount").na().drop().count();
        
        return df;
    }
}
```

### ✅ Custom Data Quality Checks

```java
@Service
public class CustomQualityChecks {
    
    public class DataQualityReport {
        Integer totalRecords;
        Integer invalidRecords;
        Double validityPercentage;
        List<String> issues;
    }
    
    public DataQualityReport runQualityChecks(Dataset<Row> df) {
        DataQualityReport report = new DataQualityReport();
        report.totalRecords = (int) df.count();
        report.issues = new ArrayList<>();
        
        // Check 1: Amount Validation
        long negativeAmounts = df
            .filter(df.col("amount").lt(0))
            .count();
        
        if (negativeAmounts > 0) {
            report.issues.add("Found " + negativeAmounts + " Negative Amounts");
        }
        
        // Check 2: Timestamp Validation
        long futureTimestamps = df
            .filter(df.col("timestamp").gt(functions.current_timestamp()))
            .count();
        
        if (futureTimestamps > 0) {
            report.issues.add("Found " + futureTimestamps + " Future Timestamps");
        }
        
        // Check 3: Merchant Category Validation
        Set<String> validCategories = Set.of(
            "RETAIL", "GROCERY", "RESTAURANT", "FUEL", "HOTEL"
        );
        
        long invalidCategories = df
            .filter(f -> !validCategories.contains(f.getAs("merchant_category")))
            .count();
        
        if (invalidCategories > 0) {
            report.issues.add("Found " + invalidCategories + " Invalid Categories");
        }
        
        // Calculate Validity
        report.invalidRecords = (int) (negativeAmounts + futureTimestamps + invalidCategories);
        report.validityPercentage = 
            (double) (report.totalRecords - report.invalidRecords) / report.totalRecords * 100;
        
        return report;
    }
}

// Real-World Example: Fraud Detection
@Service
public class FraudDataValidation {
    
    public DataQualityReport validateFraudDataset(Dataset<Row> df) {
        DataQualityReport report = new DataQualityReport();
        
        // Business Logic: Fraud Rates Should Be 1-5%
        long fraudCases = df.filter(df.col("is_fraud").equalTo(true)).count();
        double fraudRate = (double) fraudCases / df.count();
        
        if (fraudRate < 0.01 || fraudRate > 0.05) {
            report.issues.add(
                "Fraud Rate Anomaly: " + (fraudRate * 100) + "% (Expected 1-5%)"
            );
        }
        
        // Check Feature Completeness
        Set<String> requiredFeatures = Set.of(
            "transaction_amount", "merchant_type", "user_age",
            "transaction_frequency", "days_since_signup"
        );
        
        for (String feature : requiredFeatures) {
            long nullCount = df.filter(df.col(feature).isNull()).count();
            if (nullCount > 0) {
                report.issues.add("Missing Values in " + feature + ": " + nullCount);
            }
        }
        
        return report;
    }
}
```

### ✅ Integration with Data Contracts

```java
/*
DATA CONTRACTS:
  - Formal Agreements Between Data Producers and Consumers
  - Defines Schema, Freshness, Quality Expectations
  - Violations Trigger Alerts/Stops
*/

@Entity
public class DataContract {
    
    String contractId;
    String producingTeam;
    String consumingTeam;
    
    // Schema Specification
    Set<String> requiredColumns;
    Map<String, String> columnTypes;
    
    // Freshness SLA
    Duration maxDataAge;  // Data must be < 1 hour old
    
    // Quality Expectations
    Double minCompleteness;  // 99% of rows must be valid
    Double maxNullRate;      // < 0.1% null values
    Integer expectedRowCount;
    Integer rowCountTolerance;
}

// Enforce Contract at Ingest
@Service
public class ContractEnforcement {
    
    @Autowired
    private DataContractRepository contractRepository;
    
    public void validateDataContract(
        String contractId,
        Dataset<Row> df
    ) {
        DataContract contract = contractRepository.findById(contractId);
        
        // Validate Schema
        Set<String> actualColumns = Set.of(df.columns());
        if (!actualColumns.containsAll(contract.requiredColumns)) {
            throw new ContractViolation("Missing Required Columns");
        }
        
        // Validate Freshness
        Column latestTimestamp = df.agg(
            functions.max("event_timestamp")
        ).collectAsList().get(0).getTimestamp(0);
        
        Duration age = Duration.between(
            latestTimestamp.toInstant(),
            Instant.now()
        );
        
        if (age.compareTo(contract.maxDataAge) > 0) {
            throw new ContractViolation("Data Too Old: " + age);
        }
        
        // Validate Quality
        long validRows = df.filter(df.col("quality_check").equalTo(true)).count();
        double completeness = (double) validRows / df.count();
        
        if (completeness < contract.minCompleteness) {
            throw new ContractViolation(
                "Data Completeness " + completeness + " < " + 
                contract.minCompleteness
            );
        }
        
        logger.info("Data Contract {} Validated Successfully", contractId);
    }
}
```

## 💡 Why This Matters

Data quality problems tend to surface only after they've already hit downstream systems or business decisions. Great Expectations provides a framework for testing, documenting, and profiling your data pipelines. Inconsistent data types arise when data is ingested from diverse sources or when schema definitions are updated without comprehensive checks. Schema evolution, if not managed correctly, can lead to significant disruptions and data compatibility issues.

## 🎯 Key Takeaway

Start small with basic expectations: not null, type checks, value ranges. Automate validation in every ETL pipeline. Define data contracts between teams. Monitor data quality metrics like completeness and null rates. Bad data beats every optimization—prevent it with validation.

---

**Tags:** `#Java` `#JavaWisdom` `#DataValidation` `#DataQuality` `#GreatExpectations` `#DataContracts` `#SchemaValidation` `#DataGovernance` `#SpringBoot` `#ETL` `#DataPipelines` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
