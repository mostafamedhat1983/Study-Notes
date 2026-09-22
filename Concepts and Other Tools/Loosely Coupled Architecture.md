---
tags:
  - concepts
---

- **Loosely Coupled Architecture**: An architecture where components are isolated enough that they can be changed, tested, deployed, and scaled independently.

## Tight coupling

Tightly coupled components depend heavily on each other. A change in one component can require changes, testing, or deployment in several other components.

This creates large releases, complex test environments, slower delivery, and more coordination between teams.

## Loose coupling

Loosely coupled components communicate through clear interfaces, such as APIs, events, queues, or service contracts. Components still collaborate, but they do not need detailed knowledge of each other's internal implementation.

## Benefits

- Teams can test and deploy their services independently.
- Failures are more isolated and less likely to affect the whole system.
- Teams have more autonomy and fewer cross-team dependencies.
- Smaller components can scale independently.
- Changes can be delivered faster with lower risk.

## Important idea

Loose coupling does not mean teams stop collaborating. It reduces unnecessary implementation-level coordination so teams can focus their collaboration on shared goals and customer value.

## DevOps example

Instead of putting loyalty rules inside a point-of-sale application, expose loyalty as an independent service through an API. Multiple applications can then use the same service without changing the point-of-sale system.
