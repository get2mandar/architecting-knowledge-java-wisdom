# Post #84: Architecture Fitness Functions - Enforcing Design with ArchUnit
 
**Series:** Architecting Knowledge - Java Wisdom Series   
**Published:** November 22, 2026   
**Topic:** Fitness Functions, ArchUnit, Layered Architecture, Dependency Rules, CI Enforcement    
 
---
 
## The Problem
 
The architecture decision was clear and written on the wiki: controllers talk to services, services talk to repositories, nobody skips a layer. Six months later, a deadline-driven controller was injecting a repository directly, and three more followed its example.
 
Nobody was careless. The rule simply lived in a document, and documents do not fail builds. An architectural fitness function is a rule the build can check. If the design is intentional, the compiler's friends should defend it.
 
## Code Example
 
### ❌ Without Enforcement: Rules On A Wiki
 
```java
@RestController
class OrderController {
 
    // Failure Scenario
    // Web Layer Reaches Straight Into Persistence
    // Compiles, Passes Review On A Busy Day, Spreads Quietly
    private final OrderRepository repository;
}
```
 
### ✅ Solution 1: Express The Layers As A Test
 
```java
@AnalyzeClasses(packages = "com.acme.shop")
class ArchitectureTest {
 
    @ArchTest
    static final ArchRule layers = layeredArchitecture()
        .consideringAllDependencies()
        .layer("Web").definedBy("..web..")
        .layer("Service").definedBy("..service..")
        .layer("Persistence").definedBy("..persistence..")
        .whereLayer("Web").mayNotBeAccessedByAnyLayer()
        .whereLayer("Service").mayOnlyBeAccessedByLayers("Web")
        .whereLayer("Persistence").mayOnlyBeAccessedByLayers("Service");
}
```
 
### ✅ Solution 2: Add Small, Focused Rules
 
```java
@ArchTest
static final ArchRule noFieldInjection = noFields()
    .should().beAnnotatedWith(Autowired.class)
    .because("Constructor Injection Keeps Dependencies Explicit");
 
@ArchTest
static final ArchRule domainStaysPure = noClasses()
    .that().resideInAPackage("..domain..")
    .should().dependOnClassesThat().resideInAnyPackage("org.springframework..");
```
 
### ✅ Solution 3: Run Them Like Any Other Test In CI
 
```java
// Rules Run With The Normal Unit Test Phase
// A Violation Fails The Build And Names The Offending Class
// Use .because("...") So The Failure Message Teaches The Reason
```
 
## Why This Matters
 
Architecture erodes through small, reasonable shortcuts. Fitness functions turn your decisions into guardrails that new team members meet on day one, through a failing test and a clear message rather than a hallway correction. Start with two or three rules that protect your most important boundaries, and grow from there.
 
## Key Takeaway
 
A design decision that nothing checks is only a suggestion. Encode it as a test.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#ArchUnit` `#FitnessFunctions` `#EvolutionaryArchitecture` `#CleanArchitecture` `#SoftwareArchitecture` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
