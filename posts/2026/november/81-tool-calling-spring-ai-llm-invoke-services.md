# Post #81: Tool Calling in Spring AI - Letting LLMs Invoke Your Services
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** November 11, 2026  
**Topic:** Spring AI, Tool Calling, @Tool, ChatClient, Service Boundaries, Input Validation  
 
---
 
## The Problem
 
A support chatbot went live on Monday. A customer asked, "Where is order 98123?" and the bot answered confidently: "It will arrive Thursday." The order had been cancelled two days earlier.
 
The model was not lying on purpose. It simply had no door into your systems, so it produced the most plausible sentence. As an architect, the question is not "how do we make the model smarter" but "which of our services may it call, and under what rules?" Tool calling is that door, and like every door in your architecture, it needs a lock.
 
## Code Example
 
### ❌ Without Tools: The Model Guesses
 
```java
// Without Tools The Model Has No Access To Live Data
String answer = chatClient.prompt()
    .user("Where is order 98123?")
    .call()
    .content();
 
// Failure Scenario
// The Model Invents A Plausible Delivery Date
// The Customer Plans Around It
// Support Receives An Angry Call On Friday
```
 
### ✅ Solution 1: Expose A Service As A Described Tool
 
```java
@Component
class OrderTools {
 
    private final OrderService orders;
 
    OrderTools(OrderService orders) {
        this.orders = orders;
    }
 
    // The Description Is The Contract The Model Reads
    @Tool(description = "Get the current status of an order by its order id")
    OrderStatus orderStatus(
            @ToolParam(description = "Numeric order id, for example 98123") long orderId) {
        return orders.statusOf(orderId);
    }
}
 
record OrderStatus(long orderId, String state, LocalDate expectedDelivery) {}
```
 
### ✅ Solution 2: Register Tools Per Use Case, Not Globally
 
```java
// Give Each Assistant Only The Tools Its Job Needs
String answer = chatClient.prompt()
    .system("You are an order support assistant. Use tools for any order question.")
    .user(question)
    .tools(orderTools)
    .call()
    .content();
```
 
### ✅ Solution 3: Treat Tool Arguments As Untrusted Input
 
```java
@Tool(description = "Get the current status of an order by its order id")
OrderStatus orderStatus(long orderId) {
    // The Model Chose This Argument, So Validate And Authorize It
    var user = currentUser.get();
    if (!orders.belongsTo(orderId, user.customerId())) {
        throw new AccessDeniedException("Order Not Found");
    }
    // Return A Small Result Not The Whole Entity
    return orders.statusOf(orderId);
}
```
 
## Why This Matters
 
A tool is a new kind of API consumer: one that reads documentation in natural language and sometimes misreads it. Everything you already know about service design applies with more force. Keep tools narrow, start with read-only operations, validate every argument, and enforce authorization inside the service, never in the prompt. The model decides what to ask; your code decides what is allowed.
 
## Key Takeaway
 
Tool calling turns an LLM into a client of your services. Design tools like public endpoints: small, described, validated, and authorized.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#SpringAI` `#ToolCalling` `#LLM` `#AIArchitecture` `#GenerativeAI` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
