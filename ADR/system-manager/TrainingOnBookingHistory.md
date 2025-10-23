# ADR <NUMBER>: <Title>

**Status:** Accepted

**Date:** 2025-10-23 

**Deciders:** Team Katalysis

**Related Issue/Story:** <link or ID> (optional)

## Context and Problem Statement  
Demands and supply of vehicles coming from booking, checkout and return process get logged onto an
OLTP relational database. The context is about whether to develop a local ML pipeline for prediction
based on general trend, seasonal, weekly of daily variations.

## Decision   
The decision is based on balancing cost, accuracy and testability. It was discussed and the design 
decision to go with a local training service that should run either periodically to support the 
training cost within budget, or to retrain when prediction performance starts to suffer. The 
alternative of making API calls to a commercial AI provider was considered. 

## Alternatives  
- Locally train on historical data: Because data is structured and available from a RDBMS, a better 
	level of accuracy is expected using a local model. The model can be based on RNN, LSTM or 
	transformer based, the latter is known to capture the difference between general trends and
	periodic variations.
- Feed the entire dataset to commercial LLM based system: This means one needs to instruct the AI service
	that the data is in structured form. The problem is that it would create a very large context that
	cost heavily in number of tokens, still with the risk of non-determinism and loss of information in the
	middle of the context.

## Decision Outcome  
- Pros: Cost efficient and potentially accurate due to scope of experiment tracking.
- Cons: Developer expertise is needed for ANN/RNN/LSTM/Transformer, as well as MLOps.

## Links / References  
- ADRs this one depends on or supersedes  
- Supporting documents, diagrams, prototypes  
- External links (articles, patterns, etc.)

