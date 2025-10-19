# ADR-XXX: Intent Recognition Approach

## Context
Accurate understanding of user requests is critical (e.g., "I want an eBike to Dallas Museum") for recommending routes, bookings, and contextual actions.
Mobility-specific intent/entity coverage needed: place names, time, transport preferences.

## Decision
Use **Rasa NLU** for primary intent/entity parsing (with continual retraining), backed by LLM for ambiguous/edge-case queries.  
Feedback from live queries will be used to continually improve intent recognition.

## Alternatives
- **Pure Rule-based:** Fast, reliable for known cases, but fails at complex/novel queries.
- **Pure GenAI/LLM:** Handles variety, but can be inconsistent and harder to test for mobility domain specifics.

## Consequences
- **Accuracy:** Well-defined use cases covered by Rasa; fallback to LLM for rare/complex cases.
- **Trade-off:** Slightly increased infra complexity; need for ongoing retraining/audit.

## Status
Accepted
