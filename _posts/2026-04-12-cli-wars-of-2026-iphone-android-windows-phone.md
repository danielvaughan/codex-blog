---
title: "Three Terminals, Three Fates"
date: 2026-04-12T07:00:00+00:00
featured: true
tags:
  - claude-code
  - codex-cli
  - gemini-cli
  - competitive-analysis
  - market-position
  - developer-experience
  - ecosystem
  - opinion
---

![Sketchnote diagram for: Three Terminals, Three Fates](/sketchnotes/articles/2026-04-12-cli-wars-of-2026-iphone-android-windows-phone.png)

In January 2010, Android outsold iPhone for the first time. It had more carriers, more handsets, and a lower price point. None of that mattered to the average consumer walking into a phone shop. They wanted the phone that worked. Apple had 28% market share and 66% of the industry's profits[^1]. Sixteen years later, the same pattern is playing out in AI coding terminals — and if you are picking a tool to build your career around, understanding which position each player occupies matters more than any benchmark score.

## The Three Positions

Three AI CLI tools dominate the terminal in April 2026: Claude Code, Codex CLI, and Gemini CLI. Each has shipped impressive features. Each has vocal supporters. But developer adoption data, retention metrics, and community sentiment tell a story that has nothing to do with feature checklists and everything to do with platform dynamics the industry has seen before.

**Claude Code is the iPhone.** The premium product that defines the category, captures disproportionate revenue, and generates the kind of loyalty that borders on identity.

**Codex CLI is Android.** The open-source alternative that matches or exceeds the leader on specs, wins on flexibility and price, and still trails in the metric that matters most: whether developers *choose* it when money is not the constraint.

**Gemini CLI is Windows Phone.** Backed by a company with more infrastructure than anyone, impressive on paper, and struggling to convince developers it has a soul.

The comparison is not perfect — no analogy ever is. But the structural dynamics are close enough to be instructive, and the data supports each position more strongly than you might expect.

## Claude Code: The iPhone Effect

The numbers are hard to argue with. Claude Code has over 2 million weekly active users and is growing fast[^2]. Anthropic's revenue from Claude Code alone has hit $2.5 billion ARR, doubling since the start of 2026[^3]. In the JetBrains developer survey of 10,000+ developers, Claude Code reached 18% work adoption — a 6x increase from 3% just nine months earlier[^4]. The Pragmatic Engineer survey of 15,000 developers found that 46% named Claude Code their "most loved" AI coding tool, more than Cursor and GitHub Copilot combined[^5].

But the number that matters most is retention: **41% of Claude Code users return at least three times per week**[^6]. That is not a tool people try and abandon. That is a tool people build habits around.

The iPhone parallel runs deeper than popularity. What made the iPhone win was not any single feature — Android had multitasking, widgets, and file system access years before Apple. What made it win was *coherence*. The camera worked with the photo library. The photo library synced to the cloud. The cloud synced to the laptop. Each piece was individually unremarkable; together they created switching costs that had nothing to do with hardware specs.

Claude Code operates the same way. The `CLAUDE.md` file is not just a configuration format — it is a habit. Once you have spent weeks encoding your team's conventions, review standards, and architectural decisions into that file, moving to another tool means rewriting institutional knowledge. The `/compact` command, the permission model, the way Claude Code handles multi-file refactors with architectural awareness rather than file-by-file edits — none of these features are impossible to replicate. But together, they create an experience where the tool disappears and the work remains. That is what "it just works" actually means, and it is why Claude Code users report an NPS of 54 and a CSAT of 91% — the highest of any AI coding tool measured[^6].

The March 2026 source code leak is a revealing data point. When Claude Code's 512,000-line TypeScript codebase was accidentally exposed through an unminified npm package, the community reaction was mixed but telling[^7]. Some developers were genuinely impressed by the engineering. Others criticised specific choices — regexes for sentiment analysis, an "Undercover Mode" that hid Anthropic employee contributions to open source projects. Anthropic filed DMCA takedowns against mirrors, reinforcing that this is proprietary code despite being publicly viewable. But the net effect on developer sentiment was closer to a boost than a crisis. When your competitors leak your source and your reputation improves, you have iPhone-grade brand equity.

## Codex CLI: The Android Advantage (and the Android Problem)

The broader Codex platform — CLI, IDE extension, and cloud — has 3 million weekly users as of April 8, 2026[^8]. That number is not directly comparable to Claude Code's 2 million+ weekly CLI users because it aggregates across multiple surfaces, but it signals strong overall traction for OpenAI's coding tools. Codex CLI itself is open source (Apache 2.0, written in Rust), has 75,000 GitHub stars, and ships features at a pace that would exhaust most engineering teams[^9]. Both Codex CLI and Gemini CLI are genuinely open source under Apache 2.0. Claude Code, by contrast, is proprietary — the source is visible on GitHub, but the license reserves all rights to Anthropic[^17]. This matters for the analogy: Android's openness was a strategic weapon, and Codex's is too. Version 0.120.0 landed on April 11 with background agent streaming, MCP output schema support, and hook activity display. Version 0.119.0 the day before brought WebRTC voice, MCP apps, and a 60+ crate workspace extraction.

On raw speed, Codex has a meaningful edge. GPT-5.3 Codex runs at 65–70 tokens per second in standard mode — faster than Opus models, though not the order-of-magnitude gap some benchmarks suggest. The real speed story is GPT-5.3-Codex-Spark on Cerebras hardware, which hits 1,000+ tokens per second[^10]. On Terminal-Bench 2.0, which measures speed-oriented terminal tasks, Codex CLI scores 77.3% against Claude Code's 65.4%[^10]. The sandboxed execution model is genuinely superior for security-conscious workflows. The pricing — free and open source, with API costs starting at $0.75 per million input tokens on GPT-5.4-mini — makes Claude Code's $20/month Max plan look expensive.

This is the Android advantage. Better specs in specific areas, lower cost, total customisability. And yet.

In the JetBrains survey, Codex CLI shows only **3% work adoption** — one-sixth of Claude Code's 18%[^4]. Even accounting for the fact that the 3 million figure spans Codex's full product surface, the gap between "people who have tried it" and "people who use it for real work" is the Android problem in a single statistic.

The r/codex subreddit has 262,000 weekly visitors and 1,200 weekly contributors. The r/ClaudeCode subreddit has 4,200 weekly contributors — 3.5 times more active participation despite a smaller user base[^6]. This mirrors the Android dynamic precisely: more total users, less engaged per user.

Developer sentiment data from a 500+ person Reddit survey tells the same story. 65% of respondents preferred Codex in direct comparison[^11]. But when you weight by upvotes — a proxy for the broader community's agreement — the preference jumps to 80%[^11]. The people who love Codex *really* love it. They are just a narrower slice of the developer population than the people who default to Claude Code.

The emerging consensus on developer forums is a division of labour that would sound familiar to anyone who carried an Android for personal use and an iPhone for work: "Codex for keystrokes, Claude Code for commits."[^10] Codex is faster for quick edits, file generation, and sandboxed execution. Claude Code is where developers go when the stakes are higher — architectural refactors, complex multi-file changes, production-critical code reviews. The fact that this split exists at all tells you which tool developers trust more, regardless of which one they technically prefer.

## Gemini CLI: The Windows Phone Trajectory

Google has more infrastructure than Anthropic and OpenAI combined. Gemini has 750 million monthly active users across all products[^12]. The Gemini 3 Pro model offers a 1 million token context window — five times Claude Code's standard 200K. The free tier is the most generous in the market: 1,000 requests per day on Flash, 180,000 completions per month. On SWE-bench Verified, Gemini 3 Pro through Google's Antigravity scores 76.2%, beating Codex CLI's 75.2%[^13]. The Agent Development Kit just expanded to four languages with Go 1.0 and Java 1.0.0 launches this week.

On paper, Gemini CLI should be winning. This is exactly how Windows Phone looked in 2012: backed by the largest software company on Earth, competitive specs, aggressive pricing, deep integration with enterprise services. Microsoft had more developers, more enterprise relationships, and more money than Apple and Google combined. It did not matter.

Windows Phone failed because it never solved the app gap. Developers did not build for it because users were not there, and users were not there because developers had not built for it. The flywheel never spun.

Gemini CLI has its own version of this problem. The JetBrains survey shows just **6% adoption** for Google's coding tools[^4]. That is not catastrophically low, but it is one-third of Claude Code's number and moving in the wrong direction relative to the competition. More concerning is the qualitative data.

Starting March 25, 2026, Gemini CLI users hit severe rate limiting — 429 errors that affected paying customers and free users identically[^14]. Reports surfaced of the CLI silently downgrading users from Gemini Pro to Flash without notification[^15]. File destruction incidents — the CLI overwriting files instead of appending to them — eroded trust in ways that are difficult to recover from[^15]. GitHub issues appeared with titles like "BURN IT DOWN" — not the measured criticism of a competitive product, but the frustration of people who wanted it to work and watched it fail[^15].

One HackerNoon analysis titled "Google's Gemini CLI Has a Reliability Problem Developers Can't Ignore" summarised the trajectory bluntly[^15]. The authentication setup is described as "painful, confusing, or not well-documented." The safety parsing system blocks legitimate shell commands. The product vision — what Gemini CLI is *for*, specifically, that the other two are not — has not crystallised.

This does not mean Gemini CLI is dead. Windows Phone lasted from 2010 to 2017 and shipped some genuinely innovative features (Live Tiles, People Hub, Cortana integration). Gemini CLI has a real niche: developers working with massive codebases who need the 1M token context window, budget-conscious teams who need the free tier, and organisations already deep in the Google Cloud stack. The ADK expansion to four languages is a smart platform play.

But a niche is not a platform. The developer sentiment pattern — use Gemini for exploration and quick questions, then switch to Claude Code or Codex for anything that matters — is the exact same pattern Windows Phone users exhibited. "I keep it around for the camera, but my main phone is the iPhone." That sentence, adapted for CLI tools, appears in developer forum posts with uncomfortable frequency.

## Why Experience Beats Features

The smartphone wars taught the industry a lesson that the CLI wars are re-teaching: **the tool that wins is not the one with the best specs. It is the one that disappears during use.**

When a benchmark showed the Harness Performance Terminal-Bench results for Claude Code, ForgeCode, and Terminus-KIRA all running the identical Opus 4.5 model, they scored 17 problems apart on 731 total issues[^16]. Same model, different harness, different results. The agent architecture — the UX layer — accounts for more variance than the underlying model.

This is why Claude Code's 41% weekly retention dwarfs the competition despite costing more and running slower. It is why Codex's 3 million users across all surfaces convert to only 3% CLI work adoption while Claude Code's 2 million CLI users convert to 18%. The feature that matters most is not on any spec sheet: it is whether the tool earns enough trust that you stop thinking about the tool and start thinking about your code.

Apple understood this in 2007. Anthropic understands it in 2026.

## The Codex Counter-Argument

The Android analogy has a sequel that Codex CLI's supporters would want noted. Android did not stay in second place on every metric. By 2015, it held 85% global market share[^1]. Samsung's Galaxy line achieved premium status. The ecosystem matured. The "Android is for tinkerers" narrative faded as the product improved.

Codex CLI is on a similar trajectory. The gap between v0.115 and v0.120 — shipped in the space of two weeks — is remarkable. The Rust rewrite delivers genuine performance advantages. The sandboxed execution model is architecturally superior to Claude Code's approach. OpenAI's decision to make it fully open source, combined with Sam Altman's April 1 usage limit reset to celebrate 3 million Codex users across all surfaces, signals a willingness to compete on generosity[^8].

If Codex CLI can solve its retention problem — the gap between "tried it" and "use it daily" — the Android endgame is plausible. Android won on volume, then won on quality, then won on both. Codex has the volume. The quality is arriving fast.

## The Prediction

By the end of 2026, this is the most likely scenario:

**Claude Code** holds the premium position with the highest revenue per user, the strongest retention, and the default recommendation for teams that value reliability over cost. It is the tool that senior engineers recommend to junior engineers. Market share may not grow as fast as Codex, but revenue share will remain disproportionate.

**Codex CLI** continues growing faster in absolute users and closes the quality gap. The open-source model attracts infrastructure teams, security-conscious organisations, and developers who want to understand (and modify) their tools. It becomes the default for API-key users, solo developers, and cost-optimised workflows. The "Android of CLI tools" label shifts from insult to compliment.

**Gemini CLI** survives but does not thrive. Google's infrastructure advantage keeps it relevant for large-context workloads and budget-constrained teams. The ADK platform play may find traction in enterprise Google Cloud environments. But without solving the reliability crisis and the trust deficit, it remains the third option — chosen when the other two are not available or not affordable, not when developers have a free choice.

The smartphone wars took seven years to settle. The CLI wars will move faster — AI tool cycles compress time. But the structural dynamics are the same, and the lesson is the same: in a market where the underlying technology is converging, the experience layer is the only durable moat.

Build for the tool that earns your trust. The specs will converge. The habits will not.

---

## Citations

[^1]: Asymco, "Apple captures 66% of smartphone industry profits," 2010-2015 analysis. Statista global smartphone market share data, 2010-2025.

[^2]: Gradually.ai, "Claude Code Statistics 2026," April 2026. Reports 2M+ weekly active users (up from 1.6M earlier in 2026) and 11.3M daily active users.

[^3]: The AI Corner, "Anthropic Passed OpenAI in Revenue: $30B ARR," April 2026. SaaStr, "Anthropic Just Hit $14 Billion in ARR," March 2026. Yahoo Finance, "Anthropic ARR surges to $19 billion on Claude Code strength."

[^4]: JetBrains Research, "Which AI Coding Tools Do Developers Actually Use at Work?" April 2026. Survey of 10,000+ developers. Claude Code 18% work adoption, Codex CLI 3%, Google Antigravity 6%.

[^5]: Pragmatic Engineer, "AI Tooling for Software Engineers in 2026," February 2026. Survey of 15,000 developers. Claude Code 46% "most loved," 71% of regular AI agent users use Claude Code.

[^6]: DEV Community, "Claude Code vs Codex 2026: What 500+ Reddit Developers Really Think." Neuriflux, "Claude Code Review 2026." NPS 54, CSAT 91%, 41% weekly retention.

[^7]: DEV Community, "The Great Claude Code Leak of 2026," March 2026. CNBC, "Anthropic leak: Claude Code internal source," March 31, 2026. Community reaction was mixed: architecture impressed many, but "Undercover Mode" and DMCA takedowns drew criticism. Net sentiment was positive.

[^8]: BusinessToday, "OpenAI Codex celebrates 3 million weekly users, CEO Sam Altman resets usage limits," April 8, 2026. Sam Altman's post said "codex users" — this figure spans the full Codex platform (CLI, IDE extensions, and cloud), not CLI alone. No public breakdown of CLI-only users exists. Fortune, "OpenAI sees Codex users spike to 1.6 million," March 2026.

[^9]: GitHub, openai/codex repository. 74.7K stars, 10.6K forks, 421 contributors, Apache 2.0 license (as of April 12, 2026).

[^10]: Morphllm, "We Tested 15 AI Coding Agents (2026)." Terminal-Bench 2.0 scores: Codex CLI 77.3%, Claude Code 65.4%. Token speed: GPT-5.3 Codex 65-70 tok/sec standard, GPT-5.3-Codex-Spark 1,000+ tok/sec on Cerebras. Adam Holter, "GPT-5.3-Codex-Spark: 1000 Tokens Per Second."

[^11]: DEV Community, "Claude Code vs Codex 2026 — What 500+ Reddit Developers Really Think." 65.3% preference for Codex unweighted, 79.9% weighted by upvotes.

[^12]: TechCrunch, "Google Gemini surpasses 750M monthly active users," February 2026. 2.4M active API developers, 85B API requests in January 2026.

[^13]: LocalAI Master, "SWE-Bench 2026 Leaderboard." Claude Code (Opus 4.5) 80.9%, Antigravity (Gemini 3 Pro) 76.2%, Codex CLI (GPT-5.3) 75.2%.

[^14]: DEV Community, "Google Gemini CLI's Rate Limiting Crisis: When Paying Customers Get the Same Treatment as Free Users," March 2026.

[^15]: HackerNoon, "Google's Gemini CLI Has a Reliability Problem Developers Can't Ignore," April 2026. DataCamp, "Gemini CLI vs Claude Code (2026)."

[^16]: Codex Resources, "Harness Performance Terminal-Bench: Same Model, Different Harness, Different Results," April 2026. 17-problem spread across three harnesses running identical Opus 4.5.

[^17]: GitHub, anthropics/claude-code LICENSE.md: "All rights reserved. Use is subject to Anthropic's Commercial Terms of Service." GitHub, google-gemini/gemini-cli: Apache 2.0 license (code open, backend services proprietary per ToS). CodeNote, "Gemini CLI: Apache License vs ToS Gap," April 2026.
