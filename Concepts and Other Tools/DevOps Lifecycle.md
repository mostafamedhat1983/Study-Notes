---
tags:
  - concepts
---
# DevOps Lifecycle

## Definition
The DevOps lifecycle is a continuous software delivery loop that helps teams plan, build, test, release, deploy, operate, and monitor applications with automation and fast feedback.

## Main idea
DevOps is not a one-time sequence. It is an iterative cycle where feedback from production goes back into planning and development so teams can deliver software faster and more reliably.

## Stages of the DevOps lifecycle

1. **Plan**  
   Define business requirements, goals, backlog items, priorities, and release scope.

2. **Code**  
   Developers write application code and manage changes using version control systems such as Git.

3. **Build**  
   The source code is compiled or packaged into a deployable artifact such as a binary, package, or container image.

4. **Test**  
   Automated and manual tests validate functionality, quality, and stability before release.

5. **Release**  
   The validated build is prepared and approved for deployment.

6. **Deploy**  
   The application is deployed to staging or production using automation tools and CI/CD pipelines.

7. **Operate**  
   Teams keep the application running in production by handling availability, runtime configuration, updates, and support.

8. **Monitor**  
   Logs, metrics, traces, incidents, and user behavior are collected to detect issues and improvement opportunities.

9. **Feedback**  
   Monitoring results and user feedback are sent back to the team to guide the next planning and development cycle.

## 7C view of DevOps lifecycle
Another common way to explain DevOps is with the 7 Cs:
- Continuous Development
- Continuous Integration
- Continuous Testing
- Continuous Delivery/Deployment
- Continuous Monitoring
- Continuous Feedback
- Continuous Operations

## Why it matters
- Faster software delivery through automation.
- Better collaboration between development and operations teams.
- Lower deployment risk through small, frequent changes and testing.
- Continuous improvement based on real production feedback.

## Real-world example
A developer pushes code to Git, the CI pipeline builds and tests it, the CD pipeline deploys it to Kubernetes, monitoring tools collect logs and metrics, and the team uses that feedback for the next release.
