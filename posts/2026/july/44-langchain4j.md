# Post #44: LangChain4j - Java Chains for AI Workflows

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 5, 2026  
**Topic:** LangChain4j, AI Agents, RAG, Memory Management

---

## The Problem

Building complex LLM workflows is repetitive. LangChain4j provides production-ready patterns for Java.

## Code Example

### ❌ Manual Chains - Boilerplate Everywhere

```java
// Without LangChain4j - Manual Everything
public class ManualChatBot {
    
    private OpenAiChatModel model;
    private List<Message> messageHistory;
    
    public String chat(String userMessage) {
        // 1. Manage Message History
        messageHistory.add(new UserMessage(userMessage));
        
        // 2. Build Request
        ChatRequest request = ChatRequest.builder()
            .messages(messageHistory)
            .model("gpt-4o")
            .build();
        
        // 3. Call API
        ChatResponse response = model.call(request);
        String assistantMessage = response.getFirstMessage().getContent();
        
        // 4. Update History
        messageHistory.add(new AssistantMessage(assistantMessage));
        
        // 5. Manage Memory Size
        if (messageHistory.size() > 20) {
            messageHistory.remove(0);  // Simple sliding window
        }
        
        return assistantMessage;
    }
    
    // Every Feature Needs Manual Implementation!
}
```

### ✅ LangChain4j - @AiService Annotation

```java
// pom.xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-spring-boot-starter</artifactId>
</dependency>

@AiService
public interface ChatBot {
    
    @SystemMessage("You are a helpful Java assistant.")
    @ChatMemory(chatMemoryProvider = "defaultChatMemoryProvider")
    String chat(String message);
}

// That's It! LangChain4j Handles:
// - Message History Management
// - API Calls
// - Memory Cleanup
// - Error Handling
```

### ✅ AI Services - Type-Safe Agent Definition

```java
@AiService
public interface SalesAssistant {
    
    @SystemMessage("""
        You are a professional sales assistant. Help customers find products.
        Always be helpful and provide product recommendations.
        """)
    @ChatMemoryProvider("salesMemory")
    String chat(String message);
}

@Configuration
public class AIConfig {
    
    @Bean
    public SalesAssistant salesAssistant(OpenAiChatModel model) {
        return AiServices.builder(SalesAssistant.class)
            .chatLanguageModel(model)
            .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
            .build();
    }
}

// Usage
@Service
public class SalesService {
    
    @Autowired
    private SalesAssistant assistant;
    
    public String handleCustomerQuestion(String question) {
        return assistant.chat(question);  // Memory Automatic!
    }
}
```

### ✅ Chat Memory - Automatic Context Management

```java
@AiService
public interface ContextualAssistant {
    
    @SystemMessage("You are a helpful assistant. Remember context from previous messages.")
    String chat(String message);
}

@Configuration
public class MemoryConfig {
    
    @Bean
    public ContextualAssistant assistant(OpenAiChatModel model) {
        return AiServices.builder(ContextualAssistant.class)
            .chatLanguageModel(model)
            // Different Memory Strategies
            .chatMemory(MessageWindowChatMemory.withMaxMessages(10))  // Last 10 Messages
            // OR
            .chatMemory(TokenWindowChatMemory.withMaxTokens(2000))  // Last 2000 Tokens
            // OR
            .chatMemory(new CustomChatMemory())  // Implement Custom Logic
            .build();
    }
}
```

### ✅ Tools/Functions - LLM Calling Java Methods

```java
@AiService
public interface WeatherAssistant {
    
    @SystemMessage("You are a weather assistant. Use available tools to answer weather questions.")
    String chat(String message);
}

// Define Tools
public class WeatherTools {
    
    @Tool("Get current temperature for a city")
    public String getTemperature(@Parameter("city") String city) {
        // Call Real Weather API
        return "New York: 72°F";
    }
    
    @Tool("Get weather forecast")
    public String getForecast(
        @Parameter("city") String city,
        @Parameter("days") int days
    ) {
        return "7-day forecast for " + city;
    }
}

@Configuration
public class WeatherConfig {
    
    @Bean
    public WeatherAssistant weatherAssistant(OpenAiChatModel model) {
        WeatherTools tools = new WeatherTools();
        
        return AiServices.builder(WeatherAssistant.class)
            .chatLanguageModel(model)
            .tools(tools)  // Register Tools!
            .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
            .build();
    }
}

// Usage
String response = weatherAssistant.chat("What's the weather in Paris for next 3 days?");
// LLM Automatically Calls getForecast("Paris", 3)
```

### ✅ RAG - Retrieval-Augmented Generation

```java
// Load Documents
Document document = Document.from(
    "Spring Boot is a powerful framework...",
    new Metadata("source", "spring-docs")
);

EmbeddingStoreIngestor ingestor = EmbeddingStoreIngestor.builder()
    .embeddingStore(embeddingStore)  // pgvector, Pinecone, etc.
    .embeddingModel(embeddingModel)  // OpenAI Embeddings
    .splitter(DocumentSplitters.recursive(300, 0))  // Chunk Documents
    .build();

ingestor.ingest(document);

@AiService
public interface SpringDocumentQA {
    
    @SystemMessage("""
        Answer questions using the provided Spring Boot documentation.
        If you don't know the answer, say so.
        """)
    String askQuestion(String question);
}

@Configuration
public class RAGConfig {
    
    @Bean
    public SpringDocumentQA documentQA(
        OpenAiChatModel chatModel,
        EmbeddingStore<TextSegment> embeddingStore,
        OpenAiEmbeddingModel embeddingModel
    ) {
        ContentRetriever retriever = EmbeddingStoreContentRetriever.builder()
            .embeddingStore(embeddingStore)
            .embeddingModel(embeddingModel)
            .maxResults(5)  // Top 5 Similar Documents
            .minScore(0.8)  // Relevance Threshold
            .build();
        
        return AiServices.builder(SpringDocumentQA.class)
            .chatLanguageModel(chatModel)
            .contentRetriever(retriever)  // Automatic RAG!
            .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
            .build();
    }
}

// Usage
String answer = documentQA.askQuestion("How do I configure a datasource in Spring Boot?");
// LLM Uses Retrieved Documents as Context!
```

### ✅ AI Agents - Multi-Step Reasoning

```java
@AiService
public interface CodeAssistant {
    
    @SystemMessage("""
        You are an expert Java developer assistant.
        Use available tools to analyze code and provide recommendations.
        Break down complex tasks into multiple steps.
        """)
    String analyzeCode(String code);
}

public class CodeTools {
    
    @Tool("Analyze code for potential bugs")
    public String analyzeBugs(String code) {
        // Complex Analysis Logic
        return "Potential issues: null pointer, unclosed resources";
    }
    
    @Tool("Suggest performance improvements")
    public String suggestOptimizations(String code) {
        return "Use StringBuilder for string concatenation in loops";
    }
    
    @Tool("Check code against best practices")
    public String checkBestPractices(String code) {
        return "Missing error handling for database connections";
    }
}

@Configuration
public class AgentConfig {
    
    @Bean
    public CodeAssistant codeAssistant(OpenAiChatModel model) {
        CodeTools tools = new CodeTools();
        
        return AiServices.builder(CodeAssistant.class)
            .chatLanguageModel(model)
            .tools(tools)
            .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
            .build();
    }
}

// Usage
String analysis = codeAssistant.analyzeCode(userCode);
// Agent Chains Multiple Tool Calls:
// 1. Analyze for Bugs
// 2. Suggest Optimizations
// 3. Check Best Practices
// 4. Synthesize Results
```

### ✅ Advanced RAG - Multiple Retrievers

```java
@AiService
public interface EnterpriseQA {
    
    String answerQuestion(String question);
}

@Configuration
public class AdvancedRAGConfig {
    
    @Bean
    public EnterpriseQA enterpriseQA(
        OpenAiChatModel chatModel,
        EmbeddingStore<TextSegment> vectorStore,
        OpenAiEmbeddingModel embeddingModel
    ) {
        // Retriever 1: Vector Similarity
        ContentRetriever vectorRetriever = EmbeddingStoreContentRetriever.builder()
            .embeddingStore(vectorStore)
            .embeddingModel(embeddingModel)
            .maxResults(5)
            .build();
        
        // Retriever 2: Web Search
        ContentRetriever webRetriever = WebSearchContentRetriever.builder()
            .apiKey(System.getenv("SEARCH_API_KEY"))
            .maxResults(3)
            .build();
        
        // Retriever 3: Database Query
        ContentRetriever sqlRetriever = SqlContentRetriever.builder()
            .dataSource(dataSource)
            .table("company_docs")
            .textColumn("content")
            .build();
        
        // Combine All Retrievers
        ContentRetriever combined = ContentAggregator.builder()
            .retrievers(vectorRetriever, webRetriever, sqlRetriever)
            .build();
        
        return AiServices.builder(EnterpriseQA.class)
            .chatLanguageModel(chatModel)
            .contentRetriever(combined)
            .chatMemory(MessageWindowChatMemory.withMaxMessages(10))
            .build();
    }
}
```

### ✅ Real-World Example - Customer Support Agent

```java
@AiService
public interface SupportAgent {
    
    @SystemMessage("""
        You are a professional customer support agent.
        Be empathetic and helpful.
        Use available tools to find solutions.
        Always provide ticket numbers for escalations.
        """)
    String handleRequest(String customerMessage);
}

public class SupportTools {
    
    @Tool("Search knowledge base")
    public String searchKB(String query) {
        // Search Company Knowledge Base
        return "Found 5 articles matching query";
    }
    
    @Tool("Create support ticket")
    public String createTicket(String subject, String description) {
        // Create in Jira/Zendesk
        return "Ticket #12345 created";
    }
    
    @Tool("Check order status")
    public String checkOrderStatus(@Parameter("orderId") String orderId) {
        // Call Order Service
        return "Order #" + orderId + " shipped on 2026-07-01";
    }
    
    @Tool("Escalate to human agent")
    public String escalate(String reason) {
        return "Escalated to senior agent. Reference: ESCL-9876";
    }
}

@Configuration
public class SupportConfig {
    
    @Bean
    public SupportAgent supportAgent(OpenAiChatModel model) {
        return AiServices.builder(SupportAgent.class)
            .chatLanguageModel(model)
            .tools(new SupportTools())
            .chatMemory(MessageWindowChatMemory.withMaxMessages(15))
            .build();
    }
}

@RestController
public class SupportController {
    
    @Autowired
    private SupportAgent agent;
    
    @PostMapping("/support")
    public ResponseEntity<String> handleSupport(@RequestBody SupportRequest request) {
        String response = agent.handleRequest(request.message());
        return ResponseEntity.ok(response);
    }
}
```

## 💡 Why This Matters

LangChain4j simplifies complex LLM workflows with @AiService annotation. Message history automatically managed via ChatMemory interface. Tools enable LLMs to call Java methods - extending AI capabilities. ContentRetriever pattern abstracts data sources - vector, web, SQL. RAG (Retrieval-Augmented Generation) provides context for accurate answers. Multi-tool agents enable multi-step reasoning - complex problem solving. Spring Boot integration means familiar dependency injection and configuration.

## 🎯 Key Takeaway

Use @AiService for AI workflows instead of manual API calls. Leverage ChatMemory for context awareness. Implement Tools to extend LLM capabilities. Use RAG for domain-specific knowledge. Build agents for complex multi-step reasoning. LangChain4j makes enterprise AI accessible in Java.

---

**Tags:** `#Java` `#JavaWisdom` `#LangChain4j` `#LLM` `#AIAgents` `#RAG` `#SpringBoot` `#GenerativeAI` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity`
