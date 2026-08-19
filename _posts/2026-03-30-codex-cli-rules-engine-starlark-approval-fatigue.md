---
title: "Codex CLI Rules Engine: Starlark Policies, Approval Fatigue, and the Coming `general_rule` Fix"
description: "> Navigating the challenges of policy enforcement in agentic workflows."
date: 2026-03-30T00:00:00+00:00
last_modified_at: 2026-08-19T12:11:03+01:00
type: Technical Article
timestamp: 2026-03-30T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-03-30-codex-cli-rules-engine-starlark-approval-fatigue"
---
![Sketchnote diagram for: Codex CLI Rules Engine: Starlark Policies, Approval Fatigue, and the Coming `general_rule` Fix](/sketchnotes/articles/2026-03-30-codex-cli-rules-engine-starlark-approval-fatigue.png)

# Codex CLI Rules Engine: Starlark Policies, Approval Fatigue, and the Coming `general_rule` Fix

> Navigating the challenges of policy enforcement in agentic workflows.

Codex CLI's rules engine lets teams define executable policies that govern what the agent can and cannot do -- without requiring a human to approve every action. This article covers the architecture (Starlark policy files, the Guardian subagent), the current `prefix_rule` system and its limitations, the approval fatigue problem it creates, and the proposed `general_rule()` fix.

---

## Architecture Overview

The rules engine sits between the Codex CLI agent loop and the actions it wants to execute:

```
codex run
  --> CLI
    --> Guardian Subagent ("I enforce the rules!")
      --> Starlark Policies
        --> Smart Approvals
          --> Security + Automation
```

The **Guardian subagent** is the enforcement point. It loads `.rules` files written in Starlark, evaluates each proposed action against the defined policies, and either approves, denies, or escalates to the human.

---

## The `.rules` File

Rules files use **Starlark** (a Python-like configuration language originally developed for Bazel). They are loaded and evaluated by the Guardian subagent at runtime.

```python
# my_project.rules
load("...", "prefix_rule")

rule = prefix_rule(
    name = "allow-npm-install",
    prefix = "npm install",
    action = "approve"
)
```

### Why Starlark?

- **Deterministic:** No side effects, no imports, no I/O -- safe to evaluate untrusted policy files
- **Familiar syntax:** Python-like, so most developers can read and write rules immediately
- **Hermetic execution:** The Guardian evaluates rules in a sandboxed Starlark interpreter

---

## `prefix_rule()` Semantics

The current primary rule type is `prefix_rule()`, which matches commands by their prefix:

```
Input Data --> prefix_rule match?
  --> YES: Approve (auto-run)
  --> NO: Deny/Flag (require human approval or block)
```

### Example Rules

```python
# Approve common safe commands
prefix_rule(prefix="npm install", action="approve")
prefix_rule(prefix="npm test", action="approve")
prefix_rule(prefix="git status", action="approve")
prefix_rule(prefix="cat ", action="approve")

# Block dangerous commands
prefix_rule(prefix="rm -rf /", action="deny")
prefix_rule(prefix="curl | bash", action="deny")
```

### The Problem with `prefix_rule`

Prefix matching is **too narrow** for real-world agentic workflows:

1. **Every new prefix requires a new rule.** When the agent invents a slightly different command (`npm ci` instead of `npm install`, `yarn add` instead of `npm install`), it falls through to manual approval.

2. **Community rule sets are fragile.** Teams share large `prefix_rule` lists, but maintaining them is:
   - High overhead (hundreds of rules for common workflows)
   - Prone to errors (typos, missing variants)
   - Hard to scale (every new tool or command pattern needs a new entry)

3. **Pattern explosion.** Agentic workflows generate creative command variants that prefix matching cannot anticipate.

---

## The Approval Fatigue Problem

The consequence of narrow `prefix_rule` matching is **approval loop fatigue**:

```
APPROVAL REQUIRED
APPROVAL REQUIRED
APPROVAL REQUIRED
APPROVAL REQUIRED
...
```

In practice, developers face endless manual approvals for every new command prefix the agent tries. This creates several problems:

- **Developer velocity drops** -- the agent is blocked waiting for approval, and the developer is interrupted
- **Developers start rubber-stamping** -- after the 50th approval, humans stop reading and just hit "approve," defeating the security purpose
- **Innovation is throttled** -- the agent cannot explore creative solutions because every novel command needs approval
- **Agentic pod workflows suffer** -- autonomous agents in CI/CD and pod-based workflows cannot function with constant approval requirements

### Impact on Agentic Pod Workflows

Agentic pods -- where multiple Codex CLI agents work on different parts of a project simultaneously -- are particularly affected. Each agent generates unique command sequences, and the combined approval volume makes human oversight impractical. The result: either teams use `--yolo` (no safety) or they experience constant interruption (no productivity).

---

## The Current Workaround: Shared Community Rule Sets

Teams currently address approval fatigue by maintaining and sharing large `prefix_rule` collections:

```python
# community_rules.rules -- maintained by the team/community
load("...", "prefix_rule")

# Package managers (47 rules)
prefix_rule(prefix="npm install", action="approve")
prefix_rule(prefix="npm ci", action="approve")
prefix_rule(prefix="npm test", action="approve")
prefix_rule(prefix="npm run build", action="approve")
prefix_rule(prefix="yarn add", action="approve")
prefix_rule(prefix="yarn install", action="approve")
prefix_rule(prefix="pip install", action="approve")
prefix_rule(prefix="pip3 install", action="approve")
# ... dozens more ...

# Git commands (23 rules)
prefix_rule(prefix="git status", action="approve")
prefix_rule(prefix="git diff", action="approve")
prefix_rule(prefix="git log", action="approve")
prefix_rule(prefix="git add", action="approve")
# ... more ...

# Build tools (31 rules)
prefix_rule(prefix="make ", action="approve")
prefix_rule(prefix="cargo build", action="approve")
prefix_rule(prefix="cargo test", action="approve")
prefix_rule(prefix="go build", action="approve")
prefix_rule(prefix="go test", action="approve")
# ... more ...
```

This works but is **high overhead, prone to errors, and hard to scale**. When a new tool enters the workflow, every team must update their shared rules file.

---

## The Proposed Fix: `general_rule()`

The `general_rule()` proposal addresses the fundamental limitation of prefix-only matching by introducing **pattern-based, flexible policies**:

```python
# Instead of dozens of prefix_rules:
general_rule(allow_cli_safe_patterns)
```

### How `general_rule()` Differs from `prefix_rule()`

| Feature | `prefix_rule()` | `general_rule()` |
|---------|-----------------|-------------------|
| Matching | Exact prefix only | Pattern matching (globs, regex, semantic) |
| Scope | Single command | Categories of commands |
| Maintenance | One rule per command variant | One rule per pattern |
| Flexibility | Rigid | Broad, flexible policies |

### Example `general_rule()` Policies

```python
# Approve all read-only filesystem operations
general_rule(
    name = "allow-reads",
    pattern = "cat|head|tail|less|wc|file|stat|ls|find|tree *",
    action = "approve"
)

# Approve all package manager install commands
general_rule(
    name = "allow-package-install",
    pattern = "(npm|yarn|pnpm|pip|pip3|cargo|go) (install|add|get|ci)",
    action = "approve"
)

# Approve all test runners
general_rule(
    name = "allow-tests",
    pattern = "(npm|yarn|cargo|go|pytest|jest|mocha) test*",
    action = "approve"
)

# Block destructive operations regardless of prefix
general_rule(
    name = "block-destructive",
    pattern = "rm -rf /|mkfs|dd if=|:(){ :|:& };:",
    action = "deny"
)
```

### Benefits

- **Fewer rules, broader coverage** -- one `general_rule` replaces dozens of `prefix_rule` entries
- **Pattern matching, not just prefixes** -- match anywhere in the command string
- **Reduced manual intervention** -- the agent can operate more autonomously within safe bounds
- **Scales with new tools** -- patterns like `* test*` catch new test runners automatically

---

## Testing Policies: `execpolicy check`

Before deploying rules to production, validate them with the built-in policy checker:

```bash
codex execpolicy check my_policy.rules
```

This command:
- Loads and parses the `.rules` file
- Validates Starlark syntax
- Checks for conflicting rules (e.g., a rule that both approves and denies the same pattern)
- Reports coverage gaps (common commands that have no matching rule)
- Returns a pass/fail result

```
$ codex execpolicy check my_policy.rules
Parsing... OK
Conflicts... none
Coverage... 87% of common commands covered
Result: PASS
```

Always run `execpolicy check` before deploying updated rules files.

---

## A Production-Ready Default Rules File

The Codex CLI team is working on a **default rules file** that ships with the CLI:

- **Base policies** -- safe defaults for common development workflows
- **Security checks** -- blocks for known-dangerous patterns
- **Best practices** -- encoding community consensus on safe agent behavior
- **Customizable templates** -- starting points for team-specific policies

The philosophy: **start secure, customize later.** Teams can extend the default rules rather than building from scratch.

---

## Practical Checklist for Migrating to `general_rule()`

When `general_rule()` becomes available, follow this migration path:

1. **Audit existing `prefix_rules`** -- inventory all current rules and their approval patterns
2. **Identify common patterns** -- group related `prefix_rule` entries that can be consolidated
3. **Draft `general_rule()` policies** -- write pattern-based rules that replace groups of prefix rules
4. **Test with `execpolicy check`** -- validate the new rules file
5. **Communicate changes to teams** -- ensure all developers understand the new policy scope
6. **Monitor for false positives** -- watch for legitimate commands being blocked by overly broad patterns
7. **Iterate and refine** -- adjust patterns based on real-world usage data

---

## The Road Ahead

The combination of `general_rule()`, the Guardian subagent, and smart approvals points toward a future where:

- **Agentic pod workflows** run with faster deployment, autonomous decision-making within safe bounds, and increased efficiency
- **Approval fatigue disappears** for common workflows while maintaining security for novel or dangerous operations
- **Policy-as-code** becomes the standard for governing AI agent behavior, not just human-written approval lists

The rules engine is evolving from a simple prefix matcher to a full policy evaluation framework -- bringing the same rigor to AI agent governance that infrastructure-as-code brought to deployment.

---

*Sources: [sketchnotes.danielvaughan.com](https://sketchnotes.danielvaughan.com), [developers.openai.com/codex](https://developers.openai.com/codex)*
