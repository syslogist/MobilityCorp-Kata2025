# ADR C2-02: Repositioning vehicles in bulk to events is done with Admin approval

**Status:** Accepted  
**Date:** 2025-10-21  
**Deciders:** Katalysis Team  
**Related Issue/Story:** [event-discovery-use-case.md](../../Problem%20Background/C2-bulk-battery-swap-and-reposition-for-event/event-discovery-use-case.md)

## Context and Problem Statement  
Should we include an Admin in the loop to approve an event? Or should we take LLM suggestion as a decision?

## Decision   
Factors influencing decision:
- it is expensive to reposition many vehicles
- the LLM does not have all context an Admin has e.g. info about maintenance crew issues
- large events don't happen that often, allowing for a human decision maker in the loop

## Alternatives  
Describe the options that were considered (including the one chosen).  
- Option A – LLM suggestions are treated as decisions  
- Option B – An admin transforms a proposed event in an approved one


## Decision Outcome  
Repositioning vehicles to events is done with Admin approval

## Links / References  
- ADRs this one depends on or supersedes  
- Supporting documents, diagrams, prototypes  
- External links (articles, patterns, etc.)

