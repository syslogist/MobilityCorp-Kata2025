# ADR 002: Time Series Data Storage for Vehicle Analytics

**Status:** Accepted  
**Date:** 2025-10-23  
**Deciders:** Team Katalysis 
**Related Issue/Story:** Vehicle Analytics Implementation

## Context and Problem Statement
The vehicle analytics system needs to store and process time-series data from thousands of vehicles, including:
- Battery metrics (voltage, current, temperature)
- Tire pressure readings
- Location data
- Maintenance events
- Alert history

Key requirements:
- Fast read/write for real-time dashboards
- Efficient storage for historical data
- Support for complex time-based queries
- Cost-effective long-term storage
- Compliance with data retention policies

## Decision
Select an appropriate time series database solution that can handle high-volume vehicle telemetry data while supporting real-time analytics and historical analysis.

### Forces and Factors:
- Data ingestion rate: ~1000 metrics/second
- Query patterns: Both real-time and historical
- Retention requirements: 
  - Hot data: 7 days
  - Warm data: 90 days
  - Cold data: 2 years
- Performance targets:
  - Query latency < 100ms for recent data
  - Write throughput > 10K points/second

## Alternatives

### Option A: Single PostgreSQL with TimescaleDB
- Extend PostgreSQL with TimescaleDB
- Use traditional RDBMS features
- Pros:
  - Familiar technology
  - SQL interface
  - ACID compliance
- Cons:
  - Limited scalability
  - Higher storage costs
  - Complex maintenance

### Option B: InfluxDB Enterprise
- Purpose-built time series database
- Pros:
  - Optimized for time series
  - Built-in downsampling
  - Good tooling
- Cons:
  - Higher cost
  - Vendor lock-in
  - Limited SQL support

### Option C: Hybrid TimescaleDB + S3 (Chosen)
- TimescaleDB for hot/warm data
- S3 for cold storage
- Automated tiering
- Pros:
  - Cost-effective
  - Flexible scaling
  - SQL interface
  - Simple architecture
- Cons:
  - More complex setup
  - Manual optimization needed

## Decision Outcome
We will implement a hybrid storage solution using TimescaleDB and S3:

1. Hot Data Layer (TimescaleDB):
   - Recent 7 days in main tables
   - Full resolution
   - Continuous aggregates for dashboards
   - In-memory caching for active metrics

2. Warm Data Layer (TimescaleDB):
   - 90 days in compressed chunks
   - 5-minute aggregated rollups
   - Partitioned by vehicle type
   - Automated cleanup

3. Cold Storage (S3):
   - Historical data > 90 days
   - Hourly aggregates
   - Parquet format
   - Lifecycle policies

### Implementation Details
- Automatic data tiering using TimescaleDB policies
- Compression ratio target: 90%
- Retention policies:
  ```sql
  SELECT add_retention_policy(
    'metrics',
    INTERVAL '90 days'
  );
  ```
- Continuous aggregates for common queries:
  ```sql
  CREATE MATERIALIZED VIEW metrics_5min
  WITH (timescaledb.continuous) AS
  SELECT time_bucket('5 minutes', time) AS bucket,
         vehicle_id,
         avg(battery_level) as avg_battery,
         avg(tire_pressure) as avg_pressure
  FROM metrics
  GROUP BY bucket, vehicle_id;
  ```

## Links / References
- [TimescaleDB Documentation](https://docs.timescale.com/)
- [AWS S3 Lifecycle Policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-configuration-examples.html)
- Related: [Edge Processing Strategy](001-edge-processing-strategy.md)