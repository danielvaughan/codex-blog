---
title: "Context Budget Gotchas: Eight Ways Your Codex CLI Agent Forgets Mid-Task and How to Prevent Catastrophic Context Loss"
description: "Context is the agent's working memory. When it runs out, the agent does not stop — it forgets. The most dangerous context failures are the ones where the agent continues working confidently on an incomplete mental model."
parent: "Articles"
nav_order: 1332
type: Technical Article
timestamp: 2026-07-04T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-07-04-context-budget-gotchas-when-your-codex-cli-agent-forgets-mid-task-compaction-cache-positioning-bloat"
tags: ["codex-cli", "context-management", "compaction", "prompt-cache", "token-budget", "working-memory", "anti-patterns", "reliability"]
---
# Context Budget Gotchas: Eight Ways Your Codex CLI Agent Forgets Mid-Task and How to Prevent Catastrophic Context Loss


---

Context is the agent's working memory. When it runs out, the agent does not stop — it forgets. It continues working with complete confidence on an incomplete mental model, producing code that contradicts decisions made earlier in the same session, duplicating work it already completed, or silently dropping requirements that were discussed but compacted away.

The most dangerous context failures are invisible. The agent does not say "I've forgotten the architecture discussion from 40 minutes ago." It simply proceeds without it, making decisions that are locally reasonable but globally inconsistent.

The CCA-F exam tests context management extensively because it is the single largest source of subtle, expensive agent failures in production. These same gotchas apply directly to Codex CLI sessions — especially long-running goal-mode tasks, multi-file refactorings, and automation pipelines that process many items sequentially.

---

## Gotcha 1: Cleaning Up Context After It Is Already Bloated

**The mistake:** Running `/compact` or switching to a smaller context strategy *after* the agent has already consumed a large context and started producing inconsistent output.

**Why it breaks:** Once context is consumed and the model has generated responses based on it, the damage is done. Compaction can shrink the conversation history, but it cannot undo the decisions the agent already made based on context that will now be removed. The agent wrote file A based on a discussion that compaction will summarise into a single sentence. When it later writes file B, it has only the summary — and may contradict the detailed decisions embedded in file A.

**The correct approach — prevent bloat proactively:**

```toml
# config.toml — set the budget before work begins, not after problems appear
[agent]
token_budget = 300000     # Generous but bounded
context_strategy = "lean" # Prefer targeted reads over full-file dumps
```

**Proactive tactics:**

- **Use subagents for isolated tasks.** Each subagent gets fresh context. It cannot be polluted by unrelated earlier work in the parent session.
- **Read only what you need.** Instead of `cat entire-file.java`, use grep/glob to find the relevant section, then read only those lines.
- **Compact early, not late.** If you know a task will be long, compact after each major milestone — not when the context is already at 90%.

```markdown
# In AGENTS.md
## Context discipline
- After completing each feature/file, summarise what was done in a brief comment
- Do not hold entire file contents in context — read the section you need
- For multi-file changes, work on one file at a time, verify, then move on
```

---

## Gotcha 2: Putting Critical Instructions at the Beginning of Long Contexts

**The mistake:** Placing your most important architectural rules at the top of AGENTS.md, assuming "the agent will see them first and remember them."

**Why it breaks:** Attention research (and the CCA-F curriculum) demonstrates that model quality degrades with instruction position in long contexts. Instructions at the *end* of the context window outperform instructions at the beginning by up to 30% on recall tasks. This is the "lost in the middle" phenomenon — content in the middle of a long conversation is the most likely to be effectively ignored.

**The correct pattern:**

```markdown
# AGENTS.md — structure for attention

## Quick reference (top — seen early but may drift)
Brief overview of the project.

## Detailed rules (middle — highest risk of being ignored)
Extended discussion of patterns and examples.

## Critical constraints — ALWAYS FOLLOW (bottom — strongest recall)
- NEVER import Spring in domain package
- NEVER return JPA entities from controllers
- ALWAYS use typed identifiers (AccountId, not UUID)
- ALWAYS run ./gradlew verify before completing a task
```

**For Codex CLI specifically:** The CLI concatenates AGENTS.md files from multiple directories (global → project → local). Your most critical constraints should be in the *most local* file (closest to where the agent is working), because it appears last in the concatenated context.

**Even better:** Do not rely on context position for enforcement. If a rule is critical enough to worry about recall, it should be a **hook** (deterministic) not an instruction (probabilistic). See the hooks gotchas article.

---

## Gotcha 3: Treating Compaction as Lossless

**The mistake:** Assuming that `/compact` perfectly preserves all information and the agent will remember everything it knew before compaction.

**Why it breaks:** Compaction summarises the conversation history into a compressed form. Like any summarisation, it is lossy. It preserves the *gist* of what was discussed, but drops:

- Exact code snippets from earlier in the conversation
- Specific numerical values (port numbers, timeout values, version strings)
- The reasoning behind decisions ("we chose PostgreSQL because...")
- Subtle constraints mentioned in passing ("remember to also handle the edge case where...")

After compaction, the agent has a reliable summary of *what* was decided, but may lose *why* and *exactly how*. If the session later requires those details, the agent will either ask (best case) or hallucinate (worst case).

**The fix — externalise important state:**

```markdown
# In AGENTS.md
## State persistence
When making architectural decisions during a session:
1. Write the decision to docs/decisions/NNNN-decision-name.md (ADR format)
2. Write implementation plans to docs/plan.md
3. These persist across compaction — the agent can re-read them
```

**Codex CLI specifics:**
- Use `--scratchpad` or maintain a `PLAN.md` file that captures key decisions *outside* the conversation context
- After compaction, the agent should re-read critical state files rather than relying on the summary
- Goal mode's maker-verifier pattern naturally mitigates this: the verifier checks output against the spec, not against earlier conversation

---

## Gotcha 4: Placing Prompt Cache Breakpoints on Volatile Content

**The mistake:** Relying on prompt caching for cost efficiency, but placing cache breakpoints after content that changes frequently.

**Why it breaks:** Prompt caching works by hashing message prefixes. If you change *any* token before the cache breakpoint, the entire cached segment is invalidated. Common volatile elements that cause cache invalidation:

- Tool results (change every turn)
- File contents (change when the agent writes)
- Conversation messages (grow every turn)

When the cache invalidates, you pay full input token costs for the entire context — which on long sessions can be 5-10x more expensive than the cached rate.

**The correct breakpoint placement:**

```
[STABLE — cache this]
├── System prompt
├── AGENTS.md content
├── Tool definitions
├── Static context files (@context/architecture/hexagonal.md)
└── ← CACHE BREAKPOINT HERE

[VOLATILE — do not cache]
├── Conversation history
├── Tool results
└── Current file contents
```

**For Codex CLI:** The CLI manages caching internally, but you influence it through configuration:

```toml
# config.toml
[cache]
ttl = "1h"             # Appropriate for long sessions
# Batch/pipeline workloads need longer TTL:
# ttl = "4h"
```

**Rule:** Cache breakpoints go on *stable* content (system prompt, tool definitions, imported context files). Never on content that changes between turns.

---

## Gotcha 5: Using the Same Context Window for Read-Heavy and Write-Heavy Tasks

**The mistake:** Running a session that reads 50 files for analysis, then attempts to refactor them all in the same context window.

**Why it breaks:** Reading 50 files consumes context for *holding* that information. Writing refactored versions of those files generates output tokens that further fill the window. By the time the agent reaches file 30, it has lost attention on files 1-10 (the "lost in the middle" effect) and produces refactoring that contradicts earlier decisions.

**The correct pattern — divide and conquer:**

```bash
# Phase 1: Analysis (read-only, separate context)
codex exec --sandbox read-only \
  "Analyse all Java files in src/main. Write a refactoring plan to PLAN.md with file-by-file changes needed."

# Phase 2: Refactoring (per-file, fresh context each time)
for file in $(cat files-to-refactor.txt); do
  codex exec --sandbox workspace-write \
    "Read PLAN.md for the overall strategy. Refactor $file according to the plan. Run tests after."
done
```

Or with goal mode:

```markdown
## Task decomposition in AGENTS.md
For large refactorings (>10 files):
1. First pass: read all files, produce a plan (PLAN.md)
2. Second pass: refactor one module at a time, re-reading PLAN.md at the start of each module
3. Final pass: integration test across all modules
```

**The principle:** Keep read-intensive and write-intensive work in separate context scopes. Use files (plans, specs, ADRs) as the coordination mechanism between scopes.

---

## Gotcha 6: Assuming All Models Have the Same Context Window

**The mistake:** Switching between models (`-m o3-mini`, `-m codex-mini-latest`, `-m o4-mini`) without adjusting your context strategy.

**Why it breaks:** Different models have different context capacities and different attention characteristics. A prompt that fits comfortably in one model's 128k window might overflow another model's 32k window. And even within the same window size, different model architectures handle long-range attention differently.

**Model-aware configuration:**

```toml
# config.toml — profiles for different models
[profiles.deep-analysis]
model = "o3"
context_budget = 800000   # Large model, large context

[profiles.quick-fix]
model = "codex-mini-latest"
context_budget = 100000   # Smaller model, keep context lean

[profiles.cost-optimised]
model = "o4-mini"
context_budget = 200000   # Balance
```

**The rule:** When switching models in a pipeline (e.g., cheap model for analysis, expensive model for complex refactoring), do not share context between stages. Start each stage fresh with only the relevant input (plan file, spec, specific files to change).

---

## Gotcha 7: Loading Full Files When Sections Suffice

**The mistake:** Using `cat` or unrestricted file reads that dump entire 2000-line files into context when the agent only needs 30 lines.

**Why it breaks:** Every token of context displaces potential working memory for reasoning and output. A 2000-line file consumes ~6000 tokens — that is 6000 tokens of attention capacity that cannot be used for understanding the task, remembering the architecture, or planning the implementation.

**What Codex CLI actually recommends:**

```markdown
# In AGENTS.md
## File reading discipline
- Use grep to find the relevant section first
- Read only the lines you need (offset + limit)
- Never read a file >500 lines in full unless you need all of it
- For large files: read the structure (class/method signatures) first, then read specific method bodies
```

**In practice:**

```bash
# Bad: reads entire file
cat src/main/java/com/example/AccountService.java

# Good: finds and reads only the relevant method
grep -n "withdraw" src/main/java/com/example/AccountService.java
# → line 145
sed -n '140,170p' src/main/java/com/example/AccountService.java
```

Codex CLI's built-in tools already support targeted reads (offset, limit parameters). If your AGENTS.md does not instruct the agent to use them, it will default to reading full files — because that is the simpler operation.

---

## Gotcha 8: Running Sequential Tasks Without Context Isolation

**The mistake:** Processing 20 items in a loop within a single Codex CLI session, where each item's context pollutes the next.

**Why it breaks:** By item 15, the context contains the full history of items 1-14 — their file contents, tool results, reasoning traces, and the agent's responses. This accumulated history:

1. Displaces attention from the current item
2. Can cause cross-contamination (using patterns from item 3 when processing item 15)
3. Eventually triggers compaction, which may lose track of the current item's state

**The correct pattern — one context per item:**

```bash
#!/bin/bash
# Process each PR independently — fresh context per item
for pr in $(gh pr list --json number -q '.[].number'); do
  codex exec "Review PR #$pr. Check for security issues, style violations, and test coverage gaps. Post review comments."
done
```

Each `codex exec` invocation gets fresh context. No cross-contamination. No accumulated bloat. Each item gets the agent's full attention capacity.

**For long pipelines with shared context (e.g., multi-file refactoring within one codebase):** Use subagents or worktrees. Each subagent processes one module independently, and the coordinator aggregates results.

---

## The Meta-Pattern: Context is a Scarce Resource, Not a Convenience

Every gotcha in this article stems from treating context as unlimited — reading freely, accumulating history indefinitely, assuming the agent remembers everything equally. The reality:

| Assumption | Reality |
|-----------|---------|
| "The agent remembers the whole conversation" | Attention degrades with distance; middle content is weakest |
| "Compaction is lossless" | It's lossy summarisation; details and reasoning evaporate |
| "More context = better results" | Past a threshold, more context = attention dilution = worse results |
| "The agent will read only what it needs" | Without explicit instructions, it reads everything and fills the window |
| "I can fix context problems reactively" | Once context is consumed, decisions are made; retroactive compaction doesn't undo them |

The engineers who get the best results from Codex CLI treat context like memory allocation in systems programming: explicit, bounded, and deliberately managed. They use subagents as "memory scopes," external files as "persistent storage," and compaction as "garbage collection" — scheduled proactively, not triggered by crisis.

---

## References

- OpenAI, "Context Management — Codex CLI," developers.openai.com, 2026. Token budgets, compaction, context strategies.
- OpenAI, "Sub-agents and Parallel Execution — Codex CLI," developers.openai.com, 2026. Context isolation via parallel subagents.
- Anthropic, "Long Context Window Tips," docs.anthropic.com, 2025. Instruction positioning, the "lost in the middle" effect.
- CCA-F exam anti-patterns (Domain 5). "Cleaning up context after it's already bloated" and "Putting queries at the beginning of long contexts" are confirmed exam distractors.
- Liu et al., "Lost in the Middle: How Language Models Use Long Contexts," 2023. Empirical demonstration of positional attention degradation.
