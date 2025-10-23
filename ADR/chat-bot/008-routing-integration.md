# ADR 011: Routing API Integration Strategy

**Status:** Accepted  
**Date:** 2025-10-19  
**Deciders:** Team Katalysis 
**Related Issue/Story:** Routing Service Implementation

## Context and Problem Statement
The system needs reliable routing capabilities including:
- Trip planning
- Navigation
- Travel time estimation
- Distance calculation
- Map visualization
- Real-time traffic
- Points of interest

Key challenges:
- Provider reliability
- Cost management
- Coverage variations
- API changes
- Performance requirements
- Fallback strategies

## Decision
Design a resilient routing service with provider abstraction.

### Forces and Factors:
- Request volume: 500K/day
- Response time: < 200ms
- Availability: 99.99%
- Geographic coverage
- Cost per request
- Cache hit ratio target

## Alternatives

### Option A: Single Provider Direct Integration
- Direct Google Maps integration
- Pros:
  - Simple implementation
  - Best documentation
  - Reliable service
  - Rich features
- Cons:
  - Vendor lock-in
  - Cost exposure
  - Limited control
  - Single point of failure

### Option B: Self-hosted Solution
- OpenStreetMap + routing engine
- Pros:
  - Full control
  - Cost-effective
  - Customizable
  - Data ownership
- Cons:
  - High maintenance
  - Limited features
  - Coverage issues
  - Performance challenges

### Option C: Multi-Provider Strategy (Chosen)
- Primary: Google Maps
- Secondary: Mapbox
- Fallback: OpenStreetMap
- Pros:
  - High reliability
  - Cost optimization
  - Feature richness
  - Risk mitigation
- Cons:
  - Complex integration
  - Higher development effort
  - Cache complexity

## Decision Outcome
We will implement a multi-provider routing service:

1. Service Architecture:
```typescript
interface RoutingProvider {
  getRoute(request: RouteRequest): Promise<Route>;
  getETA(origin: Location, destination: Location): Promise<Duration>;
  searchPOI(query: string, location: Location): Promise<POI[]>;
}

class GoogleMapsProvider implements RoutingProvider {
  async getRoute(request: RouteRequest): Promise<Route> {
    const response = await this.client.directions({
      origin: request.origin,
      destination: request.destination,
      mode: request.mode,
      alternatives: true
    });
    return this.mapToCommonFormat(response);
  }
}

class RoutingService {
  private providers: Map<string, RoutingProvider>;
  private cache: RouteCache;
  
  async getRoute(request: RouteRequest): Promise<Route> {
    // Try cache first
    const cached = await this.cache.get(request);
    if (cached) return cached;
    
    // Try providers in sequence
    for (const provider of this.getOrderedProviders()) {
      try {
        const route = await provider.getRoute(request);
        await this.cache.set(request, route);
        return route;
      } catch (error) {
        this.handleError(error, provider);
        continue;
      }
    }
    
    throw new Error('All routing providers failed');
  }
}
```

2. Caching Strategy:
```typescript
interface RouteCache {
  async get(request: RouteRequest): Promise<Route | null>;
  async set(request: RouteRequest, route: Route): Promise<void>;
  async invalidate(pattern: string): Promise<void>;
}

class RedisCacheImplementation implements RouteCache {
  private redis: Redis;
  
  async get(request: RouteRequest): Promise<Route | null> {
    const key = this.generateKey(request);
    const cached = await this.redis.get(key);
    
    if (!cached) return null;
    
    const route = JSON.parse(cached);
    if (this.isStale(route)) {
      await this.redis.del(key);
      return null;
    }
    
    return route;
  }
  
  private isStale(route: Route): boolean {
    const age = Date.now() - route.timestamp;
    return age > this.getTTL(route.type);
  }
}
```

3. Implementation Details:
   a. Provider Configuration:
   ```yaml
   providers:
     google_maps:
       priority: 1
       quota_daily: 500000
       cost_per_request: 0.005
       timeout_ms: 1000
     
     mapbox:
       priority: 2
       quota_daily: 250000
       cost_per_request: 0.003
       timeout_ms: 1000
     
     osrm:
       priority: 3
       quota_daily: unlimited
       cost_per_request: 0.000
       timeout_ms: 2000
   ```

   b. Cache Settings:
   ```yaml
   cache:
     routes:
       ttl: 300  # 5 minutes
       max_size: 10GB
     
     static_data:
       ttl: 86400  # 24 hours
       max_size: 5GB
     
     poi:
       ttl: 3600  # 1 hour
       max_size: 2GB
   ```

   c. Monitoring:
   ```yaml
   metrics:
     performance:
       - response_time_by_provider
       - cache_hit_ratio
       - error_rate
     
     cost:
       - requests_by_provider
       - cost_per_route
       - cache_savings
     
     quality:
       - route_accuracy
       - provider_comparison
       - user_feedback
   ```

4. Error Handling:
```typescript
class RoutingErrorHandler {
  async handleError(
    error: Error,
    provider: RoutingProvider
  ): Promise<void> {
    await this.circuitBreaker.recordError(provider.name);
    await this.metrics.incrementError(provider.name);
    await this.notifyIfThreshold(provider.name);
    
    if (this.shouldFailover(error)) {
      await this.initiateFailover(provider.name);
    }
  }
}
```

## Links / References
- [Google Maps API](https://developers.google.com/maps/documentation)
- [Mapbox API](https://docs.mapbox.com/)
- [OSRM](http://project-osrm.org/)
- Related ADRs:
  - [API Fault Tolerance](006-api-fault-tolerance.md)
  - [Feature Flagging](008-feature-flagging.md)