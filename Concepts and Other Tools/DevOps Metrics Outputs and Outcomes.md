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
