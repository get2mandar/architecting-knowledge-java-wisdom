# Post #48: AI Function Calling - Integrating Tools with LLMs

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 19, 2026  
**Topic:** Function Calling, Tool Use, OpenAI API, LLM Integration

---

## The Problem

LLMs generate text. They don't execute code. Function calling bridges that gap—letting LLMs request external tools intelligently.

## Code Example

### ❌ Without Function Calling - Manual Coordination

```java
String userPrompt = "What's the weather in Paris and the temperature in Celsius?";

String response = chatModel.generate(new UserMessage(userPrompt));
// Response: "I don't have real-time access to weather data..."
// LLM Can't Fetch Real Data!
```

### ✅ With Function Calling - LLM Orchestrates Tools

```java
public record WeatherResult(
    String location,
    Integer temperature,
    String unit
) {}

@Service
public class WeatherAssistant {
    
    @Autowired
    private OpenAiChatModel chatModel;
    
    public String getWeather(String city) {
        // 1. Define Available Tools
        List<Tool> tools = List.of(
            Tool.builder()
                .name("get_current_weather")
                .description("Get current temperature for a city")
                .inputSchema(Map.of(
                    "type", "object",
                    "properties", Map.of(
                        "location", Map.of("type", "string"),
                        "unit", Map.of("type", "string", "enum", List.of("celsius", "fahrenheit"))
                    ),
                    "required", List.of("location")
                ))
                .build()
        );
        
        // 2. Send Prompt with Tools to LLM
        ChatResponse response = chatModel.call(
            ChatRequest.builder()
                .messages(List.of(
                    new UserMessage("What's the weather in " + city + "?")
                ))
                .tools(tools)  // Register Tools!
                .toolChoice(ToolChoice.AUTO)  // Let LLM Decide
                .build()
        );
        
        // 3. LLM Returns Tool Call
        ToolCall toolCall = response.getToolCall();
        
        // 4. Execute Tool
        String arguments = toolCall.getArguments();  // {"location": "Paris", "unit": "celsius"}
        WeatherResult result = callWeatherAPI(arguments);
        
        // 5. Feed Result Back to LLM
        ChatResponse finalResponse = chatModel.call(
            ChatRequest.builder()
                .messages(List.of(
                    new UserMessage("What's the weather in " + city + "?"),
                    new AssistantMessage(toolCall),
                    new ToolResultMessage(toJson(result), toolCall.getId())
                ))
                .build()
        );
        
        return finalResponse.getContent();
    }
}
```

### ✅ Function Calling Flow - Step by Step

```java
/*
Step 1: USER SENDS PROMPT
  "What's the weather in Paris?"

Step 2: CLIENT SENDS TO LLM WITH TOOL DEFINITIONS
  {
    "messages": [...],
    "tools": [{
      "name": "get_current_weather",
      "parameters": {
        "properties": {
          "location": {"type": "string"},
          "unit": {"enum": ["celsius", "fahrenheit"]}
        }
      }
    }]
  }

Step 3: LLM DECIDES IF TOOL NEEDED
  "I need weather data for Paris."
  -> Chooses get_current_weather

Step 4: LLM GENERATES TOOL CALL
  {
    "type": "function",
    "function": {
      "name": "get_current_weather",
      "arguments": {"location": "Paris", "unit": "celsius"}
    }
  }

Step 5: CLIENT EXECUTES TOOL
  result = weatherAPI.getWeather("Paris", "celsius")
  -> {"temperature": 22, "condition": "sunny"}

Step 6: CLIENT SENDS RESULT BACK TO LLM
  {
    "tool_result": {
      "temperature": 22,
      "condition": "sunny"
    }
  }

Step 7: LLM GENERATES FINAL RESPONSE
  "The weather in Paris is 22°C and sunny."
*/
```

### ✅ LangChain4j Integration - Type-Safe Tools

```java
// Define Tool Interface
public interface WeatherTools {
    
    @Tool("Get current weather for a location")
    String getCurrentWeather(
        @Parameter("City name") String location,
        @Parameter("Temperature unit: celsius or fahrenheit") String unit
    );
    
    @Tool("Get weather forecast for next N days")
    String getForecast(
        @Parameter("City name") String location,
        @Parameter("Number of days") int days
    );
}

// Implement Tools
@Component
public class WeatherToolsImpl implements WeatherTools {
    
    @Override
    public String getCurrentWeather(String location, String unit) {
        // Call Real Weather API
        return "Paris: 22°C, Sunny";
    }
    
    @Override
    public String getForecast(String location, int days) {
        return "7-day forecast: Sunny, Partly Cloudy, Rainy";
    }
}

// Register with AI Agent
@AiService
public interface WeatherAgent {
    
    @SystemMessage("You are a weather assistant. Use available tools to answer questions.")
    String answer(String question);
}

@Configuration
public class AgentConfig {
    
    @Bean
    public WeatherAgent weatherAgent(OpenAiChatModel model) {
        return AiServices.builder(WeatherAgent.class)
            .chatLanguageModel(model)
            .tools(new WeatherToolsImpl())  // Register!
            .build();
    }
}

// Usage
String answer = weatherAgent.answer("What's the weather in Paris?");
// LLM Automatically Calls getCurrentWeather("Paris", "celsius")
```

### ✅ Parallel Function Calls - Multiple Tools at Once

```java
@Service
public class ComprehensiveInfoService {
    
    public record CompanyInfo(
        String name,
        BigDecimal stockPrice,
        String newsHeadline,
        List<String> recentTweets
    ) {}
    
    public CompanyInfo getCompanyInfo(String companyName) {
        String prompt = "Get comprehensive information for " + companyName;
        
        // LLM Can Call Multiple Tools in Parallel!
        ToolCall[] toolCalls = chatModel.getToolCalls(prompt);
        
        // Execute All Simultaneously
        String price = getStockPrice(companyName);
        String news = getLatestNews(companyName);
        List<String> tweets = getTweets(companyName);
        
        // LLM Synthesizes Results
        return new CompanyInfo(companyName, price, news, tweets);
    }
}
```

### ✅ Real-World Example - Travel Planning Agent

```java
public interface TravelTools {
    
    @Tool("Find flights between two cities")
    String findFlights(
        @Parameter("Departure city") String from,
        @Parameter("Arrival city") String to,
        @Parameter("Travel date YYYY-MM-DD") String date
    );
    
    @Tool("Find hotels in a city")
    String findHotels(
        @Parameter("City") String city,
        @Parameter("Check-in date") String checkIn,
        @Parameter("Check-out date") String checkOut
    );
    
    @Tool("Get weather forecast")
    String getWeather(
        @Parameter("City") String city,
        @Parameter("Date YYYY-MM-DD") String date
    );
    
    @Tool("Check visa requirements")
    String checkVisa(
        @Parameter("Your country") String from,
        @Parameter("Destination country") String to
    );
}

@AiService
public interface TravelPlanner {
    
    @SystemMessage("""
        You are an expert travel planning assistant.
        Use available tools to create comprehensive travel itineraries.
        Consider flights, hotels, weather, and visa requirements.
        """)
    String planTrip(String request);
}

// Usage
TravelPlanner planner = AiServices.builder(TravelPlanner.class)
    .chatLanguageModel(openAiModel)
    .tools(new TravelToolsImpl())
    .build();

String itinerary = planner.planTrip(
    "Plan a 5-day trip from New York to Paris in July 2026"
);
// LLM Automatically:
// 1. Finds Flights (NYC to Paris, July dates)
// 2. Books Hotels (5 nights in Paris)
// 3. Checks Weather (July in Paris)
// 4. Verifies Visa (US citizens to France)
// 5. Synthesizes Complete Itinerary
```

### ✅ Error Handling in Function Calls

```java
@Service
public class RobustToolService {
    
    public String executeToolSafely(String userQuery) {
        try {
            ToolCall toolCall = chatModel.selectTool(userQuery);
            
            // Execute with Timeout
            Future<String> result = executor.submit(() -> {
                return executeTool(toolCall);
            });
            
            String toolResult = result.get(5, TimeUnit.SECONDS);  // 5 Second Timeout
            
            // Feed Back to LLM
            return chatModel.generateResponse(userQuery, toolResult);
            
        } catch (TimeoutException e) {
            logger.error("Tool execution timeout");
            return "Tool took too long. Please try again.";
        } catch (Exception e) {
            logger.error("Tool execution failed", e);
            return "I encountered an error while executing the tool.";
        }
    }
}
```

## 💡 Why This Matters

Function calling enables LLMs to request execution of external tools without writing code. LLM decides when and which tool to call - intelligent orchestration. Structured tool definitions (JSON schema) ensure LLM generates valid arguments. Parallel execution speeds up complex queries requiring multiple tool calls. Tools extend LLM capabilities: database queries, API calls, calculations, real-time data. LangChain4j simplifies tool integration with @Tool annotations. Error handling crucial - tools can fail, must degrade gracefully.

## 🎯 Key Takeaway

Function calling transforms LLMs from text generators to intelligent agents. Define tools clearly with descriptions and schemas. Always handle tool failures gracefully. Parallel execution of independent tools reduces latency. Let LLMs orchestrate complexity - they're great at deciding what to do.

---

**Tags:** `#Java` `#JavaWisdom` `#FunctionCalling` `#ToolUse` `#LLM` `#AIAgents` `#OpenAI` `#SpringAI` `#GenerativeAI` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity` `#TechCommunity`
