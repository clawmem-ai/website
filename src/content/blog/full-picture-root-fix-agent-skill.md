---
title: "Full-Picture Root-Fix: An Agent Skill for Bugs That Don't Stay Fixed"
description: "Turn a vague engineering symptom into an owner-level repair, then prove the real workflow recovered."
date: 2026-08-01T12:00:00
author: "Hazel"
authorPhoto: "/hazel.jpg"
---

> Turn a vague symptom into an owner-level fix, and prove the real workflow actually recovered.

An agent can make a checkout error disappear in five minutes and still leave the system broken. It can bump a timeout while the upstream dependency stays unhealthy. It can swallow a duplicate result while the operation still fires twice under the hood. It can ship a new code path and just... leave the old one reachable. Every one of these looks fine in a diff. None of them is actually a fix.

That's the gap **[Full-Picture Root-Fix](https://console.clawmem.ai/ClawMem/full-picture-root-fix/wiki)** is built to close. It is an Agent Skill for engineering work where a change that looks right is not enough. The work is complete only when the cause is understood, the system is corrected at the right layer, and the real workflow has been verified. That includes production incidents, cross-service failures, migrations, refactors, and architecture cleanup.

This isn't about how long the task takes. It's about work that needs several rounds of real evidence before anyone can honestly say it's done.

## The missing unit of work is the loop, not the prompt

Most interactions with a coding agent look like this:

```
describe a symptom → get a patch → ask for a test → ask what's next
```

Fine for a one-line change. It falls apart the moment a task needs actual investigation, a mental model of a stateful system, a real repair, and verification, spread across more than one turn. The agent loses the original goal along the way, starts optimizing whatever symptom is closest, or just stops the moment one narrow check turns green.

There's been a good amount of talk lately about **loop engineering**, basically pulling that control logic out of a chain of hand-typed prompts and putting it into something repeatable. A loop has a goal, state that survives across runs, tools and permissions, evidence that decides the next move, and an actual stopping condition. It's not "let the model run forever." It's "keep working until you can prove a defined condition, or escalate." [Addy Osmani's piece on loop engineering](https://addyosmani.com/blog/loop-engineering/) covers this shift well: from prompting turn-by-turn to designing the system that discovers, assigns, checks, and remembers work. His follow-up is the part I keep coming back to: the agent does the inner work, but an engineer still owns the outer decision boundary. [The outer-loop post](https://addyosmani.com/blog/own-the-outer-loop/) is where verification and accountability actually live.

Full-Picture Root-Fix is the engineering discipline that sits *inside* that loop:

```
goal → system model → evidence → diagnosis → correction → verification
```

![A closed engineering loop: goal, model, probe, repair, and verification return to the outcome.](/blog/full-picture-root-fix-agent-skill/01-closed-engineering-loop.jpeg)

<p class="image-caption">Full-Picture Root-Fix supplies the engineering method inside the persistent loop: each piece of evidence determines the next move toward the same outcome.</p>

| Layer | Job |
| --- | --- |
| The loop | Holds the goal and state, provides the environment, decides whether to continue, stop, or escalate. |
| Full-Picture Root-Fix | Decides *how* to investigate and fix: build the whole picture, close the evidence gap that actually matters, repair the owner, rip out the stale paths, verify against reality. |
| Human owner | Sets product intent, grants authority, makes the call to ship based on the evidence. |

A loop without a decent engineering method just repeats the wrong theory faster. A skill without a loop is useful for one turn and then runs out of runway. The loop gives you persistence; Full-Picture Root-Fix gives every turn actual judgment. You need both.

## What's actually in the skill

An agent can discover and load a versioned skill (instructions, references, optional tools) the moment a task matches. The point is reusable workflow knowledge that lives in one reviewable place instead of getting retyped into every chat. [Anthropic's writeup](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) frames this as packaging domain expertise into composable resources; [Vercel's guide](https://vercel.com/blog/agent-skills-explained-an-faq) describes skills as complete workflows: context, decision logic, success criteria, all bundled.

Full-Picture Root-Fix packages one specific workflow:

1. **Make the outcome testable.** Turn the request into a user-visible goal with real success criteria. Not "fix the bug," but something you can actually check.
2. **Build the picture.** Map the call path, data and state flow, who owns what, how it actually behaves at runtime, existing tests, user-visible behavior.
3. **Pick evidence that changes the answer.** Separate what you've actually observed from what you're assuming. Use the smallest probe that can rule causes in or out.
4. **Fix the owning layer.** Repair where the failure is *created*, not just where it happens to become visible.
5. **Delete the old shape.** Stale config, workaround logic, duplicate paths, anything that would leave the failure class half alive.
6. **Verify against the real world.** Re-run the original path plus the nearby blast radius before calling it done.

The sequencing is the whole point. It stops an agent from treating investigation, cleanup, and verification as optional extras tacked on after a patch already looks convincing, which is exactly when a task looks done but isn't.

This sequencing also makes an ambiguous request actionable. The user can state a symptom and a safety boundary; the skill turns them into a goal, success criteria, candidate causes, evidence gaps, and a next probe that can separate those causes. A product trade-off or an irreversible decision still needs a human owner. The user does not need to write the investigation plan before the agent can start.

## Results from two reproducible incident evaluations

We evaluated Full-Picture Root-Fix in two incident environments that could be independently checked. In the Cloud-OpsBench checkout evaluation, three plain no-skill diagnostic attempts reached no exact root cause; the Full-Picture Root-Fix repair loop identified the stale owner resource and completed the recovery checks. In the formal SRE-World A/B, the skill completed **2 of 6** incident closures (**33.3%**) compared with **0 of 6** without it: a **+33.3 percentage-point** difference.

### 1. Cloud-OpsBench: a checkout incident with a hidden cause

The first evaluation is a reproducible checkout incident constructed from the Cloud-OpsBench `performance/17` failure shape. Checkout degrades, several downstream services look unhealthy, and the loudest component is not the source of the failure.

The agent started with this short incident brief:

```
Use Full-Picture Root-Fix. Checkout intermittently fails.
Find and fix the cause without breaking normal shopping.
```

The brief states the outcome and the safety boundary, not a troubleshooting plan. The rest of the evaluation shows how the skill turns that small amount of context into a completed repair.

The real fault was a stale `frontend-egress-delay` NetworkChaos resource, quietly injecting 780ms of egress delay at the frontend boundary. That delay propagated into upstream deadline pressure and eventually surfaced as a checkout 502.

```
stale frontend NetworkChaos
  → frontend egress wait
  → upstream deadline pressure
  → checkout 502
```

![Cloud-OpsBench causal map: a stale NetworkChaos resource caused frontend egress delay, deadline pressure, and checkout 502; the repair deleted the stale fault-injection resource and verified adjacent shopping paths.](/blog/full-picture-root-fix-agent-skill/02-cloud-opsbench-root-cause.jpeg)

<p class="image-caption">The evidence chain made the stale NetworkChaos resource—not the loudest downstream service—the layer to repair.</p>

This is exactly the kind of task where a local fix is tempting—and wrong.

We ran three plain diagnostic attempts on the same incident, without the skill, first. Zero of three found the actual root cause. The symptoms were loud enough to trigger an investigation but not loud enough to converge on the stale NetworkChaos resource sitting behind them. Once Full-Picture Root-Fix ran as a persistent repair loop, the agent turned that vague checkout symptom into a real goal, a set of candidate causes, the evidence gaps between them, and a next probe designed to actually discriminate between theories, then closed the repair.

| Run | What we saw | End state |
| --- | --- | --- |
| **No skill (three plain diagnostic attempts)** | **0/3** reached the exact root cause. | No owner-level repair. |
| **Full-Picture Root-Fix repair loop** | Reconstructed `NetworkChaos → egress wait → deadline pressure → checkout 502`. | Removed the stale resource; checkout, catalog, sponsored ads, and config-preservation checks all passed. |

The resulting loop was small and closed:

1. Define the outcome: recover checkout without breaking catalog or sponsored ads.
2. Write down the competing explanations and the evidence gap between them.
3. Use the most discriminating evidence available to pin frontend egress delay as the cause, not just a downstream symptom of it.
4. Delete the stale NetworkChaos resource instead of tuning around it.
5. Independently verify checkout recovery, catalog behavior, sponsored ads, unrelated feature flags staying untouched, and the old resource actually being gone.

The result was more than a healthier-looking service. The incident had an explanation, the source of delay was removed, and the customer-facing path plus surrounding behavior were verified after the change. Just as importantly, the agent began with only an ambiguous symptom and a safety boundary. Full-Picture Root-Fix turned that brief into a concrete goal, candidate causes, evidence gaps, an owner-level repair, and a verification plan. The user did not need to supply the investigation plan.

### 2. SRE-World: a live Helm/Kubernetes incident A/B

The second evaluation ran on a live Helm/Kubernetes incident environment. The task was to repair intermittently slow message writes while keeping normal traffic and routine maintenance running, survive a post-repair soak check, and file one accurate incident report. Both conditions ran on `gpt-5.6-luna` at high reasoning effort, with identical task snapshots, infrastructure, timeouts, and an independent verifier. The treatment group read the Full-Picture Root-Fix skill first, derived a task-specific goal and success criteria, then kept running the evidence-driven loop. The baseline got neither the skill nor that structure.

| Formal SRE-World result | Full-Picture Root-Fix | No skill |
| --- | ---: | ---: |
| Primary reward across 12 trials | **2/6 (33.3%)** | **0/6 (0%)** |
| `06-F4-maintenance-collision` | **2/3 (66.7%)** | **0/3 (0%)** |
| Minimality / database-state checks | 5/6 · 6/6 | 5/6 · 6/6 |

![SRE-World A/B result: Full-Picture Root-Fix closed 2 of 6 incidents, compared with 0 of 6 without the skill, with the maintenance-collision case closing 2 of 3 skill-guided runs.](/blog/full-picture-root-fix-agent-skill/03-sre-world-ab-results.jpeg)

<p class="image-caption">The result visualizes full incident closures under the same environment and verifier; it does not count partial patches as successes.</p>

The headline number is simple: **2/6** full incident closures with the skill against **0/6** without it, a **+33.3 percentage-point** gain overall. Most of that gap comes from `06-F4-maintenance-collision`, where Full-Picture Root-Fix closed **2 of 3** runs (**66.7%**) against **0 of 3** without it: a **+66.7 percentage-point** swing on that single incident.

The two successful skill-guided runs correctly identified the real owner (`db.maintenance-controller`), kept the recurring maintenance job enabled while shifting it out of the write peak, restored healthy message writes, and passed the verifier's service, goodput, latency, schedule, safety, and report-attribution checks. The equal minimality and database-state scores show that the result did not come from a broader or riskier change; it came from a more *complete* repair.

Together, the two evaluations show the same closed loop: start from a user-visible outcome, close the evidence gap between plausible causes, fix the actual owner, preserve the surrounding system, and prove the repair under independent checks.

## What we saw across the broader evaluation

We ran Full-Picture Root-Fix across source-code debugging, performance, browser behavior, long-horizon feature work, incident diagnosis, and repair. The most valuable results concentrate in a few engineering behaviors: turning a vague symptom into a verifiable task, following the causal mechanism, repairing at the owner layer, and removing the old path.

| Signal | What we observed | What it means |
| --- | --- | --- |
| **From a vague symptom to a complete repair** | In the Cloud-Ops checkout incident, three plain no-skill diagnostic attempts did not reach the exact root cause; Full-Picture Root-Fix built the causal chain, removed the stale NetworkChaos resource, and passed checkout, catalog, sponsored-ad, and configuration-preservation checks. | The skill can carry “checkout is broken” through diagnosis, an owner-level change, cleanup, and proof after the change. |
| **Better mechanism direction** | In ITBench Scenario-26, the skill-guided run entered the Chaos Mesh fault-injection domain, while the comparison run selected an unrelated ad deployment. | Building the full picture directs investigation toward the causal mechanism rather than the loudest co-occurring component. |
| **Reusable root-fix discipline** | In a controlled checkout feature-flag incident and a directly visible Kubernetes root cause, skill-guided runs performed the owner-level repair and independent checks. | When the root cause is already visible, the skill preserves the same goal, cleanup, and verification standard. |
| **Better long-horizon engineering hygiene** | In long-horizon implementation work, the skill condition had a higher final partial-test pass rate, fewer regression failures, and less duplicate code. | A closed-loop method keeps attention on regression, cleanup, and maintainability—not only the next patch. |

These results describe the conditions where the skill creates the most value: several plausible explanations, evidence distributed across layers, a tempting workaround, and a real verifier for the repaired workflow. That is where a stronger engineering method is more valuable than another local patch.

## Where it earns its keep, and where it's overkill

Full-Picture Root-Fix is deliberately opinionated. The extra investigation and verification pay for themselves when a request is easy to state but hard to prove finished: several causes look plausible, the evidence is scattered, a local workaround is right there and tempting, and the fix has to leave everything else standing. Both cases above are that shape.

| Good fit | Why it helps |
| --- | --- |
| **A vague symptom, a clear desired outcome** | "Checkout fails, don't break shopping" is enough. The skill turns that into a testable goal, constraints, evidence gaps, and a first plan, no carefully engineered prompt required. |
| **A long-running task or incident with several plausible causes** | Whether it is a production incident or a multi-turn engineering task, the skill keeps the investigation anchored to one goal and uses evidence to separate propagation from root cause. |
| **Cross-service reliability, performance, consistency, or security work** | The visible failure is often far from the actual owner. A full system picture stops you from treating the symptom instead of the cause. |
| **Migrations, refactors, cleanup, architecture fixes** | It maps old and intended flows, makes ownership explicit, updates affected paths together, deletes dead code and configuration, and checks that nothing is left half-migrated. |
| **Browser, simulator, device, or UI work** | It treats the user-visible path as something to verify, not something to eyeball once and move on. |

| Poor fit | Why not |
| --- | --- |
| **A one-line, well-specified change** | The system picture and evidence loop add process without adding value. |
| **A direct lookup or a single deterministic operation** | A tool call is cheaper and clearer. |
| **Routine implementation with clear requirements and obvious tests** | The task is already bounded; a full root-fix investigation is unnecessary overhead. |
| **A task that actually needs a product or business decision nobody's made yet** | The skill can surface the missing decision. It shouldn't invent authority it doesn't have. |

The rule of thumb: reach for it when the request is easy to say and hard to prove done.

## Get it through ClawMem; pin a revision if you need to

Full-Picture Root-Fix is published and maintained by ClawMem in the wiki of its public `clawmem-skill` knowledge repo: [open the public skill page](https://console.clawmem.ai/ClawMem/full-picture-root-fix/wiki). Because the repo is public, it's a shared source for any agent or team connected to ClawMem, not something each agent has to copy and babysit separately.

If your agent already has ClawMem and access to that public repo, there's nothing else to install, no local copy, no manual sync. When a matching task shows up (root-cause work, long-running engineering, no-workaround cleanup, verification, logs, performance, edge cases) it just reads the current skill straight from the public wiki. When ClawMem ships an update, connected agents pick it up the next time a matching task calls for it.

If you need something reviewed, offline, or deliberately frozen for reproducibility, pull `SKILL.md` from the same wiki and pin that exact revision locally:

```
.agents/skills/full-picture-root-fix/SKILL.md
```

The public wiki keeps connected agents on the latest published version; a local pin freezes one reviewed version on purpose. Either way, the skill doesn't replace authority, observability, tests, or review. It's the repeatable method that tells an agent how to actually use those things to close an engineering loop.

## The takeaway

Long-horizon agent work fails when every individual turn looks reasonable but the task as a whole never becomes true. Full-Picture Root-Fix gives that work a standard to hold itself to:

```
state the outcome → model the system → close the evidence gap
→ fix the owner → remove the stale path → prove recovery
```

Use it when the job isn't just to produce a patch. It's to leave the system demonstrably better than you found it.
