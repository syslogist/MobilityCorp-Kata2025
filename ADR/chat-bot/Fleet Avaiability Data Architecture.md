# ADR-003: Fleet/Vehicle Availability Data Architecture

**Status:** Accepted 
**Date:** 2025-10-19  
**Deciders:** Team Katalysis

## Context
Real-time availability of mobility vehicles (eBikes, eScooters, cars, vans) affects routing, booking, and service reliability.

## Decision
Implement event-driven architecture for fleet telemetry:
- Use **Kafka** (or cloud pub/sub) for fleet status updates and GPS tracking.
- Microservices: Separate services for booking, status, notifications.
- Staff dashboard listens for events, enabling responsive decision-making.

## Alternatives
- **Monolithic polling:** Simpler, but less scalable, can bottleneck with more vehicles or cities.

## Consequences
- **Scalability/resilience:** Handles growth, fault tolerance, and enables granular analytics.
- **Drawbacks:** Needs infrastructure/DevOps investment.

