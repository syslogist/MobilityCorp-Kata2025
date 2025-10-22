# ADR-007: Personalization and Data Privacy

**Status:** Accepted 
**Date:** 2025-10-19  
**Deciders:** Team Katalysis

## Context
Chatbot recommendations and info require user history, location patterns, and preferences. Privacy compliance (GDPR/CCPA) is essential.

## Decision
Store user preferences, trip histories, and feedback in a secure, encrypted **vector database** (e.g., Pinecone or Weaviate), indexed by anonymized user IDs.  
No raw PII stored outside authentication services; periodic reviews for compliance.

## Alternatives
- **Flat file/database linked to user ID:** Simpler, but privacy/lookup complexity risks.

## Consequences
- **Security:** Better privacy, future compatibility for personalization.
- **Drawbacks:** More complexity for vector search and compliance audits.


