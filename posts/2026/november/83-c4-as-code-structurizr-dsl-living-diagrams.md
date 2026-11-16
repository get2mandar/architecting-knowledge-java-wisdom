# Post #83: C4 as Code - Structurizr DSL for Living Diagrams
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** November 18, 2026  
**Topic:** C4 Model, Structurizr DSL, Architecture as Code, Living Documentation, CI Pipelines  
 
---
 
## The Problem
 
The onboarding deck had a beautiful architecture diagram. A new developer spent two days trying to find the "Notification Service" on it, only to learn it had been merged into the Order Service eight months ago. The diagram was drawn once, in a slide tool, and never touched again.
 
Diagrams rot when they live outside the code review loop. The C4 model gives you the right levels of zoom; Structurizr DSL gives you a way to keep them honest by describing the model once, as text, next to the code.
 
## Code Example
 
### ❌ Without Code: A Picture With No Owner
 
```text
// Architecture-Final-v7-REAL.pptx
// Failure Scenario
// No Diff, No Review, No History
// Nobody Knows Which Boxes Are Still True
```
 
### ✅ Solution 1: Describe The Model Once
 
```text
workspace "Shop" {
    model {
        customer = person "Customer"
        shop = softwareSystem "Shop Platform" {
            api   = container "Order API" "Handles Orders" "Spring Boot"
            db    = container "Order DB" "Stores Orders" "PostgreSQL"
        }
        payments = softwareSystem "Payment Gateway" "External"
 
        customer -> api "Places Orders Using"
        api -> db "Reads And Writes"
        api -> payments "Charges Cards Using"
    }
}
```
 
### ✅ Solution 2: Generate Many Views From One Model
 
```text
views {
    systemContext shop "Context" {
        include *
        autolayout lr
    }
    container shop "Containers" {
        include *
        autolayout lr
    }
}
```
 
### ✅ Solution 3: Keep It In The Repo And Export In CI
 
```text
# Store workspace.dsl Beside The Code, Review It In Pull Requests
structurizr-cli validate -workspace docs/workspace.dsl
structurizr-cli export -workspace docs/workspace.dsl -format mermaid
```
 
## Why This Matters
 
C4 is only valuable if it stays true. When the model is text, a pull request that adds a new container can also update the diagram, and reviewers see both in one diff. Different audiences get different views from the same source, so you stop maintaining five drawings that quietly disagree.
 
## Key Takeaway
 
Treat architecture diagrams like source code: versioned, reviewed, validated, and generated.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#C4Model` `#Structurizr` `#ArchitectureAsCode` `#SoftwareArchitecture` `#LivingDocumentation` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
