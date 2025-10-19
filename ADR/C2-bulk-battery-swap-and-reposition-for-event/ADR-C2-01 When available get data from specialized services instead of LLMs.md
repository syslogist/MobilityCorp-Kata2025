# ADR C2-01: When available get data from specialized services instead of LLMs

**Status:** Accepted  
**Date:** 2025-10-21  
**Deciders:** Katalysis Team  
**Related Issue/Story:** [event-discovery-use-case.md](../../Problem%20Background/C2-bulk-battery-swap-and-reposition-for-event/event-discovery-use-case.md)

## Context and Problem Statement  
We want to reposition vehicles near event sites. Of course this should be done only if weather is ok. We discover suitable events by calling an LLM. The question is if we should also aks the LLM to consider the weather.

## Decision   
Factors influencing decision:
- ease of integration
- cost
- precision

## Alternatives  
Describe the options that were considered (including the one chosen).  
- Option A – Ask LLM for the weather  
- Option B – Call a specialized weather API  


## Decision Outcome  
We decided to ask the LLM only about events. And to get the weather by calling a specialized API.
It is easier to integrate with a weather API, they are cheap and don't hallucinate.

## Links / References  
- ADRs this one depends on or supersedes  
- Supporting documents, diagrams, prototypes  
- External links (articles, patterns, etc.)

