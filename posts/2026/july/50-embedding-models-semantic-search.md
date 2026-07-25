# Post #50: Embedding Models - Semantic Search with pgvector

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 26, 2026  
**Topic:** Embedding Models, Text-Embedding-3, Semantic Search, pgvector Integration

---

## The Problem

Not all embeddings are equal. Choosing the wrong model tanks search quality. Understanding embedding dimensions, cost, and accuracy trade-offs is critical.

## Code Example

### ❌ Poor Embedding Choice - Weak Semantic Understanding

```java
// Using Generic/Old Embedding Model
@Service
public class SearchService {
    
    @Autowired
    private OldEmbeddingModel embeddingModel;  // ada-002 (Outdated)
    
    public List<Document> search(String query) {
        // Generate Embedding
        List<Double> queryEmbedding = embeddingModel.embed(query);
        
        // Search pgvector
        return documentRepository.similarDocuments(queryEmbedding, 5);
        
        // Problem: Lower Semantic Quality
        // Query: "machine learning" finds "deep learning" weakly
        // Accuracy: ~61% (MTEB Benchmark)
    }
}
```

### ✅ Modern Embedding Models - text-embedding-3

```java
/*
EMBEDDING MODELS LANDSCAPE (2026):

OpenAI text-embedding-3-small:
  - Dimensions: 1536 (configurable down to 256)
  - Speed: Fast
  - Cost: 5x Cheaper Than Ada-002
  - Quality: 62.3% MTEB (vs 61.0% Ada)
  - Use: Cost-Sensitive, Latency Critical
  
OpenAI text-embedding-3-large:
  - Dimensions: 3072 (configurable)
  - Speed: Slower
  - Cost: Moderate
  - Quality: 64.6% MTEB (Best Commercial)
  - Use: High Accuracy Required
  
Open-Source BGE-M3 (BAAI):
  - Dimensions: 1024
  - Speed: Very Fast (Self-Hosted)
  - Cost: Free (Self-Hosted)
  - Quality: 64.8% MTEB (Best Overall)
  - Use: Privacy Critical, Cost Control
  
Voyage AI voyage-3-large:
  - Dimensions: 1024
  - Speed: Fast
  - Cost: Similar to OpenAI
  - Quality: 64.5% MTEB
  - Use: Enterprise, Retrieval-Focused
*/
```

### ✅ Spring AI Integration - Embedding Model Selection

```java
// pom.xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>

// application.properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.embedding.model=text-embedding-3-small  // or 3-large
spring.ai.openai.embedding.encoding-format=float

@Configuration
public class EmbeddingConfig {
    
    @Bean
    public EmbeddingModel embeddingModel(OpenAiEmbeddingModel model) {
        return model;  // Configured from Properties
    }
}

@Service
public class SemanticSearchService {
    
    @Autowired
    private EmbeddingModel embeddingModel;
    
    public List<Document> semanticSearch(String query, int topK) {
        // Generate Embedding Using text-embedding-3-small
        Response<Embedding> response = embeddingModel.embed(query);
        Embedding embedding = response.content();
        
        // Embedding: 1536 Dimensions of Semantic Meaning
        List<Double> vector = embedding.getOutput();
        
        return documentRepository.findSimilar(vector, topK);
    }
}
```

### ✅ Understanding Embedding Dimensions

```java
/*
DIMENSIONS MATTER:

Smaller Dimensions (128, 256):
  - Pros: Smaller Storage, Faster Queries
  - Cons: Less Semantic Nuance
  - Use: High-Volume Retrieval, Memory Constrained

Medium Dimensions (768, 1024):
  - Pros: Good Balance, Fast, Fine-Grained Meaning
  - Cons: Medium Storage
  - Use: Most RAG Applications
  
Large Dimensions (1536, 3072):
  - Pros: Captures Fine Nuances, Best Accuracy
  - Cons: More Storage, Slower Queries
  - Use: Complex Semantic Tasks, Quality Critical
*/

@Service
public class AdaptiveEmbeddingService {
    
    public List<Double> embedWithConfigurableDimensions(
        String text,
        Integer targetDimensions  // 256, 512, 1024, 1536, etc.
    ) {
        // text-embedding-3 Supports Dimension Reduction!
        Response<Embedding> response = embeddingModel.embed(
            EmbeddingRequest.builder()
                .input(text)
                .model("text-embedding-3-large")
                .dimensions(targetDimensions)  // Reduce from 3072 to Target
                .build()
        );
        
        return response.content().getOutput();
    }
}
```

### ✅ Semantic Search Pipeline - Complete Example

```java
@Service
public class ComprehensiveSemanticSearch {
    
    @Autowired
    private EmbeddingModel embeddingModel;
    
    @Autowired
    private DocumentRepository documentRepository;
    
    // Phase 1: Index Documents (Offline)
    public void indexDocuments(List<String> documents) {
        for (String doc : documents) {
            // Generate Embedding
            Response<Embedding> response = embeddingModel.embed(doc);
            List<Double> embedding = response.content().getOutput();
            
            // Store in pgvector
            Document entity = new Document();
            entity.setContent(doc);
            entity.setEmbedding(new PGvector(
                embedding.stream()
                    .mapToDouble(Double::doubleValue)
                    .toArray()
            ));
            
            documentRepository.save(entity);
        }
    }
    
    // Phase 2: Semantic Search (Online)
    public List<Document> semanticSearch(String query, int topK) {
        // Generate Query Embedding
        Response<Embedding> queryEmbedding = embeddingModel.embed(query);
        List<Double> queryVector = queryEmbedding.content().getOutput();
        
        // Similarity Search Using Cosine Distance
        return documentRepository.searchBySimilarity(queryVector, topK);
    }
    
    // Phase 3: Hybrid Search - Combine Semantic + Keyword
    public List<Document> hybridSearch(
        String query,
        int topK
    ) {
        // Semantic Results
        List<Document> semanticResults = semanticSearch(query, topK);
        
        // Keyword Results
        List<Document> keywordResults = documentRepository
            .findByContentContainingIgnoreCase(query);
        
        // Combine and Rerank
        Set<Document> combined = new LinkedHashSet<>();
        combined.addAll(semanticResults);
        combined.addAll(keywordResults);
        
        return combined.stream()
            .limit(topK)
            .collect(Collectors.toList());
    }
}
```

### ✅ Similarity Metrics - Cosine vs L2 vs Dot Product

```java
/*
SIMILARITY METRICS:

Cosine Similarity (Recommended):
  - Formula: dot(A, B) / (||A|| * ||B||)
  - Range: -1 to +1 (1 = Same Direction)
  - Pros: Invariant to Scale, Semantic
  - Cons: Computation Cost (Negligible)
  
L2 Distance (Euclidean):
  - Formula: sqrt(sum((A_i - B_i)^2))
  - Range: 0 to Infinity (0 = Identical)
  - Pros: Intuitive, Geometric Meaning
  - Cons: Scale Dependent
  
Dot Product (Fast):
  - Formula: sum(A_i * B_i)
  - Range: -Infinity to +Infinity
  - Pros: Fastest (No Sqrt)
  - Cons: Scale Dependent, Not Normalized
*/

@Repository
public interface DocumentRepository extends JpaRepository<Document, Long> {
    
    // Cosine Similarity - Recommended for RAG
    @Query(value = """
        SELECT * FROM documents 
        ORDER BY embedding <-> CAST(:queryVector AS vector)
        LIMIT :limit
        """, nativeQuery = true)
    List<Document> searchByCosine(
        @Param("queryVector") String queryVector,
        @Param("limit") int limit
    );
    
    // L2 Distance
    @Query(value = """
        SELECT * FROM documents 
        ORDER BY embedding <#> CAST(:queryVector AS vector)
        LIMIT :limit
        """, nativeQuery = true)
    List<Document> searchByL2(
        @Param("queryVector") String queryVector,
        @Param("limit") int limit
    );
    
    // With Similarity Score
    @Query(value = """
        SELECT *, 
               1 - (embedding <-> CAST(:queryVector AS vector)) AS similarity
        FROM documents 
        ORDER BY embedding <-> CAST(:queryVector AS vector)
        LIMIT :limit
        """, nativeQuery = true)
    List<Map<String, Object>> searchWithScore(
        @Param("queryVector") String queryVector,
        @Param("limit") int limit
    );
}
```

### ✅ Real-World Example - Legal Document Search

```java
@Service
public class LegalDocumentSearch {
    
    @Autowired
    private EmbeddingModel embeddingModel;
    
    @Autowired
    private DocumentRepository documentRepository;
    
    public record SearchResult(
        String documentId,
        String content,
        Double similarityScore,
        String documentType  // "contract", "policy", "precedent"
    ) {}
    
    public List<SearchResult> findRelevantLegalDocuments(
        String query,
        String documentType  // Optional Filter
    ) {
        // 1. Generate Query Embedding
        Response<Embedding> response = embeddingModel.embed(
            query + " [LEGAL_DOMAIN]"  // Domain Hint
        );
        List<Double> queryVector = response.content().getOutput();
        
        // 2. Search with Type Filter
        List<Document> results = documentRepository
            .searchByTypeAndSimilarity(queryVector, documentType, 10);
        
        // 3. Transform to Results
        return results.stream()
            .map(doc -> new SearchResult(
                doc.getId(),
                doc.getContent(),
                calculateSimilarity(queryVector, doc.getEmbedding()),
                doc.getType()
            ))
            .collect(Collectors.toList());
    }
    
    private Double calculateSimilarity(List<Double> v1, List<Double> v2) {
        // Cosine Similarity
        double dotProduct = 0;
        double norm1 = 0;
        double norm2 = 0;
        
        for (int i = 0; i < v1.size(); i++) {
            dotProduct += v1.get(i) * v2.get(i);
            norm1 += v1.get(i) * v1.get(i);
            norm2 += v2.get(i) * v2.get(i);
        }
        
        return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
    }
}
```

### ✅ Cost Optimization - Choosing the Right Model

```java
@Service
public class CostOptimizedSearch {
    
    // Use text-embedding-3-small for Most Queries
    @Autowired
    private EmbeddingModel smallEmbedding;
    
    // Use text-embedding-3-large for Complex Semantic Tasks
    @Autowired
    private EmbeddingModel largeEmbedding;
    
    public List<Document> intelligentSearch(String query) {
        // Heuristic: Simple Queries Use Small Model (Cost: 80% Reduction)
        if (isSimpleQuery(query)) {
            return searchWithSmall(query);  // 5x Cheaper
        }
        
        // Complex Queries Use Large Model (Accuracy: 2-3% Better)
        return searchWithLarge(query);
    }
    
    private boolean isSimpleQuery(String query) {
        // Simple: Short, Common Terms
        return query.length() < 50 && !query.contains("complex");
    }
    
    private List<Document> searchWithSmall(String query) {
        // text-embedding-3-small: 1536 Dims, 5x Cheaper
        // Quality: 62.3% MTEB (Excellent for Most Use Cases)
        return performSearch(smallEmbedding, query);
    }
    
    private List<Document> searchWithLarge(String query) {
        // text-embedding-3-large: 3072 Dims, Best Accuracy
        // Quality: 64.6% MTEB (Top Commercial)
        return performSearch(largeEmbedding, query);
    }
}

// Cost Calculation:
// text-embedding-3-small: $0.00002 / 1K tokens
// text-embedding-3-large: $0.00013 / 1K tokens
// Using small for 80% of queries saves 80% on embedding costs!
```

## 💡 Why This Matters

text-embedding-3 models (small/large) outperform ada-002 on all benchmarks while being cheaper. Dimension selection trades off accuracy vs. storage and speed. Cosine similarity is the right metric for semantic search - scale invariant. Hybrid search combines semantic and keyword matching for robustness. Cost optimization matters at scale - 1M+ queries per day. Choosing embedding model is as critical as LLM choice for RAG quality.

## 🎯 Key Takeaway

Use text-embedding-3-small as default - 5x cheaper, excellent quality (62.3% MTEB). Use text-embedding-3-large only when semantic precision is critical. Always use cosine similarity for normalized vectors. Index with pgvector for production-grade semantic search. Hybrid search beats pure semantic or keyword alone.

---

**Tags:** `#Java` `#JavaWisdom` `#EmbeddingModels` `#SemanticSearch` `#pgvector` `#TextEmbedding3` `#OpenAI` `#RAG` `#VectorDatabase` `#SpringBoot` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity`
