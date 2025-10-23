# ADR 007: Fleet Availability Data Architecture

**Status:** Accepted  
**Date:** 2025-10-19  
**Deciders:** Team Katalysis 
**Related Issue/Story:** Real-time Fleet Management Implementation

## Context and Problem Statement
MobilityCorp needs to manage real-time availability of diverse vehicle types:
- Electric bikes
- Electric scooters
- Cars
- Vans

Key challenges include:
- Real-time location tracking
- Battery status monitoring
- Booking state management
- Maintenance status
- Multi-city scalability
- Analytics requirements

The system must support:
- < 500ms latency for availability checks
- 100K+ vehicles across cities
- 1M+ daily booking requests
- Real-time fleet analytics
- Historical data analysis

## Decision
Design a scalable, event-driven architecture for fleet telemetry and availability management.

### Forces and Factors:
- Data volume: ~1TB/day
- Update frequency: 5-30 seconds per vehicle
- Concurrent users: 100K+
- Geographic distribution
- Regulatory compliance
- Cost optimization

## Alternatives

### Option A: Monolithic Polling System
- Centralized database with polling
- Pros:
  - Simple implementation
  - Easy to debug
  - Lower initial complexity
- Cons:
  - Poor scalability
  - High latency
  - Database bottlenecks
  - Limited real-time capability

### Option B: Pure Microservices
- Fully distributed services
- Pros:
  - High scalability
  - Service isolation
  - Independent scaling
- Cons:
  - Complex orchestration
  - Data consistency challenges
  - Higher operational cost

### Option C: Event-Driven Architecture with CQRS (Chosen)
- Kafka-based event backbone
- Command Query Responsibility Segregation
- Pros:
  - Optimal scalability
  - Real-time capabilities
  - Clear data flow
  - Analytics friendly
- Cons:
  - Initial complexity
  - Learning curve
  - Infrastructure needs

## Decision Outcome
We will implement an event-driven architecture using Apache Kafka and CQRS:

1. Event Schema:
```protobuf
message VehicleEvent {
  string vehicle_id = 1;
  EventType type = 2;
  Location location = 3;
  VehicleStatus status = 4;
  BatteryInfo battery = 5;
  google.protobuf.Timestamp timestamp = 6;
}

message Location {
  double latitude = 1;
  double longitude = 2;
  float accuracy = 3;
}

message BatteryInfo {
  float level = 1;
  float range_km = 2;
  BatteryStatus status = 3;
}
```

2. Kafka Topics Structure:
```yaml
topics:
  vehicle_telemetry:
    partitions: 24
    replication: 3
    retention: 7d
    
  vehicle_availability:
    partitions: 12
    replication: 3
    retention: 1d
    
  booking_events:
    partitions: 24
    replication: 3
    retention: 30d
```

3. Microservices Architecture:
```typescript
interface FleetService {
  // Command handlers
  async bookVehicle(vehicleId: string, userId: string): Promise<Booking>;
  async updateVehicleStatus(vehicleId: string, status: VehicleStatus): Promise<void>;
  
  // Query handlers
  async getAvailableVehicles(location: Location, radius: number): Promise<Vehicle[]>;
  async getVehicleStatus(vehicleId: string): Promise<VehicleStatus>;
}

class KafkaFleetService implements FleetService {
  private producer: KafkaProducer;
  private consumer: KafkaConsumer;
  private cache: RedisCache;
  
  async bookVehicle(vehicleId: string, userId: string): Promise<Booking> {
    // Optimistic locking pattern
    const version = await this.cache.get(`vehicle:${vehicleId}:version`);
    await this.producer.send({
      topic: 'booking_events',
      key: vehicleId,
      value: {
        type: 'BOOK',
        vehicleId,
        userId,
        version
      }
    });
    return this.waitForBookingConfirmation(vehicleId);
  }
}
```

4. Implementation Details:
   a. Data Flow:
      ```mermaid
      graph TD
        V[Vehicle IoT] -->|Status Updates| K[Kafka]
        K -->|Real-time| AS[Availability Service]
        K -->|Analytics| TS[Time Series DB]
        AS -->|Cache| R[Redis]
        R -->|Query| API[API Gateway]
        API -->|Subscribe| WS[WebSocket]
      ```

   b. Caching Strategy:
      - Redis for real-time vehicle status
      - Geospatial indexing for location queries
      - TTL-based cache invalidation

   c. Monitoring Setup:
      ```yaml
      metrics:
        latency:
          - event_processing_time
          - query_response_time
          - booking_completion_time
        throughput:
          - events_per_second
          - bookings_per_minute
          - status_updates_per_minute
        reliability:
          - event_delivery_success_rate
          - cache_hit_ratio
          - booking_success_rate
      ```

## Links / References
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Redis Geospatial](https://redis.io/commands/geosearch)
- [CQRS Pattern](https://martinfowler.com/bliki/CQRS.html)
- Related ADRs:
  - [API Fault Tolerance](006-api-fault-tolerance.md)
  - [Real-time Analytics Pipeline](003-realtime-pipeline.md)