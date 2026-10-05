# Post #70: Observability Across Distributed Boundaries - Tracing AI Requests End-to-End
 
**Series:** Architecting Knowledge - Java Wisdom Series  
**Published:** October 4, 2026  
**Topic:** Distributed Tracing, Observability, Correlation IDs, Latency Attribution  
 
---
 
## The Problem
 
A user reports a slow recommendation. The request touched the feature pipeline, three model services, and a ranking service—five log files, five separate timestamps, no shared identifier. You cannot tell which hop added the 3-second delay. Each service is independently instrumented, but the request itself has no continuous thread stitching its journey together. Without a trace, "it's slow somewhere" is the most precise diagnosis you can offer.
 
## Code Example
 
### ❌ Without Tracing - Disconnected Logs
 
```java
// Each Service Logs in Isolation, No Shared Context
@Service
public class UntracedFeatureService {
 
    public UserFeatures getFeatures(String userId) {
        long start = System.currentTimeMillis();
        UserFeatures features = computeFeatures(userId);
        logger.info("Computed Features in {}ms", System.currentTimeMillis() - start);
        return features;
        // Log Has No Link to the Recommendation Request That Triggered It
    }
}
 
// Failure Scenario:
// 1. Recommendation Request Arrives, Takes 3 Seconds
// 2. Feature Service Log: "Computed in 50ms" (Fine)
// 3. Model Service Log: "Predicted in 40ms" (Fine)
// 4. Ranking Service Log: "Ranked in 2900ms" (The Culprit!)
// 5. But No ID Connects These Three Log Lines to One Request
// 6. Engineer Manually Correlates by Timestamp - Unreliable at Scale
```
 
### ✅ Solution 1: Correlation ID - Thread the Request
 
```java
@Component
public class CorrelationIdFilter extends OncePerRequestFilter {
 
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                     FilterChain chain) throws IOException, ServletException {
        String traceId = Optional.ofNullable(req.getHeader("X-Trace-Id"))
            .orElse(UUID.randomUUID().toString());
 
        MDC.put("traceId", traceId);  // Attaches to Every Log Line on This Thread
        res.setHeader("X-Trace-Id", traceId);
 
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();
        }
    }
}
 
// Downstream Call Propagates the Same ID
public UserFeatures callFeatureService(String userId) {
    return restTemplate.exchange(
        RequestEntity.get(featureServiceUrl)
            .header("X-Trace-Id", MDC.get("traceId"))  // Same Trace Continues
            .build(),
        UserFeatures.class
    ).getBody();
}
```
 
### ✅ Solution 2: Distributed Tracing - Spans Across Boundaries
 
```java
/*
TRACE STRUCTURE:
  - Trace: One End-to-End Request (Recommendation Call)
  - Span: One Unit of Work Within It (Feature Fetch, Model Call, Ranking)
  - Parent-Child Links: Spans Nest to Show the Call Tree
*/
@Service
public class TracedRecommendationService {
 
    private final Tracer tracer;
 
    public List<Product> getRecommendations(User user) {
        Span parentSpan = tracer.nextSpan().name("get-recommendations").start();
 
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(parentSpan)) {
            UserFeatures features = fetchFeaturesTraced(user);
            List<Product> ranked = rankTraced(features);
            return ranked;
        } finally {
            parentSpan.end();  // Records Total Duration
        }
    }
 
    private UserFeatures fetchFeaturesTraced(User user) {
        Span span = tracer.nextSpan().name("fetch-features").start();
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            return featureService.getFeatures(user.getId());
        } finally {
            span.end();  // Shows Up as a Child of the Parent Span
        }
    }
}
 
// Resulting Trace (Viewed in Zipkin/Jaeger):
// get-recommendations       [====================] 3000ms
//   fetch-features          [==] 50ms
//   predict-fraud-score     [=] 40ms
//   rank-products           [==================] 2900ms  <- Bottleneck Visible
```
 
### ✅ Solution 3: Structured Latency Attribution - Where Time Actually Goes
 
```java
@Service
public class LatencyAttributionService {
 
    public record StageLatency(String stage, long durationMs) {}
 
    public List<Product> getRecommendationsWithAttribution(User user) {
        List<StageLatency> stages = new ArrayList<>();
 
        UserFeatures features = timed("feature-fetch", stages,
            () -> featureService.getFeatures(user.getId()));
 
        FraudScore fraud = timed("fraud-check", stages,
            () -> fraudService.predict(user, features));
 
        List<Product> ranked = timed("ranking", stages,
            () -> rankingService.rank(features));
 
        logger.info("Latency Breakdown: {}", stages);
        return ranked;
    }
 
    private <T> T timed(String stage, List<StageLatency> stages, Supplier<T> work) {
        long start = System.currentTimeMillis();
        T result = work.get();
        stages.add(new StageLatency(stage, System.currentTimeMillis() - start));
        return result;
    }
}
 
// Output: [feature-fetch: 50ms, fraud-check: 40ms, ranking: 2900ms]
// Attribution Is Immediate - No Cross-Service Correlation Needed for This View
```
 
## Why This Matters
 
Distributed systems fail your intuition about latency because no single log tells the whole story. A correlation ID is the minimum viable observability: it lets you grep every service's logs for one request and reconstruct the timeline manually. Distributed tracing goes further, building a parent-child span tree automatically, so the bottleneck is visible in a single trace view rather than assembled by hand. Latency attribution complements both by giving you a per-stage breakdown inside a single service call, useful when the bottleneck is internal rather than cross-service. Together, they turn "it's slow somewhere" into "ranking took 2900ms of a 3000ms request."
 
## Key Takeaway
 
Every request that crosses a service boundary needs a propagated trace ID from the first hop to the last. Instrument spans around each meaningful unit of work, not just at the service entry point. Attribute latency per stage so bottlenecks are visible without needing to a manually stitch logs together. Observability isn't logging more—it's making the request's journey reconstructable.
 
---
 
**Tags:** `#Java` `#JavaWisdom` `#DistributedTracing` `#Observability` `#CorrelationId` `#SpringBoot` `#MicrometerTracing` `#DistributedSystems` `#Microservices` `#Reliability` `#LatencyAttribution` `#JUG` `#JavaUserGroup` `#JUGIndia` `#JavaDevs` `#JavaDeveloper` `#SpringDeveloper` `#BackendDeveloper` `#EnterpriseJava` `#ServerSide` `#JavaCommunity` `#CommunityDayForJava` `#DevCommunity` `#TechCommunity`
