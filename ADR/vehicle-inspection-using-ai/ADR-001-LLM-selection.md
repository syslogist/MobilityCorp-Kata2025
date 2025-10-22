# ADR 001: LLM Model Selection for Damage Detection and Report Generation

**Status:** Proposed  
**Date:** 2025-10-22  
**Deciders:** Architecture Team, DevOps Lead  
**Related Issue/Story:** AI-powered vehicle return inspection system

## Context and Problem Statement

The vehicle return inspection system requires an AI component capable of:
1. Identifying vehicle damage (scratches, dents) from images
2. Extracting license plate information
3. Generating human-readable inspection reports from structured damage data

The team must decide whether to use pre-trained commercial LLM APIs (GPT-4o, Claude) or deploy a self-hosted solution (LLaMA). This decision impacts latency, cost, data privacy, and system reliability. The system handles rental fleet returns across European zones with real-time processing requirements.

**Key Constraints:**
- Data privacy (customer photos and vehicle information)
- Real-time performance requirements
- Cost per inspection
- Team expertise with open-source vs. commercial APIs
- European data residency compliance

## Decision Factors

- **Cost:** Commercial APIs charge per token; self-hosted requires infrastructure investment
- **Latency:** Network calls to external APIs vs. local inference (~100-300ms for LLaMA vs. 500-2000ms for cloud API)
- **Privacy:** On-premises keeps all data within infrastructure; cloud APIs may store data for model improvement
- **Reliability:** Commercial APIs offer SLAs; self-hosted depends on Kubernetes availability
- **Accuracy:** GPT-4o/Claude have broader training; LLaMA may need fine-tuning for damage specifics
- **Maintainability:** Commercial APIs require minimal maintenance; self-hosted needs model updates and monitoring

## Alternatives

### Option A: Commercial LLM API (GPT-4o or Claude)
- **Pros:**
  - State-of-the-art accuracy for damage description and multi-modal analysis (can process images directly)
  - Minimal infrastructure maintenance
  - Built-in safety filters and error handling
  - Immediate deployment
  - Structured output parsing capabilities
- **Cons:**
  - Higher per-inspection cost (~$0.05-0.15 per request)
  - Images and metadata sent to external service (privacy concern)
  - Network dependency; API downtime affects operations
  - Vendor lock-in; rate limiting may impact high-volume scenarios

### Option B: Self-Hosted Open-Source LLM (LLaMA 2, Mistral, or similar)
- **Pros:**
  - Complete data privacy; runs entirely within infrastructure
  - Lower marginal cost after initial GPU investment
  - No rate limiting; scales with local Kubernetes resources
  - Can fine-tune on proprietary damage datasets
  - Full control over model behavior and updates
- **Cons:**
  - Higher initial infrastructure cost (GPU nodes in Kubernetes)
  - Requires ML expertise to fine-tune for damage detection
  - Slower inference than commercial APIs on consumer hardware
  - Maintenance burden; model version management, dependency updates

### Option C: Hybrid Approach (LLaMA for inspection notes + GPT-4o for complex cases)
- **Pros:**
  - Balances cost and accuracy; use fast local LLM for 90% of cases
  - Fallback to GPT-4o for unusual damage patterns or edge cases
  - Reduces cloud API spend significantly
- **Cons:**
  - Increased complexity in routing logic
  - Requires monitoring to detect when to escalate to commercial API

## Decision Outcome

**We will adopt a Hybrid Approach (Option C):**

1. **Primary:** Deploy a self-hosted LLaMA 2 7B or Mistral 7B model on Kubernetes for standard damage assessment and report generation (80-90% of inspections).
2. **Fallback:** Use GPT-4o API (via Azure OpenAI in EU region) for complex cases, ambiguous damage patterns, or when local model confidence is below threshold.
3. **Rationale:**
   - Balances privacy (no sensitive images to external services in most cases)
   - Optimizes cost for high-volume rental fleet operations
   - Provides accuracy insurance through GPT-4o for edge cases
   - Maintains data residency compliance for EU operations

## Implementation Details

- Fine-tune LLaMA model on automotive damage dataset (or use existing models like YOLO damage detection outputs as structured input)
- Implement confidence scoring; route low-confidence results (< 0.7) to GPT-4o for verification
- Use prompt engineering to format LLaMA outputs consistently
- Set cost threshold: if API spend exceeds 10% of infrastructure costs, invest in fine-tuning

## Links / References

- ADR-002: Database Selection (influences data schema for LLM input)
- ADR-003: Image Processing Pipeline (output structure feeds into LLM)
- ADR-004: Kubernetes Deployment (infrastructure for self-hosted LLM)
- External: LLaMA 2 documentation, Azure OpenAI integration patterns
