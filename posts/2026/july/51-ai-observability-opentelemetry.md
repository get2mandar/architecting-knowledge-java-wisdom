# Post #51: AI Observability - Tracing LLM Calls with OpenTelemetry

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 29, 2026  
**Topic:** OpenTelemetry, LLM Observability, Distributed Tracing, Spring Boot

---

## The Problem

LLM calls are black boxes. You can't see token usage, latency, or failures. OpenTelemetry brings full visibility to AI systems.

## Code Example

### ❌ Without Observability - Blind Spot

```java
@Service
public class ChatService {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    public String chat(String prompt) {
        // LLM Call Happens Here - But Nobody Knows What Happened!
        String response = chatModel.generate(prompt);
        
        // Questions Unanswered:
        // - How many tokens were used?
        // - How long did it take?
        // - What model was called?
        // - Did it fail or degrade?
        
        return response;
    }
}

// Logs:
// "Chat response received" - Too Late, No Details!
// No Tracing, No Token Tracking, No Cost Calculation
```

### ✅ With OpenTelemetry - Full Visibility

```java
// pom.xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
</dependency>

// application.properties
spring.application.name=ai-service
management.endpoints.web.exposure.include=health,metrics
management.tracing.sampling.probability=1.0  // 100% Sampling
management.otlp.tracing.endpoint=http://localhost:4317  // Jaeger/Tempo

@Service
public class ObservableAIService {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    @Autowired
    private Tracer tracer;
    
    public String chat(String prompt) {
        // Spring Boot Auto-Creates Trace ID
        // Single Trace Spans Multiple Services
        
        Span span = tracer.spanBuilder("llm.chat")
            .setAttribute("model", "gpt-4o")
            .setAttribute("prompt.length", prompt.length())
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // LLM Call
            String response = chatModel.generate(prompt);
            
            // Span Automatically Records:
            // - Trace ID: abc123def456
            // - Model: gpt-4o
            // - Duration: 2.3 seconds
            // - Status: Success
            
            return response;
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### ✅ OpenTelemetry Architecture - Three Pillars

```java
/*
OBSERVABILITY PILLARS:

Traces (What Happened - Causality):
  - Order Flow: HTTP Request → Service A → LLM → Database → Response
  - Trace ID: Connects All Spans
  - Shows Where Time Was Spent
  
Metrics (How Much - Aggregation):
  - Token Count: 2500
  - Latency: 2.3 seconds
  - Cost: $0.045
  - Error Rate: 0.5%
  
Logs (Why It Happened - Details):
  - Model: gpt-4o
  - Prompt Tokens: 1200
  - Completion Tokens: 1300
  - Cache Hit: false
  - Timestamp: 2026-07-29T10:45:00Z
*/
```

### ✅ Spring Boot + OpenTelemetry Integration

```java
@Configuration
public class ObservabilityConfig {
    
    @Bean
    public Tracer tracer(MeterRegistry meterRegistry) {
        // Spring Boot Auto-Configures OpenTelemetry
        // No Manual Setup Required!
        return GlobalOpenTelemetry.getTracer("ai-service");
    }
}

@Service
public class InstrumentedAIService {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Autowired
    private Tracer tracer;
    
    // Automatic Span Creation via @Observed (Micrometer)
    @Observed(name = "llm.chat")
    public String chat(String prompt) {
        // Span Created Automatically!
        // - Start Time: Auto
        // - Attributes: Auto from Observation
        // - End Time: Auto
        
        return chatModel.generate(prompt);
    }
    
    // Manual Span for Fine-Grained Control
    public String advancedChat(String prompt) {
        Span span = tracer.spanBuilder("llm.advanced")
            .setAttribute("model", "gpt-4o")
            .setAttribute("temperature", 0.7)
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // Nested Spans
            Span embeddingSpan = tracer.spanBuilder("vector.embed")
                .startSpan();
            
            try {
                // Generate Embedding for Context
                String embedding = generateEmbedding(prompt);
                embeddingSpan.addEvent("embedding_complete");
            } finally {
                embeddingSpan.end();
            }
            
            // Main LLM Call
            return chatModel.generate(prompt);
        } finally {
            span.end();
        }
    }
    
    // Metrics for Cost Tracking
    private void recordTokenMetrics(int promptTokens, int completionTokens) {
        meterRegistry.counter("llm.tokens.prompt",
            Tags.of("model", "gpt-4o")).increment(promptTokens);
        meterRegistry.counter("llm.tokens.completion",
            Tags.of("model", "gpt-4o")).increment(completionTokens);
        
        // Cost Calculation (Example)
        double promptCost = promptTokens * 0.0005 / 1000;
        double completionCost = completionTokens * 0.0015 / 1000;
        
        meterRegistry.gauge("llm.cost.usd",
            Tags.of("model", "gpt-4o"),
            promptCost + completionCost);
    }
}
```

### ✅ Real-World Example - Full AI Service Tracing

```java
@RestController
public class AIController {
    
    @Autowired
    private AIOrchestrationService orchestration;
    
    @Autowired
    private Tracer tracer;
    
    @PostMapping("/chat")
    public ResponseEntity<String> chat(@RequestBody ChatRequest request) {
        // HTTP Request Creates Root Span Automatically
        // Trace ID: abc123xyz789
        
        Span httpSpan = tracer.spanBuilder("http.request")
            .setAttribute("user.id", request.userId())
            .setAttribute("intent", request.intent())
            .startSpan();
        
        try (Scope scope = httpSpan.makeCurrent()) {
            // Call Orchestration Service
            String response = orchestration.processRequest(request);
            
            httpSpan.addEvent("response_generated");
            return ResponseEntity.ok(response);
        } finally {
            httpSpan.end();
        }
    }
}

@Service
public class AIOrchestrationService {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    @Autowired
    private DocumentSearchService searchService;
    
    @Autowired
    private Tracer tracer;
    
    @Observed(name = "rag.process")
    public String processRequest(ChatRequest request) {
        // Span 1: Retrieve Documents
        Span searchSpan = tracer.spanBuilder("rag.retrieve")
            .startSpan();
        
        try {
            List<Document> docs = searchService.search(request.query());
            searchSpan.addEvent("retrieved_documents",
                Attributes.of(AttributeKey.longKey("count"), (long) docs.size()));
        } finally {
            searchSpan.end();
        }
        
        // Span 2: LLM Generation
        Span llmSpan = tracer.spanBuilder("llm.generate")
            .setAttribute("model", "gpt-4o")
            .startSpan();
        
        try {
            String response = chatModel.generate(request.query());
            llmSpan.addEvent("generation_complete");
            return response;
        } finally {
            llmSpan.end();
        }
    }
}

// Trace Visualization (in Jaeger/Tempo):
// Root Span: http.request (2500ms total)
//   ├── rag.retrieve (400ms) - Vector Search
//   ├── llm.generate (2000ms) - OpenAI API
//   └── json.parse (50ms) - Response Parsing
//
// See Exactly Where Time Was Spent!
```

### ✅ GenAI Semantic Conventions - LLM-Specific Attributes

```java
@Service
public class SemanticConventionExample {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    @Autowired
    private Tracer tracer;
    
    public String generateWithSemanticConventions(String prompt) {
        Span span = tracer.spanBuilder("gen_ai.chat.completions")
            // Model Information
            .setAttribute("gen_ai.system", "OpenAI")
            .setAttribute("gen_ai.request.model", "gpt-4o")
            .setAttribute("gen_ai.request.temperature", 0.7)
            
            // Token Information
            .setAttribute("gen_ai.usage.prompt_tokens", 1200)
            .setAttribute("gen_ai.usage.completion_tokens", 800)
            
            // Cost Information
            .setAttribute("gen_ai.usage.cost_usd", 0.045)
            
            // Operation Status
            .setAttribute("gen_ai.response.finish_reason", "stop")
            
            .startSpan();
        
        try {
            String response = chatModel.generate(prompt);
            
            span.addEvent("llm_response_received");
            span.setStatus(StatusCode.OK);
            
            return response;
        } catch (RateLimitException e) {
            span.setStatus(StatusCode.ERROR, "rate_limit_exceeded");
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
}

// OpenTelemetry Semantic Conventions for GenAI:
// https://opentelemetry.io/docs/specs/semconv/gen-ai/
// Standardized Attributes Across All LLM Providers
```

### ✅ Export to Observability Backends

```java
// application.properties
spring.application.name=ai-chatbot

# OTLP Exporter
management.otlp.tracing.endpoint=http://localhost:4317
management.otlp.tracing.protocol=grpc  // or http

# Jaeger (Drop-in)
spring.observability.exporter.jaeger.enabled=true
spring.observability.exporter.jaeger.endpoint=http://localhost:14250

# Alternative: Tempo (for Prometheus-style Scraping)
spring.observability.tracing.tempo.enabled=true

# Metrics Export (Prometheus)
management.endpoints.web.exposure.include=prometheus
management.metrics.export.prometheus.enabled=true
```

### ✅ Cost Tracking - Real Business Metrics

```java
@Service
public class LLMCostTracker {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @Autowired
    private Tracer tracer;
    
    public void trackLLMCost(String model, int promptTokens, int completionTokens) {
        // Pricing Model (Example)
        Map<String, Double> pricing = Map.of(
            "gpt-4o", 0.015,  // $ per 1M input tokens
            "gpt-4o-mini", 0.003
        );
        
        double promptCost = (promptTokens / 1_000_000.0) * pricing.get(model);
        double completionCost = (completionTokens / 1_000_000.0) * pricing.get(model) * 2;  // 2x Multiplier
        double totalCost = promptCost + completionCost;
        
        // Record Metrics
        meterRegistry.counter("llm.cost.total",
            Tags.of("model", model),
            totalCost);
        
        meterRegistry.counter("llm.tokens.input",
            Tags.of("model", model),
            promptTokens);
        
        meterRegistry.counter("llm.tokens.output",
            Tags.of("model", model),
            completionTokens);
        
        // Log to Trace
        Span span = GlobalOpenTelemetry.getTracer("ai-service")
            .spanBuilder("llm.cost_tracking")
            .setAttribute("model", model)
            .setAttribute("prompt_tokens", promptTokens)
            .setAttribute("completion_tokens", completionTokens)
            .setAttribute("cost_usd", totalCost)
            .startSpan();
        
        span.end();
    }
}

// Grafana Dashboard Shows:
// - Total Daily Cost: $1,234.56
// - Cost per Model: gpt-4o: $800, gpt-4o-mini: $200
// - Token Efficiency: 1M tokens for $0.018
// - Cost Trend: ↑ 12% Week-over-Week
```

## 💡 Why This Matters

OpenTelemetry is vendor-neutral standard - not locked into Jaeger or Datadog. Spring Boot integration makes tracing automatic - just add starter. GenAI semantic conventions standardize LLM attributes across providers. Distributed tracing connects HTTP requests through LLM calls to databases. Token tracking enables cost management - critical for high-volume AI systems. Span events allow rich context - not just timing, but what happened. Error recording captures failures with full context.

## 🎯 Key Takeaway

Add spring-boot-starter-opentelemetry for production AI services. Use @Observed annotation for automatic span creation. Implement semantic conventions for LLM calls - standardized attributes. Track tokens and costs as first-class metrics. Export to Jaeger/Tempo for distributed trace visualization. OpenTelemetry + Spring Boot = Complete AI system visibility.

---

**Tags:** `#Java` `#JavaWisdom` `#OpenTelemetry` `#Observability` `#DistributedTracing` `#LLM` `#SpringBoot` `#Monitoring` `#Metrics` `#Logging` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity` `#SoftwareEngineering`
