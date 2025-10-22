# ADR-001: Vehicle Health Record (VHR) Update

**Status:** Accepted
**Date:** 2025-10-20  
**Deciders:** Team Katalysis  
**Related Issue/Story:** <link or ID> (optional)

## Context and Problem Statement  
Improved prediction-based decision making at the system management component needs a real-time global 
state of all vehicles, including their ID, location, battery status, tire pressure and other critical 
sensor data to facilitate optimized service crew schedule and maintain real-time availability status.
This concerns the information content and frequency of the vehicle health record update.

## Decision   
Vehicle Health Record should include a combination of raw data as well as local (in-vehicle) AI model
generated data (probabilistic, e.g., hardware specific failure probability). The Update frequency 
should be dynamically adjusted based on timer, criticality, failure events, etc., so as to not incur
unnecessary communication cost to the cloud databases and pub-sub infrastructure. The in-vehicle AI 
subsystem need to be fine-tuned either by RAG or parameter fine-tuning based on the vehicle-specific
operation and service manuals. 

## Alternatives  
- Update raw data only: Lightweight process in vehicle IoT device, but puts the entire AI-based 
	decision making responsibility onto the AI system in System Management unit, thereby increasing
	communication complexity as well as coupling. Frequency is determined by the highest frequency
	of all sensor data update trigger.
- Update failure prediction only: More capable AI subsystem needed in the IoT devices which would 
	eliminate any possibility of in-vehicle model serving, cause higher frequency of remote AI API 
	calls (because every vehicle unit makes them), and yet have not way to make decision based on 
	combined dataset of records accumulated from all vehicles.
- Prepare hybrid data for VHR update: Use a mixture of raw and predicted data fields, based on 
	information and uncertainty of sensor data signals as well as vehicle operation and service manual 
	recommendation. This is the recomended choice.

## Decision Consequences  
- Efficient: Balances operational cost (communication, API calls) based on domain knowledge and data 
	communication efficiency.
- Trade-off: Slight increase in efforts while implementing AI/ML subsystem in local agent.

## Links / References  
- ADRs this one depends on or supersedes  
- Supporting documents, diagrams, prototypes  
- External links (articles, patterns, etc.)

