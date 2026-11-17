# Fetch — STAR stories for the hiring manager session

## Stuck on the thank-you screen (Walmart checkout)

Prompts: a failure and what changed after; debugging under ambiguity.

- **Situation**: Rare production reports of users frozen on the checkout thank-you screen. The only recourse was killing the app, on the highest-stakes flow at 5 to 6 million DAU. While the problem was infrequent, with Walmart's DAU count, it mattered. 
- **Task**: Find and fix a bug that did not reproduce and left little trace.
- **Action**: Narrowed it from reports and debug telemetry to a lifecycle fault. The parent view controller had been released, so the child checkout controller had nothing to dismiss it. 
- **Result**: Once we had more detailed data from the debug analytics about the path users followed that led to the problem, we were able to identify the combination of events that triggered it (a combination of how the cart was instantiated and cancelled, then reentered, then starting the checkout process) and fix the underlying issue.

## Accessibility across the iOS org (Walmart)

Prompts: stepped outside scope; mentoring or raising a team's standard.

- **Situation**: Senior management made accessibility a priority and set targets. The accessibility team was small and most iOS teams did not know how to proceed.
- **Task**: Own scope was checkout. Nobody owned getting 120 plus engineers to the targets.
- **Action**: Partnered with the accessibility team to develop implementation patterns, wrote training materials, and ran sessions across the iOS organization covering the reasoning and the iOS techniques.
- **Result**: Three separate sessions, over 120 engineers reached, multiple teams improved their accessibility score within the app over the next three months.

## Builder.io evaluation and training (Walmart platform team, 2025)

Prompts: mentoring or raising a standard; disagreed with a decision or committed.

- **Situation**: AI tooling working groups evaluating tools for the 120 plus iOS org. Most Xcode plugins were deficient for iOS work.
- **Task**: Assist the platform team in evaluating Builder.io, connecting the design library and Figma to generated iOS views.
- **Action**: Hands-on evaluation, evaluation criteria, integration recommendations, then live training for 80 plus engineers with recordings for the rest.
- **Result**: Engineering management mandated it for new view work and offered it for existing views. This was late in my time at Walmart, but teams were beginning to implement it within their respective codebases for new work.

## Swift 6.3 structured-concurrency migration (CLEAR)

Prompts: shipping under ambiguity; raising a team's standard.

- **Situation**: Production app with callback and Combine heavy patterns, six-engineer team shipping features at the same time.
- **Task**: Move to async/await, actors, Sendable, and strict checking without a feature freeze.
- **Action**: Phased plan, migrated incrementally module by module, new work in SwiftUI with Swift Testing.
- **Result**: By August 2026, most of the steps had been implemented by me or two other team members, and 80% of the modules were at the Swift 6.3 level of concurrency.
## Ground-up Walmart rewrite

Prompt: shipping under ambiguity.

- **Situation**: Small initial team replacing the flagship iOS and Android apps.
- **Task**: Choose architecture while shipping to fixed milestones.
- **Action**: Established the Swift and Combine patterns that became the foundation for the entire iOS platform as well as checkout module.
- **Result**: Both apps and much of the supporting systems replaced within two years. The modular structure, project setup, and responsibility division were instrumental in steadily increasing speed and quality of deployment.

## Binary size reduction and the component rearchitecture (Walmart)

Prompts: disagreed with a decision and committed; a cross-team dependency that did not come through; a failure and what changed after.

- **Situation**: The app binary was growing too large. Higher-level engineering management set aggressive, somewhat arbitrary reduction targets for each area and team.
- **Task**: Hit checkout's target. The low-hanging fruit would not get there; the rest required changing how components were built.
- **Action**: Assigned the low-hanging fruit to team members. Developed a plan and an initial proof of concept for rearchitecting component construction. Vetted it with the director first, then the team. Took it to the platform team, whose buy-in was required, over several weeks of back and forth.
- **Result**: The platform team liked the general idea but had backlog items that partly conflicted with or duplicated the approach, and declined to approve the plan. Abandoned it, said so, and moved on to the smaller areas the team could attack. Through the final year there, the platform team's own version was never implemented. It is unclear whether the rearchitecture would have significantly helped with the binary size since it was never developed far enough to assess.
## Operating at principal scope under team load (Walmart)

Prompts: stepped outside scope; a failure and what changed after; shipping under ambiguity about what the role should be.

- **Situation**: The principal role was meant to operate broader than the checkout team. Team workload left little room, and finding projects that fit what the director thought worth pursuing was itself a challenge.
- **Task**: Do enough out-of-band work to justify the role without the checkout commitments slipping.
- **Action**: Delivered several cross-org pieces (the accessibility patterns and training in story 2, the Builder.io evaluation and training in story 3, the engineering standards and interview bar-raiser work). Pursued two or three further projects that did not pan out: one was given to an overseas team, one the director pulled me off of for more pressing problems in our area, one I had started when the project was cancelled.
- **Result**: The work that landed was enough to justify the principal role. My manager was satisfied that the combination or projects with the iOS eco-system beyond my team were sufficient to be a successful principal level engineer