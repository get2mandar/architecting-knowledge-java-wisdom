# Post #47: RAG Pattern - Retrieval-Augmented Generation in Spring

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 15, 2026  
**Topic:** RAG, Retrieval-Augmented Generation, Document Chunking, Spring AI

---

## The Problem

LLMs hallucinate. Fine-tuning is expensive. RAG gives LLMs your data without retraining.

## Code Example

### ❌ Without RAG - Hallucination Risk

```java
String question = "What is our return policy?";

String answer = chatModel.generate(
    new UserMessage(question)
);
// "Our return policy allows returns within 30 days..."
// Problem: LLM Doesn't Know Your Policy!
// Answer Is Plausible but Potentially Wrong
```

### ✅ With RAG - Grounded in Your Data

```java
String question = "What is our return policy?";

// 1. Retrieve Relevant Documents
List<Document> relevantDocs = embeddingStore.search(question, 3);

// 2. Build Context from Documents
String context = relevantDocs.stream()
    .map(Document::getText)
    .collect(Collectors.joining("\n---\n"));

// 3. Prompt LLM with Context
String prompt = String.format("""
    Use this context to answer the question:
    %s
    
    Question: %s
    """, context, question);

String answer = chatModel.generate(new UserMessage(prompt));
// Answer Grounded in Your Actual Policy
```

### ✅ RAG Pipeline - Three Phases

```java
/*
Phase 1: INDEXING (Offline, Once)
  - Load Documents
  - Split into Chunks
  - Generate Embeddings
  - Store in Vector DB

Phase 2: RETRIEVAL (Online, Per Query)
  - Generate Query Embedding
  - Find Similar Chunks
  - Optionally Rerank
  - Collect Context

Phase 3: GENERATION (Online, Per Query)
  - Build Prompt with Context
  - Call LLM
  - Return Grounded Answer
*/
```

### ✅ Phase 1: Document Indexing

```java
@Configuration
public class RAGIndexingConfig {
    
    @Bean
    public DocumentLoaderFactory documentLoader() {
        return new DocumentLoaderFactory();
    }
}

@Service
public class DocumentIndexingService {
    
    @Autowired
    private EmbeddingStoreIngestor ingestor;
    
    @Autowired
    private OpenAiEmbeddingModel embeddingModel;
    
    public void indexDocuments(String documentPath) {
        // 1. Load Documents
        Document document = Document.from(
            documentPath,
            new FileSystemDocumentLoader()
        );
        
        System.out.println("Loaded: " + document.metadata().asMap());
        
        // 2. Ingest (Split + Embed + Store)
        ingestor.ingest(document);
        
        System.out.println("Documents indexed and stored");
    }
}

// application.properties
spring.ai.openai.embedding.model=text-embedding-3-small
spring.ai.embedding.chunk.size=300
spring.ai.embedding.chunk.overlap.size=50
```

### ✅ Chunking Strategy - Critical for Quality

```java
@Service
public class DocumentProcessingService {
    
    // Document Splitter with Overlap
    private static final DocumentSplitter SPLITTER = 
        DocumentSplitters.recursive(
            300,    // Chunk Size: 300 Tokens
            50      // Overlap: 50 Tokens (Context Preservation)
        );
    
    public void processDocument(String filePath) {
        Document document = Document.from(filePath);
        
        // Split Document into Chunks
        List<Document> chunks = SPLITTER.split(document);
        
        System.out.println("Original: " + 
            document.text().length() + " chars");
        System.out.println("Chunks: " + chunks.size());
        
        // Each Chunk Retains Original Metadata + Position
        chunks.forEach(chunk -> {
            System.out.println("Chunk: " + 
                chunk.text().substring(0, 50) + "...");
            System.out.println("Metadata: " + 
                chunk.metadata().asMap());
        });
    }
}

// Chunking Trade-offs:
// - Too Small (100 tokens): Lost Context, Many Chunks
// - Too Large (1000 tokens): Context Window Waste, Poor Ranking
// - Optimal (300-500 tokens): Balance Context & Precision
```

### ✅ Phase 2: Document Retrieval

```java
@Service
public class DocumentRetrievalService {
    
    @Autowired
    private EmbeddingStore<TextSegment> embeddingStore;
    
    @Autowired
    private OpenAiEmbeddingModel embeddingModel;
    
    public List<TextSegment> retrieveRelevantDocuments(
        String query, 
        int topK
    ) {
        // 1. Generate Query Embedding
        Response<Embedding> response = embeddingModel.embed(query);
        Embedding queryEmbedding = response.content();
        
        // 2. Search Vector Store (Cosine Similarity)
        List<TextSegment> results = embeddingStore.search(
            queryEmbedding, 
            topK  // Top 3-5 Usually Enough
        );
        
        return results;
    }
    
    // Advanced: Reranking for Better Quality
    public List<TextSegment> retrieveWithReranking(
        String query,
        int topK
    ) {
        // 1. Retrieve More Than Needed (10x)
        List<TextSegment> candidates = retrieveRelevantDocuments(query, topK * 10);
        
        // 2. Rerank Using Cross-Encoder Model
        List<TextSegment> reranked = rerankingService.rerank(
            query,
            candidates,
            topK
        );
        
        return reranked;  // Top-K Highest Quality
    }
}

// Similarity Metrics:
// - Cosine: Best for Semantic Similarity
// - L2: Euclidean Distance
// - Dot Product: Fast, Use with Normalized Vectors
```

### ✅ Phase 3: Answer Generation with Context

```java
@Service
public class RAGService {
    
    @Autowired
    private DocumentRetrievalService retrieval;
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    public String answerQuestion(String question) {
        // 1. Retrieve Context
        List<TextSegment> relevantDocs = 
            retrieval.retrieveRelevantDocuments(question, 3);
        
        // 2. Build Context String
        String context = relevantDocs.stream()
            .map(TextSegment::text)
            .collect(Collectors.joining("\n\n---\n\n"));
        
        // 3. Craft Prompt with Context + Question
        String systemPrompt = """
            You are a helpful assistant.
            Answer the user's question using ONLY the provided context.
            If the context doesn't contain the answer, say "I don't know".
            """;
        
        String userPrompt = String.format("""
            Context:
            %s
            
            Question: %s
            
            Answer:
            """, context, question);
        
        // 4. Generate Answer
        AiMessage response = chatModel.generate(
            new UserMessage(userPrompt),
            new SystemMessage(systemPrompt)
        );
        
        return response.text();
    }
}

@RestController
public class RAGController {
    
    @Autowired
    private RAGService ragService;
    
    @PostMapping("/ask")
    public ResponseEntity<String> ask(@RequestBody Map<String, String> request) {
        String answer = ragService.answerQuestion(request.get("question"));
        return ResponseEntity.ok(answer);
    }
}
```

### ✅ Real-World Example - Company Knowledge Base RAG

```java
@Configuration
public class CompanyKBConfig {
    
    @Bean
    public EmbeddingStore<TextSegment> embeddingStore() {
        // Development: In-Memory Store
        return new InMemoryEmbeddingStore<>();
        
        // Production: Use Persistent Store
        // return new PgvectorEmbeddingStore(dataSource);
        // return new PineconeEmbeddingStore(apiKey);
    }
    
    @Bean
    public EmbeddingStoreIngestor ingestor(
        OpenAiEmbeddingModel embeddingModel,
        EmbeddingStore<TextSegment> store
    ) {
        return EmbeddingStoreIngestor.builder()
            .embeddingStore(store)
            .embeddingModel(embeddingModel)
            .splitter(DocumentSplitters.recursive(300, 50))
            .build();
    }
}

@Service
public class CompanyKBService {
    
    @Autowired
    private EmbeddingStoreIngestor ingestor;
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    @Autowired
    private DocumentRetrievalService retrieval;
    
    @PostConstruct
    public void initializeKnowledgeBase() {
        // Index All Company Documents on Startup
        Document policies = Document.from("docs/policies.pdf");
        Document procedures = Document.from("docs/procedures.pdf");
        Document faq = Document.from("docs/faq.md");
        
        ingestor.ingest(policies, procedures, faq);
    }
    
    public String answerEmployeeQuestion(String question) {
        // Retrieve Relevant Company Docs
        List<TextSegment> docs = retrieval
            .retrieveWithReranking(question, 3);
        
        String context = docs.stream()
            .map(TextSegment::text)
            .collect(Collectors.joining("\n\n"));
        
        // Generate Answer from Policy
        String prompt = String.format("""
            You are an HR assistant. Answer using company policies:
            
            Policies:
            %s
            
            Question: %s
            """, context, question);
        
        return chatModel.generate(new UserMessage(prompt)).text();
    }
}
```

### ✅ Advanced RAG - Metadata Filtering

```java
public record DocumentWithMetadata(
    String content,
    Map<String, String> metadata  // source, author, date, category
) {}

@Service
public class AdvancedRAGService {
    
    public List<TextSegment> searchByCategory(
        String query,
        String category
    ) {
        // Retrieve Then Filter by Metadata
        List<TextSegment> allResults = retrieval
            .retrieveRelevantDocuments(query, 10);
        
        // Filter by Category
        return allResults.stream()
            .filter(doc -> category.equals(
                doc.metadata().get("category")
            ))
            .limit(3)
            .collect(Collectors.toList());
    }
    
    public List<TextSegment> searchWithinDateRange(
        String query,
        LocalDate startDate,
        LocalDate endDate
    ) {
        List<TextSegment> results = retrieval
            .retrieveRelevantDocuments(query, 10);
        
        return results.stream()
            .filter(doc -> {
                LocalDate docDate = LocalDate.parse(
                    doc.metadata().get("date")
                );
                return !docDate.isBefore(startDate) && 
                       !docDate.isAfter(endDate);
            })
            .limit(3)
            .collect(Collectors.toList());
    }
}
```

### ✅ Evaluation - RAGAS Score

```java
@Service
public class RAGEvaluationService {
    
    public record EvaluationResult(
        Double contextRelevance,    // 0-1: Is Context Relevant?
        Double faithfulness,        // 0-1: Is Answer Grounded?
        Double answerRelevance,     // 0-1: Does Answer Help?
        Double averageScore         // 0-1: Overall RAG Quality
    ) {}
    
    public EvaluationResult evaluateRAG(
        String question,
        String generatedAnswer,
        List<TextSegment> retrievedContext
    ) {
        // Using RAGAS (Retrieval-Augmented Generation Assessment)
        Double relevance = evaluateContextRelevance(
            question, 
            retrievedContext
        );
        
        Double faithfulness = evaluateFaithfulness(
            generatedAnswer, 
            retrievedContext
        );
        
        Double answerRelevance = evaluateAnswerRelevance(
            question, 
            generatedAnswer
        );
        
        Double average = (relevance + faithfulness + answerRelevance) / 3;
        
        return new EvaluationResult(
            relevance, 
            faithfulness, 
            answerRelevance, 
            average
        );
    }
}

// RAG Quality Metrics:
// Context Relevance: Does Retrieved Doc Match Query? (0.7+ Good)
// Faithfulness: Is Answer Supported by Context? (0.8+ Good)
// Answer Relevance: Does Answer Help with Question? (0.7+ Good)
// Overall: Average of Above (0.75+ Production Ready)
```

## 💡 Why This Matters

RAG enables LLMs to use current, domain-specific data without retraining. Chunking strategy determines retrieval quality - overlap preserves context. Embedding model choice affects semantic understanding. Vector database selection impacts scale, latency, cost. Reranking improves quality - more expensive, better results. Metadata filtering enables fine-grained access control. RAGAS evaluation quantifies RAG quality. Spring AI integrates RAG seamlessly - declarative, type-safe.

## 🎯 Key Takeaway

RAG is the dominant pattern for production LLM applications. Chunk documents carefully (300-500 tokens with overlap). Retrieve 3-5 top documents, optionally rerank. Always include context in system prompt. Evaluate RAG with RAGAS scores. Start with pgvector/in-memory, scale to Pinecone as needed.

---

**Tags:** `#Java` `#JavaWisdom` `#RAG` `#RetrievalAugmentedGeneration` `#SpringAI` `#VectorDatabase` `#LLM` `#GenerativeAI` `#Embeddings` `#DocumentRetrieval` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity`
