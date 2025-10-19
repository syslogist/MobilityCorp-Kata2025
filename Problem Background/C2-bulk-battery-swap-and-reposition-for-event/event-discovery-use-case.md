# C2 Bulk Battery Swap And Reposition For Event

## Overview

A great way to solve C2 challenge is to reposition ebikes and scooters in parking locations near large events.
A maintenance crew will collect bikes and scooters, swap their batteries with full ones and reposition them in parking locations near the event site prior to its end.

## Container Diagram 

The following diagram offers an overview of the containers providing this use case and how they interact with nearby systems.

![Container Diagram](/Diagrams/C2-bulk-battery-swap-and-reposition-for-event/event-discovery-container-diagram.drawio.png)


## Main Flow
1. Events are detected in serviced zone by periodically making a call to an LLM API using function calling https://platform.openai.com/docs/guides/function-calling
We will be specific in the prompt with respect to what events we want e.g.:
- include only events happening tomorrow
- in a defined zone
- where alcohol is not consumed
- and return the response in a specified function call format
- we will provide examples of previous events and how a response would have looked

### Declare the tool (function) schema
<details>
<summary>Show JSON schema</summary>

```json
[
  {
    "type": "function",
    "function": {
      "name": "save_events",
      "description": "Persist a batch of events found for a given date and location.",
      "parameters": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "items": {
            "type": "array",
            "minItems": 1,
            "items": {
              "type": "object",
              "additionalProperties": false,
              "required": ["event_address", "participants"],
              "properties": {
                "event_address": {
                  "type": "string",
                  "description": "Full street address for the event (include venue name if known)."
                },
                "participants": {
                  "type": "integer",
                  "minimum": 1,
                  "description": "Estimated number of participants/attendees."
                }
              }
            },
            "description": "List of large events to save."
          }
        },
        "required": ["items"]
      }
    }
  }
]
```
</details> 

### Sample response
<details>
<summary>Show sample response</summary>

```json
{
  "name": "save_events",
  "arguments": {
    "items": [
      { "event_address": "The O2, Peninsula Square, London SE10 0DX, UK", "participants": 20000 },
      { "event_address": "Wembley Stadium, London HA9 0WS, UK", "participants": 90000 },
      { "event_address": "ExCeL London, Royal Victoria Dock, London E16 1XL, UK", "participants": 15000 },
      { "event_address": "Tottenham Hotspur Stadium, 782 High Rd, London N17 0BX, UK", "participants": 62000 },
      { "event_address": "The SSE Arena, Arena Square, Engineers Way, London HA9 0AA, UK", "participants": 12500 }
    ]
  }
}
```
</details> 

2. The returned function request is used to call save_events on event discovery service
3. Event discovery service will loop over the events and enrich the data with weather data by calling a specialized external API.
4. The result is a proposed event message that is published to a topic.
5. The Admin Application will consume these proposals and submit them for approval to an Admin. 
6. If an event is approved is send further via a topic to the Crew Dispatch system for fulfillment. This system is now responsible to dispatch maintenance crews to reposition vehicles and swap their batteries. And when the event ends the parking slots near the event venue area filled with ready vehicles waiting for customers eager to get home.


## Sequence Diagram

The next diagram follows an event from discovery to approval.

![Container Diagram](/Diagrams/C2-bulk-battery-swap-and-reposition-for-event/event-discovery-sequence-diagram.drawio.png)

## ADRs
1. [ADR-C2-01 When available get data from specialized services instead of LLMs.md](../../ADR/C2-bulk-battery-swap-and-reposition-for-event/ADR-C2-01%20When%20available%20get%20data%20from%20specialized%20services%20instead%20of%20LLMs.md)
2. [ADR-C2-02 Bulk repositioning is done with Admin approval.md](../../ADR/C2-bulk-battery-swap-and-reposition-for-event/ADR-C2-02%20Bulk%20repositioning%20is%20done%20with%20Admin%20approval.md)

## Non-Functional Requirements
 

## Related Architectural Components