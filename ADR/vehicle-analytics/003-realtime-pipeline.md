# ADR 003: Real-Time Analytics Pipeline Architecture

**Status:** Accepted  
**Date:** 2025-10-23  
**Deciders:** Team Katalysis  
**Related Issue/Story:** Vehicle Analytics Implementation

## Context and Problem Statement
The vehicle analytics system requires a real-time processing pipeline to handle streaming data from vehicles and provide immediate insights. The system needs to:
- Process high-volume sensor data streams
- Detect anomalies in real-time
- Generate immediate alerts
- Support machine learning inference
- Scale horizontally with fleet growth

## Decision
Design a streaming analytics pipeline architecture that can handle real-time vehicle telemetry while supporting both immediate alerting and complex analytics.

### Forces and Factors:
- Message throughput: ~10K messages/second
- Latency requirements: < 1s for alerts
- Processing complexity: ML inference, pattern matching
- Scalability needs: 2x growth expected yearly
- High availability requirement: 99.99%
- Cost constraints: Optimize for operational costs

## Alternatives

### Option A: Lambda Architecture
- Dual batch and speed layers
- Complex but comprehensive
- Pros:
  - Complete processing
  - Handles reprocessing
  - Accurate results
- Cons:
  - High complexity
  - Maintenance overhead
  - Duplicate processing logic

### Option B: Pure Streaming (Apache Flink)
- Single streaming pipeline
- Event-time processing
- Pros:
  - Simpler architecture
  - Lower latency
  - Unified processing
- Cons:
  - Less flexible
  - Higher resource usage
  - Complex state management

### Option C: Kappa Architecture with Apache Kafka + Spark Streaming (Chosen)
- Log-based streaming
- Replayable event stream
- Stateful processing
- Pros:
  - Balanced approach
  - Good scalability
  - Mature ecosystem
- Cons:
  - Initial setup complexity
  - Needs careful tuning
  - State management overhead

## Decision Outcome
We will implement a Kappa architecture using Apache Kafka and Spark Streaming:

1. Data Ingestion (Kafka):
   ```
   Vehicle Data → Kafka Topics:
   - raw.vehicle.telemetry
   - raw.vehicle.events
   - raw.vehicle.alerts
   ```

2. Stream Processing (Spark Streaming):
   ```
   Structured Streaming Jobs:
   - Real-time metrics calculation
   - Anomaly detection
   - Alert generation
   - ML inference
   ```

3. State Management:
   - Checkpointing to S3
   - RocksDB for local state
   - Kafka for event sourcing

### Implementation Details

1. Kafka Configuration:
   ```yaml
   num.partitions: 24
   retention.ms: 604800000  # 7 days
   cleanup.policy: compact
   min.insync.replicas: 2
   ```

2. Spark Streaming Job:
   ```scala
   val vehicleStream = spark
     .readStream
     .format("kafka")
     .option("kafka.bootstrap.servers", "kafka:9092")
     .option("subscribe", "raw.vehicle.telemetry")
     .load()

   val alertStream = vehicleStream
     .select(from_json($"value", schema).as("data"))
     .withWatermark("timestamp", "10 seconds")
     .groupBy(
       window($"timestamp", "1 minute"),
       $"vehicleId"
     )
     .agg(...)
   ```

3. Scaling Configuration:
   - Kafka: 24 partitions per topic
   - Spark: Dynamic allocation enabled
   - Auto-scaling based on lag metrics

## Links / References
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [Spark Structured Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
- Related: 
  - [Edge Processing Strategy](001-edge-processing-strategy.md)
  - [Time Series Storage](002-time-series-storage.md)