---
tags:
  - concepts
---

- **DevOps Monitoring and Observability**: Using actionable signals and rich system data to understand product health, detect issues early, and investigate unexpected behavior.

## Monitoring

Traditional monitoring often checks known conditions such as server availability, CPU usage, latency, or error thresholds.

Monitoring is useful when alerts are actionable. An alert should tell a team that someone needs to investigate or take action.

## Problems with reactive monitoring

- Dashboards are used only after an incident starts.
- Alerts are noisy or false alarms.
- Teams ignore alerts because they do not require action.
- Infrastructure may be healthy while the user experience is broken.

## Proactive monitoring

Monitor signals that reflect customer and business impact, not only infrastructure health.

Examples:

- Transactions per minute
- Failed payments or failed requests
- Error rate
- API response time
- Application crash rate
- Mean Time to Detect (MTTD)
- Mean Time to Restore (MTTR)

## Observability

Observability is the ability to understand what is happening inside a system by using the data it produces.

It helps engineers investigate unknown problems in complex distributed systems. Instead of predicting every failure, collect enough context to ask new questions during an incident.

## Good alerting practices

- Alert on symptoms that affect users or business outcomes.
- Reduce alerts that do not need human action.
- Automate remediation for known, repeatable failures.
- Use gradual rollouts, canaries, feature flags, and fast rollback to reduce risk.
- Sample high-volume telemetry while keeping useful context.

## Key idea

The goal is not to monitor everything. It is to detect meaningful problems early and give engineers the context needed to diagnose them.
