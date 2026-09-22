---
tags:
  - concepts
---
Continuous integration, continuous delivery, and continuous deployment are three related but different levels of software delivery automation: **CI** checks that code changes integrate safely, **continuous delivery** keeps every validated change release-ready, and **continuous deployment** automatically releases every change that passes the required automated pipeline checks to production.

## The difference

- **Continuous integration**: Developers merge changes frequently into a shared branch, and each change is validated with automated builds and tests.
- **Continuous delivery**: This builds on CI by automating the release pipeline so the application is always in a deployable state. A manual decision or trigger may still be used for production release.
- **Continuous deployment**: This goes one step further by automatically deploying every change that passes the required automated pipeline checks directly to production, with no manual approval step.

## Easy comparison

| Practice | Main focus | Production release |
|---|---|---|
| Continuous integration | Merge code often and run automated build and test checks. | Not the goal by itself. |
| Continuous delivery | Keep every validated change release-ready through an automated pipeline. | Manual decision or trigger may be used. |
| Continuous deployment | Automatically release every change that passes the required checks. | Fully automatic. |

## Simple flow

- **CI**: Did the new code integrate safely?
- **Continuous delivery**: Is this build ready to release now?
- **Continuous deployment**: Did it pass every required check and go live automatically?

## Important nuance

Continuous deployment is an extension of continuous delivery. It usually requires strong automated testing, monitoring, rollback, and production safety controls.

Automated deployment to test or staging environments is common in continuous delivery, but it is not required by its definition.

## Practical example

If a pipeline builds, tests, scans, and deploys to staging automatically, then waits for a production trigger, it supports continuous delivery. If the pipeline deploys to production automatically after all required checks pass, it is continuous deployment.

## Interview answer

“CI means frequent code merges with automated build and test validation. Continuous delivery means every validated change is release-ready, but production release can still be manually triggered. Continuous deployment means every change that passes the required automated checks is deployed to production automatically.”
