---
tags:
  - concepts
---
![[Pasted image 20260311054540.png]]
![[Pasted image 20260311054706.png]]

# DevOps Lifecycle

## Definition
The DevOps lifecycle is a continuous software delivery loop that helps teams plan, build, test, release, deploy, operate, and monitor applications with automation and fast feedback. [web:2][web:9]

## Main idea
DevOps is not a one-time sequence. It is an iterative cycle where feedback from production goes back into planning and development so teams can deliver software faster and more reliably. [web:3][web:5][web:7]

## Stages of the DevOps lifecycle

1. **Plan**  
   Define business requirements, goals, backlog items, priorities, and release scope. [web:5][web:9]

2. **Code**  
   Developers write application code and manage changes using version control systems such as Git. [web:5][web:7]

3. **Build**  
   The source code is compiled or packaged into a deployable artifact such as a binary, package, or container image. [web:5][web:9]

4. **Test**  
   Automated and manual tests validate functionality, quality, and stability before release. [web:5][web:7]

5. **Release**  
   The validated build is prepared and approved for deployment. [web:5][web:9]

6. **Deploy**  
   The application is deployed to staging or production using automation tools and CI/CD pipelines. [web:2][web:5]

7. **Operate**  
   Teams keep the application running in production by handling availability, runtime configuration, updates, and support. [web:2][web:9]

8. **Monitor**  
   Logs, metrics, traces, incidents, and user behavior are collected to detect issues and improvement opportunities. [web:2][web:5]

9. **Feedback**  
   Monitoring results and user feedback are sent back to the team to guide the next planning and development cycle. [web:3][web:9]

## 7C view of DevOps lifecycle
Another common way to explain DevOps is with the 7 Cs:
- Continuous Development [web:2][web:7]
- Continuous Integration [web:2][web:6]
- Continuous Testing [web:6][web:7]
- Continuous Delivery/Deployment [web:4][web:6]
- Continuous Monitoring [web:2][web:7]
- Continuous Feedback [web:2][web:9]
- Continuous Operations [web:2][web:6]

## Why it matters
- Faster software delivery through automation. [web:2][web:9]
- Better collaboration between development and operations teams. [web:9][web:7]
- Lower deployment risk through small, frequent changes and testing. [web:3][web:7]
- Continuous improvement based on real production feedback. [web:2][web:5]

## Real-world example
A developer pushes code to Git, the CI pipeline builds and tests it, the CD pipeline deploys it to Kubernetes, monitoring tools collect logs and metrics, and the team uses that feedback for the next release. [web:2][web:5][web:7]
