# Post #82: Human-in-the-Loop Guardrails - Approval Gates for AI Actions
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** November 15, 2026  
**Topic:** Human-in-the-Loop, Approval Gates, Risk Tiers, Idempotency, Audit Trail, Spring AI Tools  
 
---
 
## The Problem
 
After the order-status assistant succeeded, the team got ambitious: "Let the assistant issue refunds too." A week later a customer wrote, "I'm so disappointed, I want my money back," and the assistant refunded a 40,000 rupee order on emotion alone.
 
Reading is cheap to get wrong. Writing is not. The architectural move is old and familiar from payment workflows: separate the proposal from the execution, and let a human own the final step for anything that moves money, changes data, or cannot be undone.
 
## Code Example
 
### ❌ Without A Gate: The Model Executes Directly
 
```java
@Tool(description = "Refund an order")
void refund(long orderId, BigDecimal amount) {
    // Failure Scenario
    // A Vague Message Becomes A Real Money Movement
    // No Review, No Audit Trail, No Undo
    payments.refund(orderId, amount);
}
```
 
### ✅ Solution 1: Classify Actions By Risk
 
```java
enum ActionRisk {
    READ_ONLY,      // Execute Immediately
    REVERSIBLE,     // Execute And Log
    IRREVERSIBLE    // Requires Human Approval
}
 
record ProposedAction(String id, String type, Map<String, Object> params,
                      ActionRisk risk, Instant expiresAt) {}
```
 
### ✅ Solution 2: The Tool Proposes, It Never Executes
 
```java
@Tool(description = "Request a refund for an order. A human must approve it.")
String requestRefund(long orderId, BigDecimal amount) {
    var proposal = new ProposedAction(
        UUID.randomUUID().toString(), "REFUND",
        Map.of("orderId", orderId, "amount", amount),
        ActionRisk.IRREVERSIBLE, Instant.now().plus(Duration.ofHours(24)));
 
    pendingActions.save(proposal);
    // The Model Only Learns That A Review Is Pending
    return "Refund request " + proposal.id() + " is waiting for approval";
}
```
 
### ✅ Solution 3: Execute Only Through An Authenticated Approval
 
```java
@PostMapping("/actions/{id}/approve")
@PreAuthorize("hasRole('SUPPORT_LEAD')")
void approve(@PathVariable String id, Principal approver) {
    var action = pendingActions.findValid(id);        // Rejects Expired Or Used Ids
    payments.refund(action.params(), idempotencyKey(id));
    audit.record(id, approver.getName(), "APPROVED");
    pendingActions.markDone(id);
}
```
 
## Why This Matters
 
Guardrails inside the prompt are suggestions. Guardrails inside the architecture are guarantees. An approval gate gives you an audit trail, a place for authorization, expiry for stale requests, and idempotency so a retry never refunds twice. It also lets you relax the gate later, tier by tier, as trust in the system grows.
 
## Key Takeaway
 
Let AI propose and let humans commit. Put the approval gate in code, not in the prompt.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#SpringAI` `#HumanInTheLoop` `#AIGovernance` `#AIArchitecture` `#SoftwareArchitecture` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
