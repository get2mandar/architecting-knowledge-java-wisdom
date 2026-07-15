# Post #46: Prompt Engineering - Structured Outputs in Java

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** July 12, 2026  
**Topic:** Prompt Engineering, Structured Outputs, JSON Schema, Type Safety

---

## The Problem

LLMs hallucinate. They ignore schemas. They rename fields. Prompt engineering + Structured Outputs = guaranteed compliance.

## Code Example

### ❌ Unstructured Prompts - Unpredictable Output

```java
String prompt = "Extract the customer info from: John Doe, 30 years old, john@example.com";

String response = chatModel.generate(new UserMessage(prompt));
// Response: "Customer: John Doe, Age: 30, Email: john@example.com"
// OR
// Response: "The customer is John Doe who is 30. Contact: john@example.com"
// OR (Hallucination!)
// Response: "Customer John Doe (age 30) has email john@example.com and lives in New York"

// Problem: Can't Reliably Parse String Response!
```

### ✅ Structured Outputs - JSON Schema Enforcement

```java
// Define Output Structure
public record Customer(
    String name,
    Integer age,
    String email
) {}

// Prompt with Schema Enforcement
String prompt = "Extract customer info: John Doe, 30 years old, john@example.com";

Customer customer = chatModel.generate(
    List.of(new UserMessage(prompt)),
    Customer.class  // Enforce Schema!
);

// Response: Guaranteed JSON Matching Schema
// {
//   "name": "John Doe",
//   "age": 30,
//   "email": "john@example.com"
// }

// Parsing Safe - Direct to Object!
```

### ✅ JSON Schema Definition - Type-Safe Contracts

```java
// Define Complex Nested Structure
public record ReviewAnalysis(
    String sentiment,      // positive, neutral, negative
    Integer rating,        // 1-5
    List<String> aspects,  // what was reviewed
    String summary
) {}

public record BatchReviewAnalysis(
    List<ReviewAnalysis> reviews,
    Integer averageRating,
    String overallTrend
) {}

// Schema Automatically Generated from Java Record!
// No Manual JSON Schema Writing Needed
```

### ✅ System Message + Structured Output

```java
@Service
public class DataExtractionService {
    
    public record ContractTerms(
        LocalDate startDate,
        LocalDate endDate,
        BigDecimal totalValue,
        String jurisdiction,
        List<String> keyTerms
    ) {}
    
    public ContractTerms analyzeContract(String contractText) {
        String systemPrompt = """
            You are a legal document analyzer.
            Extract key contract terms accurately.
            Be precise with dates (YYYY-MM-DD format).
            List only explicit terms mentioned in the contract.
            """;
        
        return chatModel.generate(
            List.of(
                new SystemMessage(systemPrompt),
                new UserMessage("Analyze this contract:\n" + contractText)
            ),
            ContractTerms.class  // Schema Enforced!
        );
    }
}
```

### ✅ Validation Schema - Add Constraints

```java
public record ProductReview(
    @JsonProperty(required = true)
    String productId,
    
    @JsonProperty(required = true)
    @Positive  // Must be > 0
    Integer rating,
    
    @Size(min = 10, max = 500)
    String reviewText,
    
    @Pattern(regexp = "^[A-Z][a-z]+$")
    String reviewerName
) {}

// OpenAI Enforces These Constraints!
// Invalid Values = Error (Not Hallucination)
```

### ✅ Enum Outputs - No String Confusion

```java
public enum SentimentType {
    POSITIVE, NEUTRAL, NEGATIVE, MIXED
}

public record SentimentAnalysis(
    SentimentType sentiment,  // Enum - Not String!
    Integer confidence,        // 0-100
    List<String> keyPhrases
) {}

// LLM Can Only Select from Valid Enum Values
// No Misspellings or Hallucinated Values!
```

### ✅ Real-World Example - Invoice Extraction

```java
@Service
public class InvoiceExtractionService {
    
    public record InvoiceItem(
        String description,
        Integer quantity,
        BigDecimal unitPrice,
        BigDecimal totalPrice
    ) {}
    
    public record Invoice(
        String invoiceNumber,
        LocalDate invoiceDate,
        LocalDate dueDate,
        String vendor,
        List<InvoiceItem> lineItems,
        BigDecimal subtotal,
        BigDecimal tax,
        BigDecimal total
    ) {}
    
    public Invoice extractFromImage(byte[] invoiceImage) {
        String systemPrompt = """
            You are an expert invoice processor.
            Extract all invoice details accurately.
            Dates must be in ISO 8601 format (YYYY-MM-DD).
            All monetary values as decimals (e.g., 10.99).
            Ensure line item totals match quantity × unit price.
            """;
        
        return chatModel.generate(
            List.of(
                new SystemMessage(systemPrompt),
                new UserMessage(
                    new ImageContent("image/jpeg", invoiceImage),
                    new TextContent("Extract invoice details from this image")
                )
            ),
            Invoice.class
        );
    }
}
```

### ✅ Multi-Step Reasoning with Structured Output

```java
public record MathStep(
    String operation,
    String calculation,
    Double result
) {}

public record MathSolution(
    List<MathStep> steps,
    Double finalAnswer,
    String explanation
) {}

@Service
public class MathTutorService {
    
    public MathSolution solveMathProblem(String problem) {
        String systemPrompt = """
            You are a math tutor.
            Break down complex problems into clear steps.
            Show each calculation explicitly.
            Provide step-by-step reasoning.
            """;
        
        return chatModel.generate(
            List.of(
                new SystemMessage(systemPrompt),
                new UserMessage("Solve: " + problem)
            ),
            MathSolution.class
        );
    }
}

// Usage
MathSolution solution = tutorService.solveMathProblem(
    "What is (3x + 5) when x = 4?"
);

solution.steps().forEach(step ->
    System.out.println(step.operation() + ": " + step.result())
);
```

### ✅ Error Handling with Structured Outputs

```java
@Service
public class RobustExtractionService {
    
    public record ExtractionResult<T>(
        T data,
        Boolean success,
        String error
    ) {}
    
    public ExtractionResult<Customer> safeExtract(String input) {
        try {
            Customer customer = chatModel.generate(
                List.of(new UserMessage(input)),
                Customer.class
            );
            
            return new ExtractionResult<>(customer, true, null);
        } catch (IllegalArgumentException e) {
            // Schema Validation Failed
            return new ExtractionResult<>(null, false, e.getMessage());
        }
    }
}
```

### ✅ Conditional Schemas - Flexible Structures

```java
// Union Type - One of Multiple Options
public sealed interface QueryResult permits 
    SuccessResult, ErrorResult {}

public record SuccessResult(
    List<String> results,
    Integer count
) implements QueryResult {}

public record ErrorResult(
    String errorType,
    String message
) implements QueryResult {}

// LLM Chooses Correct Type Based on Input!
```

## 💡 Why This Matters

Structured Outputs guarantee schema adherence - no hallucinations, no renamed fields. OpenAI's Structured Outputs uses constrained sampling - enforces valid JSON at token generation level. JSON Schema separates content from format - prompt decides "what", schema decides "shape". Enum types prevent invalid values - LLM can only select predefined options. Nested structures allow complex hierarchical outputs - supports real-world data. Spring Boot integration makes it seamless - just define Records, pass to ChatModel. Type safety throughout - compile-time safety, runtime guarantees.

## 🎯 Key Takeaway

Always use Structured Outputs with schemas - never rely on unstructured text parsing. Define Output as Java Records - automatic schema generation. Add validation constraints (required fields, ranges, patterns). Use Enums for categorical values - prevent string variations. Test schema compliance in development - catch issues before production.

---

**Tags:** `#Java` `#JavaWisdom` `#PromptEngineering` `#StructuredOutputs` `#JSONSchema` `#SpringAI` `#GenerativeAI` `#LLM` `#TypeSafety` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#SpringFramework` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#DevCommunity`
