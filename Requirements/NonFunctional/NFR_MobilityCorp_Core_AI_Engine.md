# Non-Functional Requirements for MobilityCorp Core AI Engine

## Top Driving Characteristics
- Availability: 99.99% server up-time, geo-redundancy
- Performance: <1s response to queries
- Extensibility: Plug-and-play fleet/routing features
- Fault Tolerance: Operational if non-core modules fail

## Others Considered
- Data Consistency: Real time sync for fleet/location data
- Recoverability: Fast restart after crash (<1min)
- Security: RBAC and strict API authentication
