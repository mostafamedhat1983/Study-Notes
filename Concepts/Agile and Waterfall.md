---
tags:
  - concepts
---
Agile is an iterative approach built around short delivery cycles, customer feedback, collaboration, and adapting to change as the work evolves.  
Waterfall is a linear model where teams finish one phase before moving to the next, such as requirements, design, implementation, testing, and maintenance.

## Why it matters

DevOps is not the same thing as Agile, but they reinforce each other because Agile speeds up development planning while DevOps improves how software is built, tested, released, and operated.  
In practice, Agile helps your team decide what to build next, while DevOps helps your team deliver it reliably through CI/CD, automation, monitoring, and shared ownership across development and operations.

## What to know

| Topic             | What you should understand                                                                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Agile mindset     | People and collaboration matter more than rigid process, working software matters more than heavy documentation, and teams should welcome changing requirements. |
| Waterfall mindset | Predictability, fixed scope, clear stages, and detailed documentation are central, which can help on stable or compliance-heavy projects.                        |
| Release style     | Agile favors short iterations and frequent delivery, while Waterfall usually leads to larger, less frequent releases.                                            |
| Feedback loops    | Agile and DevOps both depend on fast feedback; Waterfall is weaker here because feedback often arrives late in the cycle.                                        |
| Your DevOps role  | You should enable repeatable builds, automated tests, deployment pipelines, observability, rollback plans, and fast recovery, especially in Agile environments.  |

- The Agile Manifesto emphasizes four values: individuals and interactions, working software, customer collaboration, and responding to change.
- Agile differs from traditional waterfall development by promoting frequent communication, iterative progress, and flexibility to adapt to changes, reducing risk and improving project visibility.

## Agile principles 
- **Customer Satisfaction**: Prioritize satisfying the customer through early and continuous delivery of valuable software.
- **Welcoming Change**: Embrace changing requirements, even late in development, to gain a competitive advantage.
- **Frequent Delivery**: Deliver working software frequently, with a preference for shorter timescales.
- **Collaboration**: Ensure daily collaboration between business people and developers.
- **Empowerment**: Build projects around motivated individuals and trust them to get the job done.
- **Face-to-Face Communication**: Use face-to-face conversation as the most efficient method of conveying information.
- **Working Software**: Measure progress primarily through working software.
- **Sustainable Development**: Promote sustainable development practices for a constant pace of work.
- **Technical Excellence**: Focus on continuous attention to technical excellence and good design.
- **Simplicity**: Maximize the amount of work not done by focusing on simplicity.
- **Self-Organizing Teams**: Encourage self-organizing teams for the best architectures, requirements, and designs.
- **Continuous Improvement**: Reflect regularly on how to become more effective and adjust behavior accordingly.

---

Scrum and Kanban are two common ways teams apply Agile in real work. Scrum is more structured and uses fixed sprints, while Kanban is more flow-based and focuses on continuously moving work through a board.

## Scrum

Scrum is an Agile framework where teams work in short, time-boxed iterations called sprints, often 1 to 4 weeks long. It uses defined roles, shared backlogs, and regular events like sprint planning, daily standups, sprint review, and retrospective.

In Scrum, the team chooses a set of work for the sprint, tries to meet a sprint goal, and reviews results at the end before planning again. It is useful when you want rhythm, predictability, and regular checkpoints.

## Kanban

Kanban is an Agile method that visualizes work on a board, usually with columns like To Do, In Progress, and Done. Its main idea is to manage flow, limit how much work is being done at the same time, and improve delivery continuously rather than in fixed sprints.

Unlike Scrum, Kanban usually does not require sprint boundaries or specific roles like Scrum Master and Product Owner. Teams pull new work when they have capacity, which makes Kanban very useful for support, operations, platform, and ongoing DevOps tasks.

## Main difference

|Method|How work moves|Best fit|
|---|---|---|
|Scrum|Work is planned into fixed sprints.|Product development with clear iteration cycles.|
|Kanban|Work flows continuously based on team capacity.|Ops, support, maintenance, and interrupt-driven work.|

## For DevOps

As a DevOps engineer, you will often see Scrum in application teams and Kanban in infrastructure or operations teams. That is because DevOps work often includes unplanned tasks like incidents, access requests, environment fixes, pipeline issues, and production support, which fit Kanban well.

A simple way to remember it is this: Scrum is sprint-based, Kanban is flow-based. Many teams also mix them, using Scrum for planned feature work and Kanban for operational work