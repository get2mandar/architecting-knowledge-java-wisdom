# Post #43: Spring AI - Building LLM-Powered Applications

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 01, 2026  
**Topic:** Spring AI, Large Language Models, OpenAI, Anthropic Claude

---

## The Problem

Building AI apps used to require Python. Spring AI brings LLMs to Java with the Spring experience you love.

## Code Example

### ❌ Raw API Calls - Boilerplate Hell

```java
// Direct OpenAI API - Lots of Manual Work
public class RawOpenAIExample {
    
    public String chatWithGPT(String prompt) throws Exception {
        // Create HTTP Client
        HttpClient client = HttpClient.newHttpClient();
        
        // Build Request JSON
        String requestBody = """
            {
                "model": "gpt-4o",
                "messages": [
                    {"role": "user", "content": "%s"}
                ]
            }
            """.formatted(prompt);
        
        // Make HTTP Request
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.openai.com/v1/chat/completions"))
            .header("Authorization", "Bearer " + apiKey)
            .POST(HttpRequest.BodyPublishers.ofString(requestBody))
            .build();
        
        HttpResponse<String> response = client.send(request, 
            HttpResponse.BodyHandlers.ofString());
        
        // Parse JSON Response
        JsonParser parser = JsonParserFactory.getJsonParser();
        Map<String, ?> map = parser.parseMap(response.body());
        List<?> choices = (List<?>) map.get("choices");
        Map<String, ?> choice = (Map<String, ?>) choices.get(0);
        Map<String, ?> message = (Map<String, ?>) choice.get("message");
        
        return (String) message.get("content");
    }
}

// Problems:
// 1. Lots of Boilerplate
// 2. Manual JSON Parsing
// 3. Error Handling Everywhere
// 4. Hard to Switch Models
```

### ✅ Spring AI - Elegant and Spring-Like

```java
// pom.xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>

// application.properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o

@Service
public class ChatService {
    
    @Autowired
    private ChatClient chatClient;  // Injected by Spring!
    
    public String chat(String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .call()
            .content();  // That's It!
    }
}
```

### ✅ Spring AI Configuration - Multiple Providers

```java
// application.properties

# OpenAI
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.options.model=gpt-4o

# Anthropic Claude
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.anthropic.chat.options.model=claude-3-5-sonnet-20241022

# Google Gemini
spring.ai.google.gemini.api-key=${GOOGLE_API_KEY}
spring.ai.google.gemini.chat.options.model=gemini-1.5-pro
```

### ✅ ChatClient - Fluent API

```java
@Service
public class AdvancedChatService {
    
    @Autowired
    private ChatClient chatClient;
    
    public String answerQuestion(String question) {
        return chatClient.prompt()
            .system("You are a helpful Java expert.")  // System Prompt
            .user(question)
            .call()
            .content();
    }
    
    public String answerWithContext(String question, String context) {
        return chatClient.prompt()
            .system("Answer Based on Context")
            .user(umt -> umt
                .text("Context: " + context)
                .text("Question: " + question)
            )
            .call()
            .content();
    }
    
    public String chat(String... messages) {
        var promptBuilder = chatClient.prompt();
        
        for (String msg : messages) {
            promptBuilder.user(msg);
        }
        
        return promptBuilder.call().content();
    }
}
```

### ✅ Structured Output - Type-Safe Responses

```java
// Define Expected Output Structure
public record CodeReviewResult(
    String summary,
    List<Issue> issues,
    String recommendation
) {}

public record Issue(
    String line,
    String severity,
    String description
) {}

@Service
public class CodeReviewService {
    
    @Autowired
    private ChatClient chatClient;
    
    public CodeReviewResult reviewCode(String code) {
        return chatClient.prompt()
            .system("You are a code reviewer. Return JSON matching the CodeReviewResult structure.")
            .user("Review this code:\n" + code)
            .call()
            .parseResponse(CodeReviewResult.class);  // Automatic Parsing!
    }
}

// Usage
CodeReviewResult result = codeReviewService.reviewCode(myCode);
System.out.println("Summary: " + result.summary());
result.issues().forEach(issue -> 
    System.out.println("Issue: " + issue.description())
);
```

### ✅ Tool Calling - LLM Can Call Your Functions

```java
// Define Tools the LLM Can Call
public class WeatherTools {
    
    @Tool("Get Current Temperature")
    public String getCurrentTemp(@Parameter("city") String city) {
        // Actually Call Weather API
        return "New York: 72°F";
    }
    
    @Tool("Get Weather Forecast")
    public String getForecast(
        @Parameter("city") String city,
        @Parameter("days") int days
    ) {
        return "7-day forecast: Sunny, 75°F";
    }
}

@Service
public class WeatherAssistant {
    
    @Autowired
    private ChatClient chatClient;
    
    private final WeatherTools weatherTools = new WeatherTools();
    
    public String answerWeatherQuestion(String question) {
        return chatClient.prompt()
            .system("You are a weather assistant. Use available tools to answer questions.")
            .user(question)
            .functions("getCurrentTemp", "getForecast")  // Register Tools
            .call()
            .content();
    }
}

// Usage
assistant.answerWeatherQuestion("What's the weather in New York?");
// LLM Automatically Calls getCurrentTemp("New York") to Get Answer
```

### ✅ Streaming Responses - Real-Time Output

```java
@RestController
public class ChatController {
    
    @Autowired
    private ChatClient chatClient;
    
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamChat(@RequestParam String prompt) {
        return chatClient.prompt()
            .user(prompt)
            .stream()
            .content();  // Returns Flux<String> - Server-Sent Events!
    }
}

// Frontend
async function streamChat() {
    const response = await fetch('/stream?prompt=Explain React');
    const reader = response.body.getReader();
    
    while (true) {
        const {done, value} = await reader.read();
        if (done) break;
        process.stdout.write(new TextDecoder().decode(value));
    }
}
```

### ✅ Multi-Model Support - Switch Providers Easily

```java
@Configuration
public class AIConfig {
    
    @Bean("openaiClient")
    public ChatClient openaiClient(OpenAiChatModel chatModel) {
        return ChatClient.create(chatModel);
    }
    
    @Bean("claudeClient")
    public ChatClient claudeClient(AnthropicChatModel chatModel) {
        return ChatClient.create(chatModel);
    }
}

@Service
public class MultiModelService {
    
    @Autowired
    @Qualifier("openaiClient")
    private ChatClient openaiClient;
    
    @Autowired
    @Qualifier("claudeClient")
    private ChatClient claudeClient;
    
    public String compareModels(String prompt) {
        String gptResponse = openaiClient.prompt()
            .user(prompt)
            .call()
            .content();
        
        String claudeResponse = claudeClient.prompt()
            .user(prompt)
            .call()
            .content();
        
        return "GPT: " + gptResponse + "\nClaude: " + claudeResponse;
    }
    
    public String smartRoute(String prompt) {
        // Route Simple Questions to Fast Model
        if (prompt.length() < 50) {
            return openaiClient.prompt().user(prompt).call().content();
        }
        
        // Route Complex Questions to Powerful Model
        return claudeClient.prompt().user(prompt).call().content();
    }
}
```

### ✅ Vector Embeddings - Semantic Search

```java
@Configuration
public class VectorConfig {
    
    @Bean
    public EmbeddingModel embeddingModel(OpenAiEmbeddingModel model) {
        return model;
    }
}

@Service
public class DocumentSearchService {
    
    @Autowired
    private EmbeddingModel embeddingModel;
    
    public void indexDocuments(List<String> documents) {
        for (String doc : documents) {
            // Generate Embedding (Vector)
            List<Double> embedding = embeddingModel.embed(doc);
            
            // Store in Vector Database
            vectorDatabase.store(doc, embedding);
        }
    }
    
    public String semanticSearch(String query) {
        // Find Similar Documents
        List<Double> queryEmbedding = embeddingModel.embed(query);
        List<String> similarDocs = vectorDatabase.searchNearest(queryEmbedding, 5);
        
        // Use Similar Docs as Context for LLM
        String context = String.join("\n", similarDocs);
        
        return chatClient.prompt()
            .system("Answer Based on These Documents:\n" + context)
            .user(query)
            .call()
            .content();
    }
}
```

### ✅ Real-World Example - Customer Support Bot

```java
@RestController
@RequiredArgsConstructor
public class SupportBotController {
    
    private final ChatClient chatClient;
    private final DocumentSearchService documentSearch;
    private final TicketRepository ticketRepository;
    
    @PostMapping("/support/ask")
    public ResponseEntity<String> askQuestion(@RequestBody SupportRequest request) {
        // 1. Search Knowledge Base
        String relevantDocs = documentSearch.semanticSearch(request.question());
        
        // 2. Generate Response with Context
        String response = chatClient.prompt()
            .system("""
                You are a helpful support agent. Use the provided documentation 
                to answer customer questions. Be concise and friendly.
                
                Available Documentation:
                """ + relevantDocs
            )
            .user(request.question())
            .call()
            .content();
        
        // 3. Create Ticket if Escalation Needed
        if (response.contains("escalate")) {
            Ticket ticket = new Ticket(
                request.customerId(),
                request.question(),
                response,
                Ticket.Status.ESCALATED
            );
            ticketRepository.save(ticket);
        }
        
        return ResponseEntity.ok(response);
    }
    
    @GetMapping(value = "/support/chat", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<String> streamResponse(@RequestParam String question) {
        String context = documentSearch.semanticSearch(question);
        
        return chatClient.prompt()
            .system("You are a helpful support agent.\nContext:\n" + context)
            .user(question)
            .stream()
            .content();
    }
}
```

### ✅ Error Handling - Graceful Failures

```java
@Service
public class ResilientAIService {
    
    @Autowired
    private ChatClient chatClient;
    
    public String safeChat(String prompt) {
        try {
            return chatClient.prompt()
                .user(prompt)
                .call()
                .content();
        } catch (RateLimitException e) {
            logger.warn("Rate Limited - Retrying");
            return "I'm temporarily busy. Please try again soon.";
        } catch (ApiException e) {
            logger.error("API Error", e);
            return "I'm having technical difficulties. Please contact support.";
        } catch (Exception e) {
            logger.error("Unexpected Error", e);
            throw new RuntimeException("AI Service Unavailable");
        }
    }
}
```

## 💡 Why This Matters

Spring AI abstracts away boilerplate for LLM integration - no manual HTTP requests or JSON parsing. ChatClient provides familiar fluent API - Spring developers immediately productive. Supports multiple providers (OpenAI, Anthropic, Google) - flexible to switch or compare. Structured output parsing eliminates manual JSON handling - type-safe responses. Tool calling enables LLMs to invoke Java methods - extends LLM capabilities. Vector embeddings for semantic search - not keyword matching. Streaming responses enable real-time UIs - Server-Sent Events support built-in. Spring Boot integration - auto-configuration, dependency injection, configuration properties.

## 🎯 Key Takeaway

Add Spring AI to build intelligent Java applications without Python. Use ChatClient for simple interactions, tool calling for complex workflows. Structured outputs keep responses type-safe and predictable. Leverage semantic search with embeddings for better context. Multi-model support lets you compare or optimize for different use cases. Spring AI makes AI accessible to enterprise Java developers.

---

**Tags:** `#Java` `#JavaWisdom` `#SpringAI` `#LLM` `#GenerativeAI` `#OpenAI` `#Anthropic` `#SpringBoot` `#AI` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity` `#SoftwareEngineering`
