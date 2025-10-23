# ADR 009: Intent Recognition Approach

**Status:** Accepted  
**Date:** 2025-10-23  
**Deciders:** Team Katalysis, AI/ML Team, NLP Specialists  
**Related Issue/Story:** Chatbot Natural Language Understanding

## Context and Problem Statement
The chatbot must accurately understand diverse user requests in the mobility domain, such as:
- "I need an eBike to Dallas Museum by 3 PM"
- "Is there a van available near downtown?"
- "What's the fastest route to the airport?"

Key challenges:
- Domain-specific vocabulary
- Location entity recognition
- Time expressions
- Multi-intent queries
- Contextual understanding
- Multilingual support

Requirements:
- 95%+ accuracy for common intents
- < 100ms response time
- Support for 10+ languages
- Continuous improvement capability

## Decision
Design a hybrid intent recognition system combining traditional NLU with LLM capabilities.

### Forces and Factors:
- Query volume: 100K+/day
- Intent types: 50+
- Entity types: 30+
- Response time requirements
- Training data availability
- Maintenance overhead
- Cost considerations

## Alternatives

### Option A: Pure Rule-based System
- Regular expressions and pattern matching
- Pros:
  - Fast execution
  - Predictable behavior
  - Easy to test
  - Low latency
- Cons:
  - Limited flexibility
  - High maintenance
  - Poor handling of variations
  - No learning capability

### Option B: Pure LLM Approach
- Direct LLM parsing for all queries
- Pros:
  - Handles complexity well
  - Natural language variation
  - Easy to extend
  - Less training data needed
- Cons:
  - Higher latency
  - Inconsistent results
  - Higher cost
  - Black box behavior

### Option C: Hybrid Rasa + LLM Approach (Chosen)
- Rasa for primary intent/entity recognition
- LLM for edge cases and disambiguation
- Pros:
  - Balanced approach
  - Continuous improvement
  - Cost-effective
  - Maintainable
- Cons:
  - More complex architecture
  - Training overhead
  - Integration complexity

## Decision Outcome
We will implement a hybrid system using Rasa NLU with LLM backup:

1. Intent Pipeline Configuration:
```yaml
language: en
pipeline:
  - name: WhitespaceTokenizer
  - name: RegexFeaturizer
  - name: LexicalSyntacticFeaturizer
  - name: CountVectorsFeaturizer
  - name: CountVectorsFeaturizer
    analyzer: char_wb
    min_ngram: 1
    max_ngram: 4
  - name: DIETClassifier
    epochs: 100
    constrain_similarities: true
  - name: EntitySynonymMapper
  - name: FallbackClassifier
    threshold: 0.7
    ambiguity_threshold: 0.1
```

2. Intent Handler Implementation:
```typescript
interface IntentHandler {
  recognize(text: string, context: Context): Promise<IntentResult>;
  train(examples: TrainingExample[]): Promise<void>;
  feedback(result: IntentResult, correct: boolean): Promise<void>;
}

class HybridIntentHandler implements IntentHandler {
  private rasaNLU: RasaClient;
  private llmClient: LLMClient;
  private vectorStore: VectorStore;
  
  async recognize(
    text: string, 
    context: Context
  ): Promise<IntentResult> {
    // Try Rasa first
    const rasaResult = await this.rasaNLU.parse(text);
    
    if (this.isConfident(rasaResult)) {
      return this.enrichWithEntities(rasaResult);
    }
    
    // Fallback to LLM for unclear cases
    return this.llmFallback(text, context);
  }
  
  private isConfident(result: RasaResult): boolean {
    return (
      result.confidence > 0.7 &&
      result.intent_ranking[0].confidence -
      result.intent_ranking[1].confidence > 0.3
    );
  }
}
```

3. Training Pipeline:
```python
class IntentTrainer:
    def __init__(self):
        self.rasa_trainer = RasaTrainer()
        self.vector_store = VectorStore()
        
    async def train_pipeline(self, training_data: List[Example]):
        # Train Rasa model
        await self.rasa_trainer.train(training_data)
        
        # Update vector embeddings
        embeddings = self.compute_embeddings(training_data)
        await self.vector_store.update(embeddings)
        
    async def evaluate_performance(self) -> Metrics:
        return await self.run_test_suite()
```

4. Implementation Details:
   a. Intent Categories:
      ```json
      {
        "booking": {
          "book_vehicle",
          "check_availability",
          "cancel_booking"
        },
        "routing": {
          "find_route",
          "estimate_time",
          "find_parking"
        },
        "support": {
          "report_issue",
          "get_help",
          "contact_support"
        }
      }
      ```

   b. Entity Types:
      - Locations (addresses, landmarks)
      - Times (absolute, relative)
      - Vehicle types
      - Duration expressions
      - Monetary values

   c. Continuous Learning:
      1. User feedback collection
      2. Automated retraining
      3. Performance monitoring
      4. Manual review queue

## Links / References
- [Rasa Documentation](https://rasa.com/docs/)
- [DIET Classifier Paper](https://arxiv.org/abs/2004.09936)
- [Transformer-based NER](https://arxiv.org/abs/1910.03771)
- Related ADRs:
  - [LLM Provider Selection](004-llm-provider-selection.md)
  - [GenAI Output Verification](005-genai-output-verification.md)
  - [Feature Flagging for AI](008-feature-flagging.md)