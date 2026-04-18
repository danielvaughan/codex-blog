# Article Backlog

## New Articles — Sourced from Google Search Console Data (2026-04-18)

### High Priority

1. ~~**Codex CLI Offline Mode: What Works Without Internet**~~ ✅ Written 2026-04-18 → `2026-04-18-codex-cli-offline-local-models-air-gapped-guide.md`
   - Source: "codex cli offline mode" (10 impressions, 0 clicks)
   - Scope: Local model options, caching, what features require connectivity, air-gapped enterprise use cases

2. ~~**Codex CLI Headless and Batch Mode: Non-Interactive Automation Guide**~~ ✅ Written 2026-04-18 → `2026-04-18-codex-cli-headless-batch-mode-automation.md`
   - Source: "openai codex cli headless batch mode" (11 impressions, 0 clicks)
   - Scope: Headless execution, batch processing, background mode, performance tips for CI/CD pipelines
   - Note: May overlap with existing codex-exec articles — consider enhancing those instead

3. ~~**Codex CLI SWE-Bench Scores and Benchmark Results Explained**~~ ✅ Written 2026-04-18 → `2026-04-18-codex-cli-swe-bench-benchmark-scores-explained.md`
   - Source: "codex swe-bench score" (4 impressions, 0 clicks)
   - Scope: Official benchmark results, methodology, how to interpret scores, comparison with other tools

### Medium Priority

1. ~~**Running Multiple Codex Agent Instances: Parallel Orchestration Patterns**~~ ✅ Written 2026-04-18 → `2026-04-18-running-multiple-codex-agents-parallel-orchestration.md`
   - Source: "running multiple coding agent instances", "codex create multiple agents for different jobs"
   - Scope: Concurrent sessions, resource management, workload distribution, tmux/screen patterns

2. ~~**Codex CLI Proxy Configuration: SOCKS, HTTP, and Corporate Networks**~~ ✅ Written 2026-04-18 → `2026-04-18-codex-cli-proxy-configuration-socks-http-corporate-networks.md`
   - Source: "codex cli socks proxy"
   - Scope: Proxy setup, corporate firewall traversal, environment variable configuration

3. ~~**Transferring ChatGPT Conversations to Codex CLI**~~ ✅ Written 2026-04-18 → `2026-04-18-transferring-chatgpt-conversations-to-codex-cli.md`
   - Source: "transfer chatgpt chat to codex"
   - Scope: Export/import workflows, context preservation, practical migration steps

4. **Codex CLI vs Codex Cloud: When to Use Each**
   - Source: "codex vs cloud"
   - Scope: Decision framework, cost comparison, capability differences, hybrid workflows

5. **How to Make Codex CLI and Claude Code Work Together**
   - Source: "how to make codex and claude code work together"
   - Scope: Practical integration patterns, MCP bridging, cross-tool workflows, when to use which
   - Note: Existing comparison articles exist but may lack practical "how to use both" guidance

6. **Codex CLI Token Usage and Cost by Reasoning Effort Level**
   - Source: "codex token usage comparison effort"
   - Scope: Token consumption by effort setting, cost optimization, when to use high vs low effort

7. **Codex CLI Agent Loop Explained for Beginners**
    - Source: "codex loop", "understanding codex runtime agent loop", "claude code rust loop explained"
    - Scope: Simplified explanation of the agent execution loop, iteration patterns, how to debug loops

### Lower Priority

1. **Codex CLI Jira Integration: Atlassian MCP Server Setup**
    - Source: "codex jira plugin" (4 impressions)
    - Scope: Jira MCP server configuration, ticket-driven development workflows
    - Note: Existing codex-cli-jira-atlassian-mcp-server article exists — may just need SEO enhancement

2. **Codex CLI HIPAA and Compliance Guide**
    - Source: "codex hipaa" (2 impressions)
    - Scope: HIPAA compliance checklist, data handling, audit trails
    - Note: Existing codex-cli-regulated-environments-hipaa-soc2 covers this — enhance instead

3. **Codex CLI Guardian Approval: Configuring Auto-Review Policies**
    - Source: "codex guardian_approval" (4 impressions)
    - Scope: Guardian approval configuration reference, policy examples
    - Note: Existing guardian articles may cover this — enhance with "guardian_approval" keyword

4. **OpenAI Codex CLI Official Documentation Guide (2026)**
    - Source: "openai codex cli official docs 2026" (20 impressions, 0 clicks)
    - Scope: Curated guide to official documentation, getting started pathway, resource directory
    - Note: High impressions — users want an authoritative docs starting point

---

## New Articles — Sourced from Book Reader Feedback (2026-04-18)

Reader feedback highlighted that the value of agentic engineering doesn't land fast enough — people need concrete "before and after" examples, not just theory. These article ideas bridge that gap by showing specific, pictureable outcomes.

### High Priority

1. **Your First 30 Minutes with Codex CLI: From Install to First Fix**
    - Source: Book reader feedback — "what can a developer actually build or run?"
    - Scope: Zero-to-working walkthrough. Install → point at a real repo → give it a bug → watch the agent loop → review the diff. One terminal command, one concrete result. Show elapsed time, what the agent did step by step, and the before/after.
    - Angle: The article equivalent of the book's proposed "First 30 Minutes" fast-track. Standalone value for anyone evaluating Codex CLI.

2. **Before and After: 5 Developer Workflows Transformed by Codex CLI**
    - Source: Book feedback — "one concrete outcome they can picture immediately"
    - Scope: Side-by-side comparisons of manual vs agentic workflows: (1) Bug fix from Sentry alert — 45 min manual → 4 min agentic, (2) PR code review — 25 min manual → 3 min agentic, (3) Test coverage gap — 2 hours manual → 12 min agentic, (4) Multi-file refactor — half a day manual → 20 min parallel agents, (5) CI failure triage — 30 min manual → 5 min automated. Each with specific commands and real timings.
    - Angle: The "show don't tell" article. Every comparison should be pictureable in 10 seconds.

3. **The Codex CLI Agent Loop Explained: What Actually Happens When You Hit Enter**
    - Source: Book feedback ("complex or fragmented") + Search Console ("understanding codex runtime agent loop", "codex loop")
    - Scope: Visual walkthrough of a single agent session from prompt to commit. Annotated terminal output showing each phase: file discovery → context gathering → reasoning → tool calls → test execution → commit. Mermaid sequence diagram. Demystify the black box.
    - Angle: Addresses both the book reader's "fragmented" concern and the Search Console demand for loop explanations. Cross-link to book chapters.

4. **What You Can Build with Codex CLI: 10 Real-World Setups from Simple to Advanced**
    - Source: Book feedback — "How are you getting developers to quickly see what they'll be able to build?"
    - Scope: A progression of 10 concrete setups, each with a one-paragraph description and the key commands/files needed: (1) Single-command bug fix, (2) AGENTS.md-driven project conventions, (3) Automated PR review hook, (4) MCP integration with external service, (5) Parallel worktree refactor, (6) CI pipeline with codex exec, (7) Guardian auto-review, (8) Multi-agent pod with Designer-Developer-Tester, (9) Cost-managed team deployment, (10) Full agentic engineering factory.
    - Angle: A "menu" article — readers can scan it in 60 seconds and find their entry point. Each item links to the relevant book chapter and detailed article.

5. **I Used This Setup → This Is What Changed: An Agentic Engineering Case Study**
    - Source: Book reader's exact framing — "I used this setup → this is what the agent is now doing for me → this is what changed in my workflow"
    - Scope: A narrative case study following one developer (could be Daniel's own experience) through the progression: week 1 (single agent, bug fixes), week 2 (AGENTS.md + approval modes), week 3 (hooks + MCP), week 4 (parallel agents + CI integration). Show concrete metrics: time saved, PRs merged, bugs caught, workflow changes. Include the failures and learning curves, not just the wins.
    - Angle: The "testimonial article" that proves the book's promise. Authentic, specific, honest about limitations.

### Medium Priority

1. **Codex CLI for the Sceptic: Honest Answers to "Why Should I Bother?"**
    - Source: Book feedback about agentic AI content feeling "complex or fragmented"
    - Scope: Address the top 7 objections: "It's just fancy autocomplete," "I'll spend more time fixing AI code than writing my own," "It'll hallucinate and break everything," "My codebase is too complex," "It's too expensive for daily use," "I'll lose my coding skills," "My company won't allow it." For each: honest assessment, when the objection is valid, when it isn't, and evidence.
    - Angle: The article you send to a sceptical colleague. No hype, no dismissal — just honest engineering trade-offs.

2. **From ChatGPT to Codex CLI: What Changes When Your AI Can Actually Run Code**
    - Source: Search Console ("transfer chatgpt chat to codex") + book feedback about clarity
    - Scope: For developers who use ChatGPT for coding help but haven't tried agentic tools. The key shift: from "copy-paste suggestions" to "autonomous execution in your actual codebase." Show the same task done in ChatGPT vs Codex CLI side by side. Cover: what Codex can see that ChatGPT can't, why the sandbox matters, when to still use ChatGPT instead.
    - Angle: Bridge article for the largest audience — ChatGPT users who don't yet know what they're missing.
