# ADR 004: LLM Provider Selection for MobilityCorp Chatbot

**Status:** Accepted  
**Date:** 2025-10-23  
**Deciders:** Team Katalysis, AI/ML Architects, Security Team  
**Related Issue/Story:** Chatbot Core AI Implementation

## Context and Problem Statement
MobilityCorp's chatbot requires a Large Language Model (LLM) to provide natural dialogue, route planning assistance, and multi-modal user interaction. The selection of an LLM provider is critical as it affects:
- User experience quality
- Operational costs
- System reliability
- Data privacy compliance
- Future scalability

### Key Requirements:
- Response latency < 2 seconds
- 99.9% uptime SLA
- GDPR and CCPA compliance
- Domain-specific knowledge in mobility/transportation
- Multi-language support
- Cost predictability

## Decision
We need to select an LLM provider and design the integration architecture to balance performance, cost, and flexibility.

### Forces and Factors:
- Expected traffic: 100K conversations/day
- Cost sensitivity: $0.01-0.05 per interaction target
- Privacy requirements: EU/US data handling
- Integration complexity
- Model update frequency
- Vendor lock-in risks

## Alternatives

### Option A: Single Vendor (OpenAI GPT-4)
- Direct integration with OpenAI's API
- Pros:
  - Best-in-class performance
  - Regular model updates
  - Extensive documentation
  - Strong developer tools
- Cons:
  - Higher cost per token
  - Limited customization
  - Potential vendor lock-in
  - Data privacy concerns

### Option B: Self-Hosted Open Source (Llama 2)
- Deploy and manage our own model
- Pros:
  - Full control over deployment
  - Lower operational costs
  - Data privacy guaranteed
  - Customization possible
- Cons:
  - High infrastructure costs
  - Maintenance overhead
  - Lower performance vs GPT-4
  - Limited multilingual support

### Option C: Multi-Provider Strategy with Abstraction Layer (Chosen)
- Primary: OpenAI GPT-4
- Backup: Anthropic Claude
- Future option: Self-hosted Llama 2
- Pros:
  - Vendor flexibility
  - Risk mitigation
  - Cost optimization
  - Future-proof architecture
- Cons:
  - Complex integration
  - Higher development effort
  - Testing overhead

## Decision Outcome
We will implement a multi-provider strategy with GPT-4 as primary:

1. Architecture:
```typescript
interface LLMProvider {
  generateResponse(prompt: string, context: Context): Promise<Response>;
  embedText(text: string): Promise<Vector>;
  estimateCost(tokens: number): number;
}

class OpenAIProvider implements LLMProvider {
  // Primary implementation
}

class AnthropicProvider implements LLMProvider {
  // Backup implementation
}

class LLMService {
  private primaryProvider: LLMProvider;
  private backupProvider: LLMProvider;
  
  async getResponse(prompt: string, context: Context): Promise<Response> {
    try {
      return await this.primaryProvider.generateResponse(prompt, context);
    } catch (error) {
      if (this.shouldFailover(error)) {
        return await this.backupProvider.generateResponse(prompt, context);
      }
      throw error;
    }
  }
}
```

2. Implementation Strategy:
   - Abstract LLM calls through provider interface
   - Implement circuit breaker pattern
   - Monitor costs and performance
   - Cache common responses
   - Regular evaluation of alternatives

3. Monitoring and Metrics:
   ```yaml
   metrics:
     - response_time_p95
     - token_cost_per_interaction
     - failover_rate
     - cache_hit_ratio
     - error_rate_by_provider
   ```

4. Cost Control:
   - Token usage limits
   - Caching strategy
   - Automatic fallback for cost spikes

## Links / References
- [OpenAI API Documentation](https://platform.openai.com/docs/)
- [Anthropic Claude Documentation](https://docs.anthropic.com/)
- [Llama 2 Repository](https://github.com/facebookresearch/llama)
- Related ADRs:
  - [Gen AI Output Verification Strategy](./Gen%20AI%20Output%20Verification%20Strategy.md)
  - [API Fault Tolerance-Resiliency](./API%20Fault%20Tolerance-Resiliency.md)