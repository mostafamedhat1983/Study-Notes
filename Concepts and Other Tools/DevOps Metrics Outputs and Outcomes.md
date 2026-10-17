---
tags:
  - concepts
---

## Outputs vs outcomes

- **Output**: Work delivered by a team, such as a deployed feature, fixed bug, or completed ticket.
- **Outcome**: The value or result created by that work, such as increased revenue, better customer retention, lower cost, or improved reliability.

DevOps focuses on outcomes, not only how much work a team delivers.

## DORA metrics

- **Deployment frequency**: How often code is deployed to production.
- **Lead time for changes**: Time from code commit to production deployment.
- **Change failure rate**: Percentage of deployments that cause a production failure or require remediation.
- **Time to restore service**: Average time to recover service after a production incident.

## Reading DORA metrics together

Use the four metrics together rather than treating one number as success. High deployment frequency is valuable only when change failure rate stays controlled and service restoration is fast.

Good delivery performance is not a trade-off between speed and stability. Teams improve both by building quality into the delivery process instead of relying on long manual test cycles or release gates.

## Capabilities behind the metrics

- **Version control**: Keep application code, infrastructure, and configuration versioned so changes are traceable and repeatable.
- **Automated testing**: Detect defects early and make frequent changes safer.
- **Continuous delivery**: Keep software in a deployable state and make releases routine.
- **Monitoring and observability**: Detect service problems quickly and support faster restoration.

These capabilities help reduce lead time, lower change risk, and improve recovery when failures occur.

## Other useful metrics

- **Deployment pain**: The effort, stress, time, and number of people needed for a deployment.
- **Unplanned work**: Work caused by incidents, support requests, defects, or unexpected changes that interrupt planned work.
- **Employee Net Promoter Score (eNPS)**: Measures how likely employees are to recommend their team or organization as a place to work.

## Tracking unplanned work

Track unplanned work separately from planned features, technical debt, and operational work. This makes capacity loss visible and helps teams improve estimates, reserve capacity, and reduce recurring sources of disruption.

## eNPS calculation

Employees score how likely they are to recommend their team or organization from 0 to 10.

- **Promoters**: Scores of 9 or 10.
- **Passives**: Scores of 7 or 8.
- **Detractors**: Scores from 0 to 6.

**eNPS = Percentage of promoters - Percentage of detractors**

Use the score as a starting point for learning about team health, workload, ownership, and improvement opportunities.

## Key idea

Measure delivery speed and reliability together. Faster delivery is useful only when changes remain stable and services can recover quickly.
