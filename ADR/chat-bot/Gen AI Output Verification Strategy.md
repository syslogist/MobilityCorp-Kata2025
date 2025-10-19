# ADR-XXX: GenAI Output Verification Strategy

## Context
LLM/GenAI can produce misleading, biased, or hallucinated outputs, especially for travel safety or payments.

## Decision
Implement automated pipeline for output sampling/verification:
- Route critical business flows to deterministic templates or human approval when ambiguity detected ("show booking options" vs "make a booking").
- Log all GenAI responses and sample periodically for human review.

## Alternatives
- **Always trust GenAI output:** Fast, but risks business impact or user trust.

## Consequences
- **Quality Assurance:** Safer, more reliable, especially for high-consequence actions.
- **Drawbacks:** More ops for review, possible (slight) slowing of some user flows.

## Status
Accepted
