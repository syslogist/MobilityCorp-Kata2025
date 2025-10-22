# ADR 002: Database Selection - Relational vs. Non-Relational

**Status:** Proposed  
**Date:** 2025-10-22  
**Deciders:** Architecture Team, Backend Lead  
**Related Issue/Story:** AI-powered vehicle return inspection system

## Context and Problem Statement

The inspection system needs to store:
1. **Structured metadata:** VIN, license plate, timestamp, GPS location, inspection ID, rental agreement ID, user ID
2. **Semi-structured data:** Damage assessment results (list of damage objects with coordinates, severity, type)
3. **Image references:** S3/Blob storage URIs, image checksums, image metadata (resolution, capture device)
4. **Audit trails:** Inspection history, dispute resolutions, human reviews

The system must support:
- Real-time queries (check if vehicle already inspected)
- Analytics (damage trends, fleet maintenance patterns)
- Fast lookups by VIN and inspection date
- Complex joins (inspection ↔ rental ↔ user ↔ damage records)
- EU data residency compliance

**Key Constraints:**
- 5,000 bikes/scooters in European zones; estimated 50-100 inspections per day per zone
- Sub-100ms query latency required for dashboard
- Data must remain in EU (PostgreSQL on-premises or EU Cloud regions)
- Need audit trail for rental disputes

## Decision Factors

- **Data Structure:** Mostly structured (VIN, timestamps, damage coords) with some flexibility (damage types vary)
- **Query Patterns:** Complex joins likely (rental + inspection + damage history); strong consistency required
- **Scalability:** Moderate volume (5K vehicles, 5K-10K inspections/month) doesn't require massive horizontal scaling
- **Compliance:** EU GDPR; easier with traditional relational databases (clear data ownership, easy data deletion)
- **Team Expertise:** Development team familiarity with PostgreSQL vs. MongoDB/DynamoDB
- **Latency:** Relational databases excel at complex queries; NoSQL faster for simple lookups but slower for joins
- **Operational Overhead:** PostgreSQL requires backup/replication; DynamoDB is managed but vendor-locked

## Alternatives

### Option A: PostgreSQL (Relational)
- **Pros:**
  - Perfect for structured inspection metadata and relationships
  - Strong ACID guarantees; transaction support critical for rental disputes
  - Complex queries (damage history per vehicle, stats by zone) simple to write
  - EU-friendly: runs on-premises or Azure EU; full GDPR control
  - Proven with Kubernetes via StatefulSets with persistent volumes
  - Free and open-source
  - JSON/JSONB columns provide flexibility for damage data without schema changes
- **Cons:**
  - Requires DBA expertise for backups, replication, scaling
  - Schema migrations needed if damage types expand significantly
  - Single-machine bottleneck (though replication/sharding possible)

### Option B: MongoDB (Document-Oriented NoSQL)
- **Pros:**
  - Flexible schema; damage objects stored directly as documents
  - Good for rapid iteration on damage data structure
  - Horizontal scaling via sharding
  - Developer-friendly, schema-less approach
- **Cons:**
  - Weak transaction support (multi-document ACID added in v4.0, still limited)
  - Harder to query across collections (e.g., find all inspections for a rental with disputes)
  - More memory-intensive than PostgreSQL for same data
  - EU deployments require careful configuration for data residency
  - Operational complexity in Kubernetes similar to PostgreSQL

### Option C: DynamoDB (AWS NoSQL Managed)
- **Pros:**
  - Fully managed by AWS; no operational overhead
  - Infinitely scalable
  - Built-in encryption and backups
- **Cons:**
  - **Critical:** AWS DynamoDB has limited EU presence; data residency compliance difficult
  - Expensive for complex queries (requires custom indexing strategies)
  - Vendor lock-in
  - Cold-start latency unpredictable
  - Not suitable for audit-heavy systems (limited query flexibility)

### Option D: Hybrid Approach (PostgreSQL + Redis Cache)
- **Pros:**
  - PostgreSQL for authoritative data and audit trails
  - Redis for caching frequent queries (inspection status, recent damage alerts)
  - Maintains EU data residency
  - Best query performance for dashboard
- **Cons:**
  - Adds operational complexity (cache invalidation, dual system monitoring)
  - Increases infrastructure cost (two databases)

## Decision Outcome

**We will adopt PostgreSQL with JSONB columns and Redis caching (Option D):**

1. **Primary Data Store:** PostgreSQL running in Kubernetes StatefulSet with persistent volumes in EU region
2. **Purpose:**
   - Inspection metadata (ID, VIN, timestamp, GPS, rental_id, user_id)
   - Damage assessment results stored as JSONB (allows flexible damage object structures)
   - Audit trails and dispute history
   - User and vehicle registrations
3. **Caching Layer:** Redis for:
   - Recent inspection status checks (5-minute TTL)
   - Dashboard aggregations (damage trends, fleet health)
   - Session management
4. **Rationale:**
   - PostgreSQL JSONB gives relational structure AND flexibility for damage data evolution
   - Maintains transaction integrity for rental disputes (critical for business logic)
   - EU data residency fully under control
   - Strong audit trail support (all mutations logged)
   - Redis reduces load on PostgreSQL for read-heavy dashboard queries

## Schema Outline (PostgreSQL)

```sql
-- Core tables
CREATE TABLE vehicles (
    id UUID PRIMARY KEY,
    vin VARCHAR(17) UNIQUE NOT NULL,
    license_plate VARCHAR(20),
    rental_zone VARCHAR(50),
    created_at TIMESTAMP
);

CREATE TABLE inspections (
    id UUID PRIMARY KEY,
    vehicle_id UUID REFERENCES vehicles(id),
    timestamp TIMESTAMP NOT NULL,
    gps_location POINT,
    damage_assessment JSONB,
    confidence_score DECIMAL,
    created_by_user_id UUID,
    created_at TIMESTAMP
);

CREATE TABLE damage_records (
    id UUID PRIMARY KEY,
    inspection_id UUID REFERENCES inspections(id),
    damage_type VARCHAR(50),
    severity ENUM('minor', 'moderate', 'severe'),
    location POINT,
    image_uri VARCHAR(500),
    created_at TIMESTAMP
);

-- Indexes for performance
CREATE INDEX idx_inspections_vehicle_timestamp 
ON inspections(vehicle_id, timestamp DESC);

CREATE INDEX idx_damage_severity 
ON damage_records(severity);
```

## Links / References

- ADR-001: LLM Model Selection (damage assessment feeds into JSONB storage)
- ADR-003: Image Processing Pipeline (image URIs stored here)
- ADR-004: Kubernetes Deployment (PostgreSQL StatefulSet configuration)
- Related: GDPR compliance documentation, backup/restore procedures
- External: PostgreSQL JSONB documentation, Redis caching patterns for Kubernetes
