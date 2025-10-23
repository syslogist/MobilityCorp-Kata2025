# ADR 006: API Fault Tolerance and Resiliency Strategy

**Status:** Accepted  
**Date:** 2025-10-23  
**Deciders:** Team Katalysis, Platform Reliability Team  
**Related Issue/Story:** System Reliability Implementation

## Context and Problem Statement
The chatbot system relies on multiple third-party APIs for critical functionality:
- Weather data
- Points of Interest (POI)
- Routing services
- Event information
- Payment processing

These dependencies create several challenges:
- API outages and downtime
- Network latency variations
- Rate limiting and quotas
- API version changes
- Data inconsistency

Key requirements:
- 99.99% system availability
- < 2s response time
- Graceful degradation
- Cost management

## Decision
Design a comprehensive fault tolerance and resiliency strategy for all API integrations.

### Forces and Factors:
- Multiple API dependencies (10+ services)
- Variable API reliability (95-99.9%)
- Cost of redundancy
- Complexity management
- Response time requirements
- Recovery time objectives

## Alternatives

### Option A: Basic Retry Logic
- Simple retry with backoff
- Pros:
  - Easy to implement
  - Low overhead
  - Simple maintenance
- Cons:
  - Limited protection
  - No degradation strategy
  - Poor resource management

### Option B: Full API Redundancy
- Duplicate all services
- Pros:
  - Maximum availability
  - Real-time failover
  - Load distribution
- Cons:
  - High cost
  - Complex management
  - Integration overhead

### Option C: Circuit Breaker with Fallbacks (Chosen)
- Resilience4j implementation
- Cached fallbacks
- Monitoring integration
- Pros:
  - Balanced approach
  - Resource protection
  - Graceful degradation
- Cons:
  - Implementation complexity
  - Cache management
  - Configuration overhead

## Decision Outcome
We will implement a circuit breaker pattern with intelligent fallbacks:

1. Circuit Breaker Configuration:
```java
@Configuration
public class ResilienceConfig {
    
    @Bean
    public CircuitBreakerConfig weatherApiConfig() {
        return CircuitBreakerConfig.custom()
            .failureRateThreshold(50)
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .permittedNumberOfCallsInHalfOpenState(10)
            .slidingWindowSize(100)
            .recordExceptions(IOException.class, TimeoutException.class)
            .build();
    }
    
    @Bean
    public TimeLimiterConfig timeLimiterConfig() {
        return TimeLimiterConfig.custom()
            .timeoutDuration(Duration.ofSeconds(2))
            .build();
    }
}
```

2. Fallback Strategy:
```typescript
interface APIService<T> {
    primary(): Promise<T>;
    fallback(): Promise<T>;
    cache(): Promise<T>;
}

class WeatherService implements APIService<WeatherData> {
    @CircuitBreaker(name = "weather")
    @TimeLimiter(name = "weather")
    async primary(): Promise<WeatherData> {
        return await weatherApi.getForecast();
    }
    
    async fallback(): Promise<WeatherData> {
        const cached = await cache.get('weather');
        if (cached) return cached;
        return getDefaultWeather();
    }
}
```

3. Monitoring Setup:
```yaml
metrics:
  circuit_breaker:
    - failure_rate
    - slow_call_rate
    - state_transition
    - recovery_time
  
  fallback:
    - cache_hit_ratio
    - degraded_response_count
    - error_distribution
```

4. Implementation Details:
   a. Circuit Breaker States:
      - CLOSED: Normal operation
      - OPEN: Stop calls, use fallback
      - HALF_OPEN: Testing recovery

   b. Fallback Hierarchy:
      1. Secondary API provider
      2. Cached data (with TTL)
      3. Default responses
      4. Graceful degradation

   c. Recovery Strategy:
      - Exponential backoff
      - Health checks
      - Partial service restoration

## Links / References
- [Resilience4j Documentation](https://resilience4j.readme.io/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Fault Tolerance Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/category/resiliency)
- Related ADRs:
  - [LLM Provider Selection](004-llm-provider-selection.md)
  - [GenAI Output Verification](005-genai-output-verification.md)