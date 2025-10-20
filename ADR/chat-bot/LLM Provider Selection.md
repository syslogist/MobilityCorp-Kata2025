# ADR-006: LLM Provider Selection

**Status:** Accepted 
**Date:** 2025-10-19  
**Deciders:** Team Katalysis

## Context
MobilityCorp’s chatbot needs a robust Large Language Model (LLM) for natural dialogue, route planning, and multi-modal user interaction.
Key requirements:
- High accuracy and contextual fluency
- Ability to tailor responses for mobility/travel domain
- Data privacy and business agility in case of pricing/model changes

## Decision
Adopt **OpenAI GPT-4** (via API) as the initial LLM provider for production use, leveraging its superior language capabilities and ecosystem.  
**Design:** Wrap the LLM calls in an abstraction/microservice, enabling rapid switch to alternate providers (Google Gemini, Anthropic, or self-hosted Llama 2) if pricing, reliability, or compliance requirements change.

## Alternatives
- **Single vendor integration:** Simpler, but risks of lock-in and unplanned changes if the provider modifies terms or model quality.
- **Self-hosted LLM only:** Full control, but increased ops complexity and maintenance overhead.

## Consequences
- **Pros:** Flexibility, minimal vendor lock-in, future-proofing against cost and compliance risks.
- **Cons:** More upfront work for abstraction, some possible integration complexity if switching LLMs.

