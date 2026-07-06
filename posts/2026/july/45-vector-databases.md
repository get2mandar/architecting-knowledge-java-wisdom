# Post #45: Vector Databases - Embedding Search with Spring AI

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 8, 2026  
**Topic:** Vector Databases, Embeddings, Similarity Search, pgvector, Pinecone

---

## The Problem

Traditional SQL can't find semantically similar documents. Vector databases solve this with embedding search.

## Code Example

### ❌ Keyword Search - Misses Meaning

```sql
-- Traditional Search: Exact Keywords Only
SELECT * FROM documents 
WHERE content LIKE '%machine learning%'
ORDER BY created_at DESC;

-- Problem: Misses "deep learning", "neural networks", "AI"
-- Finds: Irrelevant keyword matches
```

### ✅ Vector Search - Semantic Similarity

```sql
-- pgvector: Find Semantically Similar Documents
SELECT * FROM documents 
ORDER BY embedding <-> '[0.23, -0.41, 0.88, ..., 0.15]'::vector
LIMIT 10;

-- Finds: Semantically Related Content
-- "machine learning" AND "deep learning" AND "neural networks"
```

### ✅ Vector Database Basics - What Are Embeddings?

```java
// Text → Vector (1536 Dimensions for OpenAI)
String text = "Machine learning is a subset of artificial intelligence";
List<Double> embedding = embeddingModel.embed(text);
// Result: [-0.023, 0.041, -0.088, ..., 0.15]

// Similar Texts Have Similar Vectors
String similar = "Deep learning is part of machine learning";
List<Double> similarEmbedding = embeddingModel.embed(similar);
// Vector Distance: Small! (Cosine Similarity: 0.92)

String different = "Pizza is delicious food";
List<Double> differentEmbedding = embeddingModel.embed(different);
// Vector Distance: Large! (Cosine Similarity: 0.12)
```

### ✅ pgvector - Postgres with Vector Search

```sql
-- Enable pgvector Extension
CREATE EXTENSION vector;

-- Create Table with Vector Column
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    embedding vector(1536),  -- OpenAI Embedding Size
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create HNSW Index for Fast Search
CREATE INDEX ON documents 
USING hnsw (embedding vector_cosine_ops);

-- Insert Document with Embedding
INSERT INTO documents (content, embedding) VALUES (
    'Spring Boot is a powerful Java framework',
    '[0.023, -0.041, 0.088, ..., -0.15]'::vector
);

-- Similarity Search - Cosine Distance
SELECT 
    id, 
    content,
    1 - (embedding <=> query_vector) AS similarity
FROM documents
WHERE (embedding <=> query_vector) < 0.3  -- Top 10% Similar
ORDER BY embedding <=> query_vector
LIMIT 10;
```

### ✅ Pinecone - Managed Vector Database

```java
// pom.xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-pinecone</artifactId>
</dependency>

@Configuration
public class PineconeConfig {
    
    @Bean
    public EmbeddingStore<TextSegment> pineconeStore() {
        return PineconeEmbeddingStore.builder()
            .apiKey(System.getenv("PINECONE_API_KEY"))
            .index("documents")
            .namespace("production")
            .dimension(1536)
            .build();
    }
}

@Service
public class DocumentService {
    
    @Autowired
    private EmbeddingStore<TextSegment> embeddingStore;
    
    @Autowired
    private OpenAiEmbeddingModel embeddingModel;
    
    public void indexDocument(String documentId, String content) {
        // Create Embedding
        Response<Embedding> response = embeddingModel.embed(content);
        Embedding embedding = response.content();
        
        // Store in Pinecone
        TextSegment textSegment = new TextSegment(
            content,
            new Metadata("doc_id", documentId)
        );
        embeddingStore.add(embedding, textSegment);
    }
    
    public List<TextSegment> search(String query, int topK) {
        // Generate Query Embedding
        Response<Embedding> response = embeddingModel.embed(query);
        Embedding queryEmbedding = response.content();
        
        // Search Pinecone
        return embeddingStore.search(queryEmbedding, topK);
    }
}
```

### ✅ Spring Data with pgvector

```java
// pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-vector</artifactId>
</dependency>

@Entity
public class Document {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String content;
    
    @Column(columnDefinition = "vector(1536)")
    private PGvector embedding;  // pgvector Type
    
    @Convert(converter = JsonConverter.class)
    private Map<String, String> metadata;
}

@Repository
public interface DocumentRepository extends JpaRepository<Document, Long> {
    
    // Native Query for Vector Similarity Search
    @Query(value = """
        SELECT * FROM documents 
        ORDER BY embedding <=> CAST(:query AS vector)
        LIMIT :limit
        """, nativeQuery = true)
    List<Document> similarDocuments(
        @Param("query") String queryVector,
        @Param("limit") int limit
    );
}

@Service
public class DocumentService {
    
    @Autowired
    private DocumentRepository documentRepository;
    
    @Autowired
    private OpenAiEmbeddingModel embeddingModel;
    
    @Transactional
    public void indexDocument(String content) {
        // Generate Embedding
        Response<Embedding> response = embeddingModel.embed(content);
        List<Double> embedding = response.content().vector();
        
        // Save to Database
        Document doc = new Document();
        doc.setContent(content);
        doc.setEmbedding(new PGvector(embedding.stream()
            .mapToDouble(Double::doubleValue)
            .toArray()));
        
        documentRepository.save(doc);
    }
    
    public List<Document> search(String query, int topK) {
        // Generate Query Embedding
        Response<Embedding> response = embeddingModel.embed(query);
        List<Double> embedding = response.content().vector();
        
        // Convert to pgvector Format
        String vectorString = embedding.toString();
        
        // Search Database
        return documentRepository.similarDocuments(vectorString, topK);
    }
}
```

### ✅ Vector Search with Metadata Filtering

```java
// Search + Filter by Metadata
@Repository
public interface DocumentRepository extends JpaRepository<Document, Long> {
    
    @Query(value = """
        SELECT * FROM documents 
        WHERE metadata ->> 'category' = :category
        ORDER BY embedding <=> CAST(:query AS vector)
        LIMIT :limit
        """, nativeQuery = true)
    List<Document> searchByCategory(
        @Param("query") String queryVector,
        @Param("category") String category,
        @Param("limit") int limit
    );
    
    @Query(value = """
        SELECT * FROM documents 
        WHERE created_at > :since
        AND tenant_id = :tenantId
        ORDER BY embedding <=> CAST(:query AS vector)
        LIMIT :limit
        """, nativeQuery = true)
    List<Document> searchWithMetadata(
        @Param("query") String queryVector,
        @Param("since") LocalDateTime since,
        @Param("tenantId") String tenantId,
        @Param("limit") int limit
    );
}
```

### ✅ Real-World Example - Document RAG System

```java
@Configuration
public class RAGConfig {
    
    @Bean
    public EmbeddingStore<TextSegment> embeddingStore() {
        return new InMemoryEmbeddingStore<>();  // Or pgvector, Pinecone
    }
    
    @Bean
    public EmbeddingStoreIngestor embeddingStoreIngestor(
        OpenAiEmbeddingModel embeddingModel,
        EmbeddingStore<TextSegment> embeddingStore
    ) {
        return EmbeddingStoreIngestor.builder()
            .embeddingStore(embeddingStore)
            .embeddingModel(embeddingModel)
            .splitter(DocumentSplitters.recursive(300, 0))  // 300 Token Chunks
            .build();
    }
}

@Service
public class DocumentRAGService {
    
    @Autowired
    private EmbeddingStoreIngestor ingestor;
    
    @Autowired
    private EmbeddingStore<TextSegment> embeddingStore;
    
    @Autowired
    private OpenAiEmbeddingModel embeddingModel;
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    // Index Documents
    public void indexPDF(String filePath) {
        Document document = Document.from(
            filePath,
            new FileSystemDocumentLoader()
        );
        
        ingestor.ingest(document);
    }
    
    // Search and Generate Answer
    public String answerQuestion(String question) {
        // 1. Generate Query Embedding
        Response<Embedding> queryEmbedding = embeddingModel.embed(question);
        
        // 2. Search Similar Documents
        List<TextSegment> relevant = embeddingStore.search(
            queryEmbedding.content(),
            5  // Top 5
        );
        
        // 3. Build Context
        String context = relevant.stream()
            .map(TextSegment::text)
            .collect(Collectors.joining("\n\n---\n\n"));
        
        // 4. Generate Answer with Context
        String prompt = String.format("""
            Use the following documents to answer the question.
            
            Documents:
            %s
            
            Question: %s
            
            Answer:
            """, context, question);
        
        AiMessage response = chatModel.generate(
            new UserMessage(prompt)
        );
        
        return response.text();
    }
}

@RestController
public class RAGController {
    
    @Autowired
    private DocumentRAGService ragService;
    
    @PostMapping("/index")
    public ResponseEntity<String> indexDocument(@RequestParam String filePath) {
        ragService.indexPDF(filePath);
        return ResponseEntity.ok("Document indexed");
    }
    
    @PostMapping("/ask")
    public ResponseEntity<String> askQuestion(@RequestBody Map<String, String> request) {
        String answer = ragService.answerQuestion(request.get("question"));
        return ResponseEntity.ok(answer);
    }
}
```

### ✅ Choosing the Right Vector Database

```java
/*
pgvector:
  - Best for: < 10M vectors, ACID needed, cost-sensitive
  - Pros: One database, SQL joins, ACID transactions
  - Cons: Limited scale, slower than dedicated VectorDBs
  - Use case: Internal tools, small-scale RAG

Pinecone:
  - Best for: > 100M vectors, auto-scaling, simplicity
  - Pros: Fully managed, global scale, simple API
  - Cons: Vendor lock-in, expensive at scale
  - Use case: Production SaaS, billions of vectors

Weaviate:
  - Best for: Hybrid search, rich features, flexibility
  - Pros: Hybrid BM25+vector, GraphQL, modules
  - Cons: Complex setup, steep learning curve
  - Use case: Enterprise search, hybrid capabilities

Milvus:
  - Best for: High throughput, GPU acceleration
  - Pros: Distributed, fast, cost-effective
  - Cons: Operational complexity
  - Use case: Large-scale production deployments
*/
```

## 💡 Why This Matters

Vector databases enable semantic search - finding conceptually related documents, not just keyword matches. Embeddings are numerical representations of meaning - similar concepts have similar vectors. pgvector lets you stay in PostgreSQL for both relational and vector data. Pinecone provides fully managed serverless vector search at any scale. Metadata filtering combines vector similarity with traditional SQL filtering. Chunking strategies impact retrieval quality - proper splitting is critical. Embedding model choice affects search quality - OpenAI embeddings are industry standard.

## 🎯 Key Takeaway

Use vectors for semantic search in RAG systems. pgvector for small-scale with ACID requirements. Pinecone for production scale without ops overhead. Chunk documents strategically (300-500 tokens). Index documents at startup or incrementally. Always include metadata for filtering. Combine vector similarity with traditional filters for best results.

---

**Tags:** `#Java` `#JavaWisdom` `#VectorDatabase` `#pgvector` `#Pinecone` `#Embeddings` `#RAG` `#SemanticSearch` `#SpringBoot` `#GenerativeAI` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity`
