# Post #56: Feature Engineering in Production - From Notebook to Pipeline

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** August 16, 2026  
**Topic:** Feature Engineering, Pandas to Spark, Distributed Processing, Scalability

---

## The Problem

Your Jupyter notebook works perfectly. The feature engineering logic is elegant in pandas. Deploy to production and it crawls. Features take hours instead of seconds. Scaling a notebook is not scaling engineering—it's a complete rewrite.

## Code Example

### ❌ Notebook-Only Approach - Local Processing Hell

```python
# Jupyter Notebook - Works Great on 100K Rows
import pandas as pd
import numpy as np

# Load Data (Small Dataset)
df = pd.read_csv('users.csv')  # 100K Rows, Fits in Memory

# Feature Engineering (Beautiful, Readable)
def engineer_features(df):
    # Time-Based Features
    df['days_since_signup'] = (pd.Timestamp.now() - df['signup_date']).dt.days
    
    # Aggregations
    purchase_stats = df.groupby('user_id').agg({
        'purchase_amount': ['sum', 'mean', 'std'],
        'purchase_date': 'count'
    }).reset_index()
    
    # Joins
    df = df.merge(purchase_stats, on='user_id', how='left')
    
    # Complex Transformations
    df['log_purchase_sum'] = np.log(df[('purchase_amount', 'sum')] + 1)
    df['purchase_frequency_per_day'] = df[('purchase_date', 'count')] / df['days_since_signup']
    
    return df

# Run on 100K Rows: 5 Seconds ✅
features = engineer_features(df)

# PROBLEM: Deploy to Production with 500M Rows
# Expected: 500M / 100K * 5 Seconds = 25,000 Seconds (6.9 Hours!)
# Actual: ❌ Out of Memory
# Memory Needed: 500M * 200 Bytes/Row = 100 GB
# Server Has: 32 GB
# Result: Server Crashes
```

### ✅ Migrate to Apache Spark - Distributed Processing

```java
// PySpark Version - Scales to Billions of Rows
from pyspark.sql import SparkSession, functions as F
from pyspark.sql.types import DoubleType

# Initialize Spark
spark = SparkSession.builder \
    .appName("feature-engineering") \
    .getOrCreate()

# Load Data (Can Be GB/TB/PB - Distributed)
df = spark.read.parquet("s3://data/users/")  # 500M Rows, Distributed

# Same Feature Engineering Logic (Works on Any Scale!)
def engineer_features_spark(df):
    # Time-Based Features
    df = df.withColumn(
        "days_since_signup",
        F.datediff(F.current_timestamp(), F.col("signup_date"))
    )
    
    # Aggregations (Distributed!)
    purchase_stats = df.groupBy("user_id").agg(
        F.sum("purchase_amount").alias("purchase_sum"),
        F.avg("purchase_amount").alias("purchase_avg"),
        F.stddev("purchase_amount").alias("purchase_stddev"),
        F.count("purchase_date").alias("purchase_count")
    )
    
    # Joins (Distributed!)
    df = df.join(purchase_stats, on="user_id", how="left")
    
    # Complex Transformations (Vectorized!)
    df = df.withColumn(
        "log_purchase_sum",
        F.log(F.col("purchase_sum") + 1)
    ).withColumn(
        "purchase_frequency_per_day",
        F.col("purchase_count") / F.col("days_since_signup")
    )
    
    return df

# Run on 500M Rows: 60 Seconds ✅
features = engineer_features_spark(df)

# Write to Feature Store
features.write.mode("overwrite").parquet("s3://features/")
```

### ✅ Java Implementation - Production-Grade Feature Pipelines

```java
// Production Feature Pipeline in Java/Spark
@Service
public class FeatureEngineeringPipeline {
    
    @Autowired
    private SparkSession spark;
    
    @Autowired
    private FeatureStoreRepository featureStore;
    
    @Scheduled(cron = "0 0 * * *")  // Daily
    public void runFeatureEngineering() {
        // Step 1: Load Source Data (Distributed)
        Dataset<Row> rawData = spark.read()
            .parquet("s3://raw-data/events/");
        
        // Step 2: Apply Feature Transformations
        Dataset<Row> engineeredFeatures = engineFeatures(rawData);
        
        // Step 3: Write to Feature Store
        engineeredFeatures.write()
            .mode(SaveMode.Overwrite)
            .parquet("s3://features/online-store/");
        
        logger.info("Feature Engineering Complete: {} rows processed",
            engineeredFeatures.count());
    }
    
    private Dataset<Row> engineFeatures(Dataset<Row> df) {
        return df
            // Time-Based Features
            .withColumn(
                "days_since_event",
                functions.datediff(functions.current_timestamp(),
                    functions.col("event_timestamp"))
            )
            
            // Aggregations
            .withColumn(
                "user_total_amount",
                functions.sum("amount").over(
                    Window.partitionBy("user_id")
                        .orderBy(functions.col("timestamp").cast("long"))
                        .rangeBetween(-86400*30*1000, 0)  // Last 30 Days
                )
            )
            
            // Complex Transformations
            .withColumn(
                "purchase_frequency_ratio",
                functions.col("purchase_count") / 
                    functions.greatest(functions.col("days_active"), 1)
            );
    }
}
```

### ✅ Scaling Strategies - From Pandas to Spark

```java
/*
SCALING DECISION TREE:

Data Size < 10 GB?
  → Use Pandas (Simple, Familiar)
  
10 GB < Data Size < 1 TB?
  → Use Spark (Balanced)
  
Data Size > 1 TB or Streaming?
  → Use Spark + Streaming
  
Need Real-Time (<1 Second)?
  → On-Demand Computation (Flink/Beam)
*/

@Configuration
public class FeatureComputationStrategy {
    
    enum ComputeStrategy {
        PANDAS,      // Single Machine, < 10 GB
        SPARK_BATCH, // Distributed, < 1 TB, Batch
        SPARK_STREAM // Continuous, Real-Time
    }
    
    public ComputeStrategy selectStrategy(long dataSize, boolean realTime) {
        if (realTime) {
            return ComputeStrategy.SPARK_STREAM;
        }
        
        if (dataSize < 10_000_000_000L) {  // 10 GB
            return ComputeStrategy.PANDAS;
        }
        
        return ComputeStrategy.SPARK_BATCH;
    }
}

// Strategy 1: PANDAS (Small Data)
@Service
public class PandasFeatureEngine {
    
    public DataFrame engineerFeatures(DataFrame df) {
        // Pandas: Simple, Readable, Familiar
        return df
            .assign(
                days_active = (pd.Timestamp.now() - df['signup_date']).dt.days,
                purchase_sum = df.groupby('user_id')['amount'].transform('sum'),
                purchase_avg = df.groupby('user_id')['amount'].transform('mean')
            )
    }
}

// Strategy 2: SPARK BATCH (Medium Data)
@Service
public class SparkBatchFeatureEngine {
    
    public Dataset<Row> engineerFeatures(Dataset<Row> df) {
        // Spark: Distributed, Scalable, Fault-Tolerant
        return df
            .withColumn("days_active", 
                functions.datediff(functions.current_timestamp(),
                    functions.col("signup_date")))
            .withColumn("purchase_sum",
                functions.sum("amount").over(
                    Window.partitionBy("user_id")))
    }
}

// Strategy 3: SPARK STREAMING (Real-Time)
@Service
public class SparkStreamingFeatureEngine {
    
    public void engineerStreamingFeatures() {
        StreamingQuery query = spark
            .readStream()
            .kafka("localhost:9092", subscribe="events")
            .selectExpr("CAST(value AS STRING)")
            .select(functions.from_json(functions.col("value"), schema))
            .writeStream()
            .outputMode("append")
            .format("kafka")
            .option("topic", "features")
            .option("checkpointLocation", "/tmp/checkpoint")
            .start();
        
        query.awaitTermination();
    }
}
```

### ✅ Real-World Example - E-Commerce Feature Pipeline

```java
@Service
public class ECommerceFeaturePipeline {
    
    @Autowired
    private SparkSession spark;
    
    @Scheduled(cron = "0 2 * * *")  // 2 AM Daily
    public void computeUserFeatures() {
        // Load 500M Daily Events from S3
        Dataset<Row> events = spark.read()
            .parquet("s3://events/daily/");
        
        // Engineer Features
        Dataset<Row> features = events
            // Time Window Features
            .withColumn("timestamp_unix", 
                functions.col("timestamp").cast("long"))
            .withColumn("purchase_last_7d",
                functions.sum("amount").over(
                    Window.partitionBy("user_id")
                        .orderBy(functions.col("timestamp_unix"))
                        .rangeBetween(-604800, 0)
                ))
            .withColumn("purchase_last_30d",
                functions.sum("amount").over(
                    Window.partitionBy("user_id")
                        .orderBy(functions.col("timestamp_unix"))
                        .rangeBetween(-2592000, 0)
                ))
            
            // Categorical Features
            .withColumn("favorite_category",
                functions.max_by("category", "amount").over(
                    Window.partitionBy("user_id")
                ))
            
            // Behavioral Features
            .withColumn("purchase_frequency",
                functions.count("*").over(
                    Window.partitionBy("user_id")
                        .orderBy(functions.col("timestamp_unix"))
                        .rangeBetween(-2592000, 0)
                ))
            
            // Aggregate to User Level
            .groupBy("user_id")
            .agg(
                functions.max("purchase_last_7d").alias("purchase_7d_total"),
                functions.max("purchase_last_30d").alias("purchase_30d_total"),
                functions.first("favorite_category").alias("top_category"),
                functions.max("purchase_frequency").alias("monthly_purchases")
            );
        
        // Write to Online Feature Store (Redis/DynamoDB)
        features.write()
            .mode(SaveMode.Overwrite)
            .option("header", "true")
            .parquet("s3://feature-store/online/");
        
        logger.info("Feature computation complete: {} users", features.count());
    }
}

// Performance:
// - Pandas (Notebook): 500M rows = OOM ❌
// - Spark (60 Executors): 500M rows = 60 seconds ✅
// - Scaling: Linear (2x Executors = 2x Faster)
```

## 💡 Why This Matters

For data scientists to scale single-machine code to a multi-node cluster, Dask mirrors Pandas and NumPy APIs, allowing data scientists to swap the scheduler with minimal changes. Spark's strength is structured, production ETL pipelines where schema enforcement, Spark SQL semantics, and fault tolerance at petabyte scale matter. Notebook code doesn't scale—distributed systems require architectural changes.

## 🎯 Key Takeaway

Use Pandas for notebooks and small data (<10 GB). Migrate to Spark for production and large datasets. Spark SQL looks like Pandas but scales to petabytes. Window functions replace groupby tricks. Write once, scale infinitely—same feature logic works at any scale.

---

**Tags:** `#Java` `#JavaWisdom` `#FeatureEngineering` `#Spark` `#ApacheSpark` `#DistributedComputing` `#Pandas` `#DataPipelines` `#MLOps` `#Scalability` `#SpringBoot` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity` `#SoftwareEngineering`
