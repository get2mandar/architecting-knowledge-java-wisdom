# Post #79: Vector Databases and Semantic Search - Building RAG Foundations
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** November 4, 2026  
**Topic:** Vector Databases, Embeddings, Semantic Search, Spring AI, RAG Foundations
 
---
 
## The Problem
 
A support engineer searches the knowledge base for "customer can't log in after password reset." The relevant article is titled "Session invalidation following credential rotation." Keyword search returns nothing, because the words don't match even though the meaning does. Retrieval-Augmented Generation depends on finding the right context first, and keyword matching cannot find meaning. Semantic search can, and vector databases are the infrastructure behind it.
 
## Code Example
 
### ❌ Keyword Search - Matches Words, Misses Meaning
 
```java
// SQL LIKE Matching on Literal Text
List<Article> results = jdbc.query(
    "SELECT * FROM articles WHERE content LIKE ?",
    articleMapper,
    "%can't log in%"
);
 
// Failure Scenario:
// 1. Query Words Differ From Document Words
// 2. Relevant Article Never Surfaces
// 3. An LLM Given Poor Context Produces Confident Wrong Answers
```
 
### ✅ Solution 1: Embeddings - Meaning as Numbers
 
```java
/*
EMBEDDING:
  - Text Becomes a Vector; Similar Meaning = Nearby Vectors
*/
@Service
public class EmbeddingService {
 
    private final EmbeddingModel embeddingModel;  // Spring AI Abstraction
 
    public float[] embed(String text) {
        return embeddingModel.embed(text);  // e.g. 1536 Dimensions
    }
}
 
// "can't log in after password reset" and "session invalidation after
// credential rotation" Land Close Together in Vector Space
```
 
### ✅ Solution 2: Vector Store - Similarity Search
 
```java
@Service
public class KnowledgeSearchService {
 
    private final VectorStore vectorStore;
 
    public void index(Article article) {
        vectorStore.add(List.of(new Document(article.content(),
            Map.of("articleId", article.id(), "category", article.category()))));
    }
 
    public List<Document> search(String question) {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(question)
                .topK(5)                      // Five Nearest Neighbours
                .similarityThreshold(0.75)    // Ignore Weak Matches
                .build()
        );
    }
}
```
 
### ✅ Solution 3: Chunking - Retrieval Quality Starts at Ingestion
 
```java
public List<Document> prepare(Article article) {
    // Whole Articles Dilute Meaning; Focused Chunks Retrieve Precisely
    TokenTextSplitter splitter = new TokenTextSplitter(400, 100, 5, 10000, true);
    return splitter.apply(List.of(new Document(article.content(),
        Map.of("articleId", article.id()))));
}
 
// Chunk Size Is a Trade-off:
// Too Large -> Noisy Context, Diluted Embeddings
// Too Small -> Lost Context, Fragmented Answers
```
 
## Why This Matters
 
RAG quality is capped by retrieval quality. If the right passage never reaches the model, no prompt can rescue the answer. Embeddings let the system match intent rather than vocabulary, and a vector store makes nearest-neighbour search fast enough for production through approximate indexes. Chunking is the quiet decision with the biggest impact: it determines what each vector actually represents. Treat ingestion as an architectural concern, not a preprocessing afterthought.
 
## Key Takeaway
 
Retrieval is the foundation of RAG. Use embeddings to match meaning, a vector store to search at scale, and deliberate chunking to keep each vector focused. Tune topK and similarity thresholds against real queries, because the best model cannot compensate for the wrong context.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#VectorDatabase` `#SemanticSearch` `#RAG` `#Embeddings` `#SpringAI` `#SpringBoot` `#DistributedSystems` `#Microservices` `#GenAI` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
