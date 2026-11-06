# Post #80: RAG Pipelines in Production - Prompt Engineering and Retrieval Strategies
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** November 8, 2026  
**Topic:** RAG Pipelines, Prompt Engineering, Hybrid Retrieval, Re-ranking, Grounded Answers
 
---
 
## The Problem
 
The RAG prototype impressed everyone in the demo. In production, users report answers that sound authoritative but cite policies that don't exist, or ignore the document that clearly holds the answer. The model was fine. The pipeline around it wasn't: retrieval returned loosely related chunks, the prompt gave no rules for missing information, and nothing verified the answer against its sources. Production RAG is a pipeline-engineering problem, not a prompt-writing exercise.
 
## Code Example
 
### ❌ Naive RAG - Retrieve, Concatenate, Hope
 
```java
public String answer(String question) {
    List<Document> docs = vectorStore.similaritySearch(question);
    String context = docs.stream().map(Document::getText).collect(Collectors.joining("\n"));
    return chatClient.prompt("Answer this: " + question + "\n" + context).call().content();
}
 
// Failure Scenario:
// 1. Top Results Are Similar Text, Not Necessarily Relevant Text
// 2. No Instruction for "Context Doesn't Contain the Answer"
// 3. Model Fills the Gap With Plausible Invention
// 4. No Way to Trace an Answer Back to a Source
```
 
### ✅ Solution 1: Grounded Prompt - Rules for the Model
 
```java
private static final String SYSTEM_PROMPT = """
    You answer using ONLY the provided context.
    If the context does not contain the answer, reply: "I could not find this in the knowledge base."
    Cite the source id for every claim, like [doc-12].
    """;
 
public String answer(String question, List<Document> context) {
    return chatClient.prompt()
        .system(SYSTEM_PROMPT)
        .user(u -> u.text("Context:\n{context}\n\nQuestion: {question}")
            .param("context", format(context))
            .param("question", question))
        .call().content();
}
```
 
### ✅ Solution 2: Hybrid Retrieval and Re-ranking
 
```java
public List<Document> retrieve(String question) {
    // Semantic Search Catches Meaning; Keyword Search Catches Exact Terms
    List<Document> semantic = vectorStore.similaritySearch(question);
    List<Document> keyword  = keywordIndex.search(question, 20);
 
    List<Document> merged = reciprocalRankFusion(semantic, keyword);
 
    // Second Pass: A Re-ranker Scores Each Candidate Against the Question
    return reranker.rerank(question, merged).stream().limit(5).toList();
}
```
 
### ✅ Solution 3: Evaluate the Pipeline, Not Just the Model
 
```java
public record RagEvaluation(
    double retrievalHitRate,   // Was the Right Chunk in the Top K?
    double faithfulness,       // Is the Answer Supported by the Context?
    double answerRelevance     // Does It Address the Question?
) {}
 
// Run a Fixed Question Set on Every Pipeline Change:
// Chunk Size, Embedding Model, Prompt Wording, or topK
// A Regression in Hit Rate Explains Bad Answers Faster Than Prompt Tweaking Ever Will
```
 
## Why This Matters
 
Most bad RAG answers trace back to retrieval, not generation. Hybrid retrieval protects against the blind spots of either method alone, and re-ranking spends extra compute on a small candidate set to put the best evidence first. A grounded prompt gives the model an explicit exit when context is missing, which is where hallucination usually begins. Evaluation closes the loop: without measuring hit rate and faithfulness, every change to chunking, models, or prompts is a guess.
 
## Key Takeaway
 
Treat RAG as a pipeline with measurable stages. Retrieve with hybrid search and re-rank before generating, constrain the model with a grounded prompt that allows "I don't know" and requires citations, and evaluate every change against a fixed question set. Reliable answers come from the system around the model.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#RAG` `#PromptEngineering` `#HybridSearch` `#Reranking` `#SpringAI` `#GenAI` `#SpringBoot` `#DistributedSystems` `#Microservices` `#Reliability` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
