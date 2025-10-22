# ADR-008: Routing API Integration Strategy

**Status:** Accepted 
**Date:** 2025-10-19  
**Deciders:** Team Katalysis

## Context
Users need trip options, directions, travel times, and map data.
External APIs (Google Maps, Mapbox, OpenRouteService) offer similar features but can change SLA, pricing, or region coverage.

## Decision
Use **Google Maps API** as default for routing, with microservice abstraction layer to swap out for alternatives if needed.
Service returns routes, distances, times, costs, also packages error messages in case of upstream API failure.

## Alternatives
- **Direct/private API integration only:** Simpler short-term, but risky if service or terms change.
- **Open-source mapping stack:** More control, but more ops effort.

## Consequences
- **Flexibility:** Can change routing provider with minimal disruption.
- **Drawback:** Needs proper design/testing to ensure interface matches across providers.

