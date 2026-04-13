---
title: "McAgile: The McDonaldization of Software Development Meets AI Coding Agents"
date: 2026-04-12T11:00:00+00:00
featured: true
tags:
  - codex-cli
  - enterprise
  - safe
  - agile
  - process
  - adoption
  - opinion
---

![Sketchnote diagram for: McAgile: The McDonaldization of Software Development Meets AI Coding Agents](/sketchnotes/articles/2026-04-12-mcagile-mcdonaldization-safe-ai-coding-agents.png)

In 1993, the sociologist George Ritzer published *The McDonaldization of Society*, arguing that the principles making McDonald's successful — efficiency, calculability, predictability, and control — were colonising every sector of modern life[^1]. Three decades later, his framework explains something that practitioners have felt but struggled to articulate: what happened when the Agile Manifesto met the enterprise consulting industry and became SAFe.

This is a companion piece to "[SAFe Was Bad for Agility. For AI Coding Agents, It's Worse.](2026-04-12-safe-enterprise-processes-ai-coding-agents.md)" That article documented the specific ways SAFe breaks AI coding workflows. This one asks a deeper question: *why* does SAFe break them? The answer is that SAFe is not agile at all. It is McDonaldized software development — **McAgile** — and AI coding agents are the force that finally makes the irrationality undeniable.

## The Four Pillars of McAgile

Ritzer identified four dimensions through which McDonaldization operates. Each maps precisely to SAFe.

### 1. Efficiency: The Optimum Method That Optimises the Wrong Thing

McDonaldization defines efficiency as the optimum method for getting from point A to point B. McDonald's achieves this through drive-throughs, pre-assembled ingredients, and transferring work to the customer (you carry your own tray, clear your own table)[^1].

SAFe's efficiency takes the same form. PI Planning is presented as the optimum method for aligning 50-125 people across an Agile Release Train. Standardised ceremonies — daily standups, iteration reviews, system demos, inspect-and-adapt workshops — provide the prescribed efficient path from idea to delivery. Teams are expected to be "self-organising," which, like McDonald's self-service counters, transfers coordination work from management to the people doing the actual work[^2].

The problem is identical to McDonald's: the system optimises for *throughput of the process*, not *quality of the outcome*. A McDonald's drive-through is optimised for cars-per-hour, not nutrition. SAFe's PI Planning is optimised for work-items-per-increment, not customer value. US Bank discovered this the hard way: teams followed SAFe perfectly but delivered no customer value, eventually abandoning the framework for Kanban[^3].

Now add AI coding agents. Codex CLI can generate a working feature in 20 minutes. SAFe's "efficient" process requires that feature to be decomposed into stories, estimated in points, assigned to a sprint, reviewed in a ceremony, and demonstrated in a system demo. The process designed for efficiency adds weeks of overhead to work that takes minutes. The drive-through queue is longer than walking to the restaurant.

### 2. Calculability: When Story Points Replace Thinking

Ritzer's calculability principle holds that in McDonaldized systems, quantity becomes a surrogate for quality. A "Quarter Pounder" tells you the weight; it tells you nothing about whether it is good. The "Big Mac" is big — that is the selling point[^1].

SAFe's calculability is story points, velocity, flow metrics (Flow Velocity, Flow Time, Flow Load, Flow Efficiency, Flow Distribution, Flow Predictability), and the Program Predictability Measure targeting 80-100%[^4]. The entire measurement apparatus privileges what can be counted over what matters. A team with high velocity is "performing well" regardless of whether their output solves customer problems. A PI with 90% predictability is "successful" even if the features delivered were the wrong ones.

This is what Ritzer calls the "Big Mac fallacy" — the assumption that more is better[^1]. SAFe's metrics create exactly this distortion. Faros AI's enterprise data shows teams with high AI adoption merge 98% more pull requests[^5]. By SAFe's calculability logic, this is a triumph. But PR review time increases 91%, and Google's DORA research found that 25% increased AI adoption correlates with 7.2% *decreased* delivery stability[^6]. More PRs. Less working software. The Quarter Pounder got bigger but less nutritious.

The deeper problem: AI coding agents make calculability metrics explode. Codex CLI can generate 50 PRs in a day. Velocity goes through the roof. Every SAFe dashboard turns green. But as Addy Osmani documents, developers are merging AI-generated code they cannot explain three days later[^7]. The numbers look better than ever while comprehension debt accumulates silently underneath. McDonaldization's quantity-over-quality trap, amplified by artificial intelligence.

### 3. Predictability: The Same Process Everywhere, Regardless of Context

A Big Mac in London tastes identical to one in Tokyo. That is the point. Predictability means no surprises — and no adaptation to local conditions[^1].

SAFe achieves predictability through identical ceremonies, roles, and artifacts across all Agile Release Trains. The Implementation Roadmap prescribes a uniform adoption path regardless of organisational context[^2]. Every team gets the same role taxonomy (RTE, Product Owner, Scrum Master, System Architect), the same cadence, the same rituals. This is why SAFe appeals to enterprise leadership: it promises that a team in the London office will operate identically to a team in Singapore. The franchise model.

Gabriel Steinhardt identified this pattern explicitly: Scrum's insistence that all team members are simply "Developers" creates "a standard worker who can be tasked with any development chore, moved from one development team to another, and easily replaced"[^8]. This is the McDonaldized workforce — interchangeable, standardised, and deskilled. SAFe scales this to the entire organisation through its certification programme: over 2 million SAFe-certified practitioners worldwide, each trained to implement the same framework the same way[^2].

AI coding agents demolish the case for this predictability. The toolchain evolves faster than any standardised process can accommodate. Codex CLI shipped version 0.119 on April 10 and 0.120 on April 11. Features that did not exist on Monday are in production by Wednesday. A framework that guarantees "the same process everywhere" guarantees that every team is equally unable to adapt to new capabilities. The Big Mac recipe does not change when better ingredients become available — and neither does the SAFe PI when better tools arrive mid-increment.

### 4. Control: Replacing Human Judgment with Non-Human Technology

The most insidious of Ritzer's four pillars. McDonald's replaces human judgment with automated systems: timers that beep when fries are done, assembly-line layouts that dictate worker movement, scripts that standardise customer interaction. The worker becomes an appendage of the machine[^1].

SAFe's control mechanisms are subtler but structurally identical. PI Planning commits teams to work in advance, removing the judgment to reprioritise mid-flight. Jira and ALM tooling enforce process conformity — you cannot create a ticket that does not fit the taxonomy. The certification system ($1,295-$5,000+ per certification) creates compliance through credentialing: you are not qualified to do agile unless a SAFe-authorised instructor says so[^2]. The RTE role exists primarily as a control mechanism, ensuring trains run on the prescribed schedule regardless of whether the destination is correct.

Here is where AI coding agents create an irresolvable contradiction. The entire point of tools like Codex CLI is to *amplify* human judgment — to let a developer express intent ("refactor this module for testability") and have the agent handle implementation. SAFe's control structures do the opposite: they *constrain* human judgment within predetermined channels. An agent running `full-auto` with sandbox guardrails is exercising more autonomous judgment than SAFe allows its human developers.

The teams getting real value from AI coding agents are the ones who trust the agent within defined boundaries — `auto-edit` for safe refactors, `suggest` for infrastructure changes, trust encoded in `codex.toml` rather than meeting invitations. SAFe organisations cannot adopt this model because it bypasses the control structures that justify the framework's existence. You cannot run `full-auto` when the process requires human sign-off at every level. The agent must be degraded to "expensive autocomplete" — the AI equivalent of a McDonald's worker who is not permitted to deviate from the script.

## The Irrationality of Rationality: SAFe's Fifth Dimension

Ritzer's most powerful insight is the "fifth dimension" — the irrationality of rationality. Systems designed for maximum rationality inevitably produce irrational outcomes. McDonald's pursuit of efficiency creates junk food. Its pursuit of predictability destroys local culinary tradition. Its pursuit of control deskills workers and strips dignity from labour. Ritzer calls this the "iron cage" — Weber's metaphor for inescapable rational structures that suppress human freedom[^1].

SAFe's iron cage is now visible to anyone who looks:

**The framework meant to enable agility destroys it.** Devoteam's analysis concludes that SAFe "prioritises predictability over agility" and contradicts the Agile Manifesto's "Responding to change over following a plan" through "rigid PI Planning every 8-12 weeks"[^9]. A philosophy born to free teams from bureaucracy has been, in their words, "weaponised into a new form of bureaucracy."

**The creators disown the product.** Ron Jeffries, co-author of the Agile Manifesto, warns developers to "abandon Agile" (capital-A). Dave Thomas, another signatory, says the word is "effectively meaningless"[^10]. When the people who invented the thing tell you the scaled version has nothing to do with what they intended, the irrationality is complete.

**The certification mill produces credentials, not capability.** Over 2 million people hold SAFe certifications. The AI4Agile Practitioners Report found that 83% of agile practitioners use AI tools but most spend 10% or less of their time with them[^11]. The certifications did not prepare them for the most significant shift in software development since version control. The franchise trained millions of people to flip burgers — and then the kitchen went fully automated.

**Jeff Gothelf's insurance company is Ritzer's case study made real.** Eight months, three PI planning cycles, a 21-slide deck explaining the AI strategy — zero AI-powered features shipped. Gothelf calls it "SAFe working exactly as designed"[^12]. That is the irrationality of rationality in a single sentence. The process worked perfectly. The outcome was nothing.

## Why AI Agents Make the Irrationality Undeniable

SAFe's McDonaldization worked — or at least appeared to work — when the gap between "process overhead" and "development speed" was narrow. If a feature takes two weeks to build, adding a week of ceremony is a 50% overhead. Costly, but tolerable. Organisations could tell themselves the predictability was worth it.

AI coding agents make the gap obscene. When Codex CLI can build the feature in 20 minutes, a week of ceremony is not a 50% overhead — it is a 50,000% overhead. The irrationality that was always present becomes mathematically undeniable.

This is the same dynamic Ritzer documented in other McDonaldized industries. Healthcare: standardised 15-minute appointments work until a patient needs 45 minutes. Education: standardised testing works until a student learns differently. Software: standardised PI planning works until an AI agent can iterate faster than the planning cycle.

The Gartner prediction that over 40% of agentic AI projects will be cancelled by end of 2027[^13] is not a technology forecast. It is a McDonaldization forecast. Those projects will fail not because the agents cannot do the work, but because the McAgile processes surrounding them cannot absorb the output.

## De-McDonaldizing Enterprise AI Adoption

Ritzer himself identified the antidote: he called them "islands of the living" — spaces that resist McDonaldization through craft, local adaptation, and human judgment[^1]. In software development, this translates to specific structural changes:

**Replace efficiency theatre with outcome measurement.** Stop measuring velocity and start measuring customer problems solved. When the agent can generate code in minutes, the only metric that matters is whether the code solved the right problem. This requires what Marty Cagan calls product teams — empowered groups that maximise problems-solved rather than features-delivered[^14].

**Replace calculability with comprehension.** Instead of counting PRs merged, measure whether the team understands the code in their system. Osmani's research shows 48% of developers do not consistently review AI code before committing[^7]. The McAgile response is to add more review ceremonies. The craft response is to invest in specifications that make agent output reviewable — 70% effort on problem definition, 30% on execution.

**Replace predictability with adaptability.** Weekly 30-minute capability reviews instead of quarterly PI planning. What has the toolchain shipped this week? What experiments should we run? Microsoft's research found it takes 11 weeks for developers to realise full productivity benefits from AI tools[^6]. By the time a PI cycle assesses a tool's value, the tool has already evolved past three generations.

**Replace control with trust boundaries.** Encode governance in `codex.toml` profiles and sandbox constraints, not in approval hierarchies and meeting schedules. The agent operates freely within defined boundaries — the craft kitchen, not the McDonald's assembly line.

**Invest in the Harness Architect role.** Agility at Scale identifies this emerging staff-level specialisation: the person who designs constraints that improve thousands of lines of agent output[^15]. SAFe has no equivalent role, no career path for it, and no ceremony that produces the artefacts it requires. McAgile cannot accommodate a craft role because craft is the antithesis of McDonaldization.

## The McAgile Reckoning

The Agile Manifesto was a craft movement. Seventeen practitioners gathered at a ski lodge to articulate values they had discovered through practice: individuals over processes, working software over documentation, customer collaboration over contracts, responding to change over following a plan[^10].

SAFe took those craft values and McDonaldized them. It created the franchise model: standardised processes, certified practitioners, predictable outcomes, centralised control. It replaced the ski lodge with a drive-through. And for a decade, enterprises bought the franchise because it promised agility without the discomfort of actually being agile.

AI coding agents are the reckoning. They do not care about your PI planning cycle. They do not respect your sprint boundaries. They do not need your story point estimates. They compress the feedback loop between idea and working code from weeks to minutes — and in doing so, they expose the 50,000% overhead that McAgile has been hiding behind the appearance of efficiency.

The enterprises that succeed with Codex CLI will be the ones that de-McDonaldize their development processes. They will replace the franchise with the craft kitchen — outcome-based, adaptive, trust-driven, and built for the speed at which AI agents actually operate.

The tools are ready. The question is whether enterprises are willing to close the drive-through.

---

## Citations

[^1]: George Ritzer, *The McDonaldization of Society* (Pine Forge Press, 1993; 10th edition 2023). Four pillars: efficiency, calculability, predictability, control. The "irrationality of rationality" as the fifth dimension. "Islands of the living" as resistance to McDonaldization.

[^2]: Scaled Agile Inc., [SAFe 6.0 Framework](https://scaledagile.com/). PI Planning, Implementation Roadmap, role taxonomy, certification programme. Over 2 million SAFe-certified practitioners globally.

[^3]: DEV Community, [Why SAFe Fails: A Critical Analysis](https://dev.to/tracywhodoesnot/why-safe-fails-a-critical-analysis-of-scaled-agile-frameworks-and-better-alternatives-o9k). US Bank case study: teams followed SAFe but delivered no customer value, abandoned for Kanban. SAFe rated "Low" for team autonomy.

[^4]: Agilemania, [SAFe Metrics for Agile Transformation](https://agilemania.com/safe%C2%AE-metrics-for-agile-transformation). Flow Velocity, Flow Time, Flow Load, Flow Efficiency, Flow Distribution, Flow Predictability. Program Predictability Measure targeting 80-100%.

[^5]: Faros AI, [Enterprise AI Coding Assistant Adoption: Scaling to Thousands](https://www.faros.ai/blog/enterprise-ai-coding-assistant-adoption-scaling-guide). 21% more tasks, 98% more PRs, 91% more review time. EdTech case study: 15,324% ROI.

[^6]: Java Code Geeks, [The AI Coding Assistant Has Been on Your Team for a Year](https://www.javacodegeeks.com/2026/04/the-ai-coding-assistant-has-been-on-your-team-for-a-year-what-did-it-actually-change-about-how-we-write-software.html), April 2026. DORA: 25% AI adoption → 7.2% decreased delivery stability ("Vacuum Hypothesis"). 11-week ramp-up. Fortune 50: 1,000→10,000 monthly security findings.

[^7]: Addy Osmani, [The 80% Problem in Agentic Coding](https://addyo.substack.com/p/the-80-problem-in-agentic-coding). Comprehension debt. 48% don't review AI code. 38% find AI code harder to review. 70/30 spec/execution split.

[^8]: Gabriel Steinhardt, [The McDonaldization of the Development Team](https://ebrary.net/129516/management/mcdonald_) (Blackblot). Direct application of Ritzer to Scrum: "a standard worker who can be tasked with any development chore, moved from one development team to another, and easily replaced." Proposes medieval guild model as alternative.

[^9]: Devoteam, [From the Agile Promise to Industrial Bureaucracy](https://www.devoteam.com/expert-view/agile-a-betrayed-philosophy/). SAFe "prioritises predictability over agility." Agile "weaponised into a new form of bureaucracy."

[^10]: Ron Jeffries, [Developers Should Abandon Agile](https://ronjeffries.com/articles/018-01ff/abandon-1/) (2018). Dave Thomas: "The word 'agile' has been subverted to the point where it is effectively meaningless." Agile Manifesto signatories disowning scaled implementations.

[^11]: DZone, [The AI4Agile Practitioners Report 2026](https://dzone.com/articles/ai4agile-practitioners-report). 83% of Agile practitioners use AI, most spend ≤10% of time with it. Integration uncertainty top barrier at 54.3%.

[^12]: Jeff Gothelf, [SAFe Was Bad for Agility. For AI, It's Catastrophic](https://jeffgothelf.com/blog/safe-was-bad-for-agility-for-ai-its-catastrophic/), April 2026. Insurance company: 8 months, 3 PI cycles, 21-slide deck, zero features. "SAFe working exactly as designed."

[^13]: Gartner, [Predicts 2026: AI Agents Will Reshape Infrastructure & Operations](https://www.itential.com/resource/analyst-report/gartner-predicts-2026-ai-agents-will-reshape-infrastructure-operations/). Over 40% of agentic AI projects cancelled by end of 2027.

[^14]: Marty Cagan / SVPG, [A Vision For Product Teams](https://www.svpg.com/a-vision-for-product-teams/). Product teams must maximise problems-solved, not features-delivered. Bottleneck is direction and learning, not implementation.

[^15]: Agility at Scale, [SAFe Built-in Quality When AI Agents Write the Code](https://agility-at-scale.com/safe/safe-built-in-quality-when-ai-agents-write-the-code/). Harness Architect as staff-level role. 6-layer harness engineering governance. AI agents produce 98% more code with 1.7× more defects.
