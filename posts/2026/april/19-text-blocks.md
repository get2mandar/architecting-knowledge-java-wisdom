# Post #19: Text Blocks - Readable Strings Finally

**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** April 8, 2026  
**Topic:** Text Blocks, Modern Java, String Literals

---

## The Problem

Stop escaping quotes. Stop concatenating. Just write.

## Code Example

### ❌ Traditional Strings - Escape Hell

```java
String json = "{\n" +
              "  \"name\": \"John\",\n" +
              "  \"age\": 30,\n" +
              "  \"city\": \"New York\"\n" +
              "}";

String html = "<html>\n" +
              "  <body>\n" +
              "    <h1>Welcome</h1>\n" +
              "  </body>\n" +
              "</html>";

String sql = "SELECT u.id, u.name, o.total\n" +
             "FROM users u\n" +
             "JOIN orders o ON u.id = o.user_id\n" +
             "WHERE o.status = 'ACTIVE'";
```

### ✅ Text Blocks - Natural and Clean

```java
String json = """
    {
      "name": "John",
      "age": 30,
      "city": "New York"
    }
    """;

String html = """
    <html>
      <body>
        <h1>Welcome</h1>
      </body>
    </html>
    """;

String sql = """
    SELECT u.id, u.name, o.total
    FROM users u
    JOIN orders o ON u.id = o.user_id
    WHERE o.status = 'ACTIVE'
    """;
```

### ❌ Formatting Mistakes - Confusing Indentation

```java
String query = """
        SELECT * FROM users
    WHERE active = true
      ORDER BY name
    """;
// Inconsistent White Space - Compiler Warning!
```

### ✅ Consistent Indentation - Clean Output

```java
String query = """
    SELECT * FROM users
    WHERE active = true
    ORDER BY name
    """;
```

### ✅ Variable Interpolation - Use Formatted

```java
String userId = "12345";
String status = "ACTIVE";

String query = """
    SELECT * FROM orders
    WHERE user_id = '%s'
    AND status = '%s'
    """.formatted(userId, status);
```

### ✅ Line Continuation - Suppress Newline

```java
String longLine = """
    This is a very long sentence that \
    continues on the next line without \
    creating a newline character.
    """;
// Output: "This is a very long sentence that continues..."
```

### ✅ Preserve Quotes - No Escaping Needed

```java
String message = """
    He said, "Hello World!"
    Then she replied, "How are you?"
    """;
// Double Quotes Work Naturally!
```

### ✅ Real-World Example - REST API Response

```java
@GetMapping("/user/{id}")
public String getUserProfile(@PathVariable String id) {
    return """
        {
          "userId": "%s",
          "profile": {
            "displayName": "John Doe",
            "email": "[email protected]",
            "preferences": {
              "theme": "dark",
              "notifications": true
            }
          }
        }
        """.formatted(id);
}
```

## 💡 Why This Matters

Text blocks (Java 15+) eliminate escape sequences for quotes and newlines. Triple-quotes `"""` delimit multi-line strings. Indentation is automatically stripped based on the closing delimiter position. No more `\n` or `+` concatenation. Line terminators are normalized to `\n` regardless of platform. Use `\` at end of line to suppress newline. The `formatted()` method works perfectly with text blocks for variable substitution. Compiler warns about inconsistent indentation with `-Xlint:text-blocks`.

## 🎯 Key Takeaway

Use text blocks for JSON, SQL, HTML, XML, or any multi-line content. Keep indentation consistent (spaces OR tabs, not mixed). Position closing `"""` to control base indentation. Your code becomes readable, your strings become maintainable.

---

**Tags:** `#Java` `#JavaWisdom` `#TextBlocks` `#ModernJava` `#Java15` `#CleanCode`
