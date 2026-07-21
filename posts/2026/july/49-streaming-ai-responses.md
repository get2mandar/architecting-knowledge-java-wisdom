# Post #49: Streaming AI Responses - Server-Sent Events in Spring

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 22, 2026  
**Topic:** Server-Sent Events, Spring WebFlux, Real-Time Streaming, Reactive

---

## The Problem

Users hate waiting for complete responses. Stream chunks as they arrive. SSE enables real-time updates without WebSocket complexity.

## Code Example

### ❌ Without Streaming - Full Response Wait

```java
@PostMapping("/ask")
public ResponseEntity<String> ask(@RequestBody String question) {
    // Waits Until LLM Completes Entire Response (30+ seconds!)
    String fullResponse = chatModel.generate(question);
    
    return ResponseEntity.ok(fullResponse);
}

// User Experience: 30 Second Blank Screen ❌
```

### ✅ With Streaming - Real-Time Chunks

```java
@GetMapping(value = "/ask/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> askStream(@RequestParam String question) {
    return chatModel.stream(question)  // Returns Flux<String>
        .map(chunk -> {
            // Each chunk arrives as it's generated
            logger.info("Chunk: " + chunk);
            return chunk;
        });
}

// User Experience: Instant Response, Updating In Real-Time ✅
```

### ✅ Understanding SSE vs WebSocket

```java
/*
SSE (Server-Sent Events):
  - Protocol: HTTP (simple)
  - Direction: Server → Client Only (unidirectional)
  - Use: Notifications, Streaming Responses, Live Data
  - Setup: Set Content-Type to text/event-stream
  - Browser Support: Native EventSource API
  - Firewall: Passes Through (Standard HTTP)

WebSocket:
  - Protocol: WebSocket (complex)
  - Direction: Bidirectional
  - Use: Real-Time Chat, Games, Collaborative Tools
  - Setup: Handshake, Protocol Upgrade
  - Browser Support: Native WebSocket API
  - Firewall: May Be Blocked

LLM Streaming: SSE is Perfect!
  - Only Server Sends Data ✓
  - Simple HTTP Connection ✓
  - Works Through Firewalls ✓
  - No Complex Handshake ✓
*/
```

### ✅ Spring WebFlux Streaming - Flux<String>

```java
@RestController
public class AIStreamController {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    // Stream as Strings
    @GetMapping(
        value = "/stream/text",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<String> streamText(@RequestParam String question) {
        return chatModel.stream(question);
    }
    
    // Stream with Delay Simulation (Realistic)
    @GetMapping(
        value = "/stream/slow",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<String> streamWithDelay(@RequestParam String question) {
        return chatModel.stream(question)
            .delayElement(Duration.ofMillis(50));  // 50ms Between Chunks
    }
    
    // Stream Complex Objects
    @GetMapping(
        value = "/stream/json",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<ServerSentEvent<String>> streamJSON(@RequestParam String question) {
        return chatModel.stream(question)
            .map(chunk -> ServerSentEvent.builder()
                .data(chunk)
                .id(UUID.randomUUID().toString())
                .event("message")
                .build());
    }
}
```

### ✅ Client-Side: EventSource API

```html
<!-- HTML -->
<div id="response"></div>

<!-- JavaScript -->
<script>
function startStream() {
    const question = document.getElementById('question').value;
    
    // Create EventSource Connection
    const eventSource = new EventSource(`/stream/text?question=${question}`);
    
    // Handle Incoming Messages
    eventSource.onmessage = (event) => {
        // Event Data is Streamed Text
        document.getElementById('response').innerHTML += event.data;
    };
    
    // Handle Errors
    eventSource.onerror = (error) => {
        console.error('Streaming error:', error);
        eventSource.close();
    };
    
    // Handle Completion
    eventSource.addEventListener('done', () => {
        eventSource.close();
        console.log('Stream completed');
    });
}
</script>
```

### ✅ Spring with ServerSentEvent - Typed Streaming

```java
public record ChatMessage(
    String content,
    String timestamp,
    String id
) {}

@RestController
public class TypedStreamController {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    @GetMapping(
        value = "/stream/typed",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<ServerSentEvent<ChatMessage>> streamTyped(@RequestParam String question) {
        AtomicInteger counter = new AtomicInteger(0);
        
        return chatModel.stream(question)
            .map(chunk -> ServerSentEvent.<ChatMessage>builder()
                .id(String.valueOf(counter.incrementAndGet()))
                .event("message")
                .data(new ChatMessage(
                    chunk,
                    Instant.now().toString(),
                    UUID.randomUUID().toString()
                ))
                .build())
            .doOnComplete(() -> {
                // Emit Completion Event
                // Or Handle Cleanup
            });
    }
}

// Client-Side (React)
const response = useCallback(async (question) => {
    const eventSource = new EventSource(`/stream/typed?question=${question}`);
    
    eventSource.addEventListener('message', (event) => {
        const chatMsg = JSON.parse(event.data);
        setMessages(prev => [...prev, chatMsg]);
    });
}, []);
```

### ✅ Reactor Sink - Push-Based Streaming

```java
@Service
public class NotificationStreamService {
    
    private final Sinks.Multicast<String> notificationSink = 
        Sinks.multicast()
            .onBackpressureBuffer();
    
    public Flux<String> subscribeToNotifications() {
        return notificationSink.asFlux();
    }
    
    public void sendNotification(String message) {
        // Push Notification to All Subscribers
        notificationSink.tryEmitNext(message);
    }
}

@RestController
public class NotificationController {
    
    @Autowired
    private NotificationStreamService notificationService;
    
    @GetMapping(
        value = "/notifications",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<String> subscribe() {
        return notificationService.subscribeToNotifications();
    }
    
    @PostMapping("/notify")
    public ResponseEntity<?> notify(@RequestBody String message) {
        notificationService.sendNotification(message);
        return ResponseEntity.ok().build();
    }
}
```

### ✅ Real-World Example - Live Code Generation

```java
@RestController
public class CodeGeneratorController {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    @PostMapping(
        value = "/generate-code/stream",
        produces = MediaType.TEXT_EVENT_STREAM_VALUE
    )
    public Flux<ServerSentEvent<CodeChunk>> generateCodeStream(
        @RequestBody CodeRequest request
    ) {
        String prompt = buildCodePrompt(request);
        
        return chatModel.stream(prompt)
            .bufferUntilChanged()  // Group Similar Chunks
            .map(chunks -> String.join("", chunks))
            .map(code -> ServerSentEvent.<CodeChunk>builder()
                .id(UUID.randomUUID().toString())
                .event("code")
                .data(new CodeChunk(
                    code,
                    detectLanguage(code),
                    Instant.now()
                ))
                .build());
    }
    
    private String buildCodePrompt(CodeRequest request) {
        return String.format("""
            Write %s code to: %s
            Include comments and best practices.
            """, request.language(), request.description());
    }
}

// Frontend (React Component)
function CodeGenerator() {
    const [code, setCode] = useState('');
    const [loading, setLoading] = useState(false);
    
    const generateCode = async (request) => {
        setLoading(true);
        setCode('');
        
        const response = await fetch('/generate-code/stream', {
            method: 'POST',
            body: JSON.stringify(request)
        });
        
        const reader = response.body.getReader();
        const decoder = new TextDecoder();
        
        while (true) {
            const {done, value} = await reader.read();
            if (done) break;
            
            const text = decoder.decode(value);
            const lines = text.split('\n');
            
            lines.forEach(line => {
                if (line.startsWith('data:')) {
                    const json = JSON.parse(line.substring(5));
                    setCode(prev => prev + json.data.content);
                }
            });
        }
        
        setLoading(false);
    };
    
    return (
        <div>
            <button onClick={() => generateCode({language: 'Java'})}>
                Generate Java Code
            </button>
            <pre>{code}</pre>
        </div>
    );
}
```

### ✅ Error Handling and Backpressure

```java
@Service
public class ResilientStreamService {
    
    public Flux<ServerSentEvent<String>> streamWithErrorHandling(String question) {
        return chatModel.stream(question)
            // Handle Backpressure - Buffer Strategy
            .onBackpressureBuffer(100)  // Buffer 100 Items
            
            // Timeout Protection
            .timeout(Duration.ofSeconds(30))
            .onErrorResume(TimeoutException.class, e -> {
                logger.warn("Stream timeout");
                return Flux.just("Stream timed out");
            })
            
            // Retry on Transient Failures
            .retryWhen(Retry.backoff(2, Duration.ofSeconds(1)))
            
            // Transform to ServerSentEvent
            .map(chunk -> ServerSentEvent.builder()
                .data(chunk)
                .build())
            
            // Final Error Fallback
            .onErrorResume(e -> {
                logger.error("Stream failed", e);
                return Flux.just(ServerSentEvent.builder()
                    .event("error")
                    .data("Streaming failed: " + e.getMessage())
                    .build());
            });
    }
}
```

## 💡 Why This Matters

SSE enables streaming with simple HTTP - no WebSocket protocol complexity. Spring WebFlux returns Flux<T> which SSE automatically formats as text/event-stream. Server-Sent Events work through firewalls - standard HTTP connection. Streaming improves UX - users see content instantly, not after 30 second wait. Backpressure handling prevents memory issues - buffer, drop, or slow down. Reactor Sinks enable push-based streaming - notifications, live updates. Error handling crucial - timeout, retry, fallback to graceful degradation.

## 🎯 Key Takeaway

Use SSE for server-to-client streaming - simpler than WebSocket, perfect for LLM responses. Return Flux<String> with TEXT_EVENT_STREAM_VALUE media type. Handle backpressure with buffer strategies. Implement timeout and error recovery. Stream improves perceived performance dramatically.

---

**Tags:** `#Java` `#JavaWisdom` `#ServerSentEvents` `#SSE` `#SpringWebFlux` `#Streaming` `#ReactiveStreams` `#RealTime` `#SpringBoot` `#Flux` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity` `#SoftwareEngineering`
