# ADR-XXX: Feature Flagging for AI Components

## Context
AI and GenAI features may need quick enable/disable due to drift, outages, or new releases.

## Decision
Use **feature flagging service** (LaunchDarkly, Unleash, or open-source) for all risky/new AI features, enabling targeted rollout and controlled rollback.

## Alternatives
- **No feature flags:** Simpler, but much slower response to issues.

## Consequences
- **Agility:** Can test, rollback, and update user features safely.
- **Drawbacks:** Needs flag management and regular review.

## Status
Accepted
