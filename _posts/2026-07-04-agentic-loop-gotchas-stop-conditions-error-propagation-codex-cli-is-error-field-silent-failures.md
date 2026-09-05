---
title: "Agentic Loop Gotchas: Six Stop-Condition and Error-Propagation Mistakes That Cause Silent Failures in Codex CLI"
description: "An agentic loop is simple in theory — call the model, execute tools, feed results back, repeat until done. In practice, mishandling stop conditions and error propagation creates agents that silently stop too early, loop forever, or proceed on corrupted state."
type: Technical Article
timestamp: 2026-07-04T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-07-04-agentic-loop-gotchas-stop-conditions-error-propagation-codex-cli-is-error-field-silent-failures"
tags: ["codex-cli", "agentic-loop", "stop-reason", "error-handling", "is-error", "anti-patterns", "reliability", "goal-mode"]
date: 2026-07-04T09:00:00+00:00
last_modified_at: 2026-09-05T10:08:50+01:00
---
# Agentic Loop Gotchas: Six Stop-Condition and Error-Propagation Mistakes That Cause Silent Failures in Codex CLI


---

An agentic loop is simple in theory — call the model, execute tools, feed results back, repeat until done. In practice, mishandling stop conditions and error propagation creates agents that silently stop too early, loop forever, or proceed on corrupted state. These failures are insidious because the agent appears to complete successfully. You only discover the problem when reviewing output hours later, or when a downstream consumer hits corrupted data.

The Claude Certified Architect (CCA-F) exam tests these patterns extensively because they represent the most common source of production agentic failures. The same gotchas apply to Codex CLI, whether you are running interactive sessions, `codex exec` pipelines, or goal-mode orchestrations.

---

## Gotcha 1: Parsing Natural Language to Detect Completion

**The mistake:** Checking whether the agent's text response contains phrases like "I'm done", "task complete", or "all finished" to decide whether to stop the loop.

**Why it breaks:** The model's text output is not a structured signal. It can say "I'm done with the first file" while intending to continue with the second. It can say "that completes the task" as part of explaining what it did, then issue another tool call in the same response. Text is narrative, not protocol.

**What Codex CLI actually uses:** The `stop_reason` field in the API response is the *only* reliable signal:

| stop_reason | Meaning | Correct action |
|------------|---------|----------------|
| `tool_use` | Model wants to call tools | Continue the loop — execute tools, return results |
| `end_turn` | Model believes the task is complete | Stop the loop |
| `max_tokens` | Output was truncated | Extend output or summarise and continue |
| `refusal` | Model refused the request | Log, report to user, stop |
| `pause_turn` | Model yielding for human input | Wait for input or provide context |

**The fix for custom harnesses:**

```python
while True:
    response = client.messages.create(...)

    # Process any tool calls in the response
    tool_results = execute_tool_calls(response)

    # THE ONLY CORRECT TERMINATION CHECK
    if response.stop_reason != "tool_use":
        break

    # Feed results back
    messages.append(response)
    messages.append(tool_results)
```

**For Codex CLI users:** This is handled automatically in interactive mode and goal mode. But if you are building automation around `codex exec` with `--json` output and parsing the response programmatically, check the structured output — never grep the text for completion phrases.

---

## Gotcha 2: Using Iteration Caps as the Primary Stop Mechanism

**The mistake:** Setting `max_iterations = 10` (or similar) as the main way to prevent runaway loops.

**Why it breaks in both directions:**

- **Too low:** A legitimate multi-file refactoring might need 30+ tool calls (read files, plan, write, test, fix, repeat). Cutting off at 10 means the agent stops mid-task, leaving a half-refactored codebase.
- **Too high:** An agent stuck in a retry loop (failing test → same fix → failing test) wastes hundreds of API calls before hitting the cap.

The underlying problem: an iteration cap cannot distinguish between productive progress and unproductive looping. It is a blunt instrument applied without understanding.

**The correct pattern:** Use `stop_reason` as the primary control. Use iteration caps only as a *safety net* for catastrophic scenarios (infinite loops due to API bugs, malformed tool responses):

```toml
# config.toml — safety net, not primary control
[agent]
max_turns = 200          # Emergency brake — should never be hit in normal operation
token_budget = 500000    # Cost ceiling — more meaningful than turn count
```

**For goal mode:** Codex CLI's goal mode already handles this correctly — it uses the model's own completion signal and budget constraints rather than arbitrary turn limits. If you find yourself lowering `max_turns` to "keep costs down", you have a prompt/scope problem, not a configuration problem.

---

## Gotcha 3: Suppressing Errors with Status OK

**The mistake:** Tool implementations that return `{"status": "ok", "data": null}` or `{"status": "success", "result": ""}` when they actually failed — because the developer did not want the agent to "get confused" by error messages.

**Why it breaks:** The model cannot distinguish between "the operation succeeded but returned nothing" and "the operation failed." It proceeds with its plan based on corrupted state. Three turns later, it writes code that depends on a file that was never created, references data that was never fetched, or commits changes to a branch that does not exist.

**The `is_error` pattern:**

When returning tool results in the API, the `is_error` field explicitly tells the model "this was a failure, adjust your plan":

```json
{
  "type": "tool_result",
  "tool_use_id": "call_abc123",
  "is_error": true,
  "content": "Failed to read /src/main/java/App.java: file not found. The file may have been renamed or moved during the previous refactoring step."
}
```

**Why this matters for Codex CLI:** When writing MCP servers or custom tools for Codex CLI, your tool implementation should:

1. **Return `is_error: true`** with a descriptive message on failure — never swallow errors
2. **Include recovery suggestions** in the error message ("file not found — check if the previous rename completed")
3. **Never return empty success** when the operation had no effect

```typescript
// MCP server tool handler — correct error propagation
async function readFileHandler(params: { path: string }) {
  try {
    const content = await fs.readFile(params.path, 'utf-8');
    return { content: [{ type: "text", text: content }] };
  } catch (err) {
    return {
      isError: true,
      content: [{
        type: "text",
        text: `Cannot read ${params.path}: ${err.message}. Check the path exists and the sandbox allows read access to this location.`
      }]
    };
  }
}
```

---

## Gotcha 4: Not Handling `max_tokens` Distinctly from `end_turn`

**The mistake:** Treating any non-`tool_use` stop reason as "the agent is done."

**Why it breaks:** `max_tokens` means the model's output was truncated mid-sentence — it did not finish its thought or its tool call. If you stop the loop here, you get a half-written file, an incomplete plan, or a malformed JSON tool call that never executes.

`model_context_window_exceeded` is even more dangerous — it means the conversation has grown beyond what the model can process. Simply retrying will hit the same wall.

**The correct multi-branch handling:**

```python
match response.stop_reason:
    case "tool_use":
        # Continue — execute tools and loop
        pass
    case "end_turn":
        # Agent believes task is complete — stop
        break
    case "max_tokens":
        # Output truncated — extend or compact
        messages.append({"role": "user", "content": "Continue from where you left off."})
    case "refusal":
        # Model refused — cannot retry the same request
        log_refusal(response)
        break
    case "pause_turn":
        # Waiting for human — in automation, provide default or stop
        break
    case "model_context_window_exceeded":
        # Critical — must reduce context before continuing
        messages = compact_messages(messages)
    case _:
        # Unknown — log and stop safely
        log_unknown_stop_reason(response)
        break
```

**For Codex CLI:** In interactive sessions, the CLI handles these branches internally. In `codex exec` pipelines and goal mode, `max_tokens` truncation is the most common silent failure — the agent's final output is cut off, and downstream scripts receive incomplete JSON or half-written code.

**The defence:** Set generous `max_output_tokens` for generation-heavy tasks, and always validate structured output before using it downstream.

---

## Gotcha 5: Immediate Termination on First Tool Error

**The mistake:** Building retry logic that gives up immediately when a tool call fails, or having no retry logic at all.

**Why it breaks:** Many tool failures are transient:

- File locks held by another process (retry in 1 second)
- Network timeouts to an MCP server (retry with backoff)
- Rate limits on external APIs (retry after delay)
- Compilation errors from partial writes (the agent hasn't finished writing all files)

Terminating immediately means the agent never recovers from situations that would resolve in 2-3 seconds.

**The correct pattern — graduated retry:**

```python
MAX_RETRIES = 3

for attempt in range(MAX_RETRIES):
    result = execute_tool(tool_call)

    if not result.is_error:
        break

    if attempt < MAX_RETRIES - 1:
        # Feed the error back to the model — it may adjust its approach
        messages.append(tool_error_result(result))
        response = client.messages.create(...)  # Model sees the error
        # Model may try a different approach on next tool call
    else:
        # Final failure — propagate to model with full context
        messages.append(final_failure_result(result, attempts=MAX_RETRIES))
```

**For Codex CLI:** The CLI's built-in retry logic handles transient failures for its own tools (file reads, bash commands). But MCP server tools and custom integrations need their own retry handling. If your MCP server returns errors for transient conditions, the agent will often attempt a different approach rather than retrying — which may be worse. Consider implementing retry logic *inside* your MCP server rather than relying on the agent to retry.

---

## Gotcha 6: Subagent Errors That Vanish at the Coordinator Level

**The mistake:** Spawning subagents (via `--decompose parallel` or goal mode's internal parallelism) and aggregating their results without checking individual completion status.

**Why it breaks:** A coordinator spawns three subagents: one to refactor module A, one for module B, one for module C. Module B's subagent hits a compilation error and produces partial output. The coordinator receives all three results, does not notice that B's result is incomplete, and synthesises a "completed" summary that omits the B module changes.

The result: two-thirds of the refactoring is done, the coordinator reports success, and module B is left in an inconsistent state.

**The correct pattern:**

```markdown
# In AGENTS.md — instruction for coordinator behaviour

## Subagent result handling

When aggregating results from parallel subagents:
1. Check each subagent's exit status explicitly
2. If ANY subagent reports failure, do NOT mark the overall task as complete
3. Report partial completion with specific details of which subtasks failed
4. Attempt recovery only if the failure is isolated and does not affect other subtasks
```

For custom harnesses:

```python
results = await asyncio.gather(*[run_subagent(task) for task in subtasks])

failures = [r for r in results if r.status == "failed"]
if failures:
    # Do NOT suppress — report structured failure
    return CoordinatorResult(
        status="partial_failure",
        completed=[r.task_id for r in results if r.status == "success"],
        failed=[{"task_id": r.task_id, "error": r.error} for r in failures],
        summary=f"{len(results) - len(failures)}/{len(results)} subtasks completed"
    )
```

---

## The Meta-Pattern: Structured Signals Over Natural Language

Every gotcha in this article reduces to the same principle: **use structured, machine-readable signals for control flow; reserve natural language for communication with humans.**

| Decision | Wrong signal | Right signal |
|----------|-------------|-------------|
| Should the loop continue? | Model says "I'm done" | `stop_reason == "tool_use"` |
| Did the tool succeed? | Empty response body | `is_error: true/false` |
| Is the output complete? | Text ends with a period | `stop_reason != "max_tokens"` |
| Did the subagent finish? | Summary sounds complete | Exit status + structured result |
| Should we retry? | Error message "looks transient" | Error category + retry policy |

The agentic loop is a state machine. State machines run on structured transitions, not vibes. Every time you make a control-flow decision based on natural language interpretation, you introduce a probabilistic failure mode into what should be a deterministic system.

Codex CLI's internal loop already follows these principles. The gotchas emerge when you build automation around it (scripts parsing `--json` output), when you write MCP servers (returning results without `is_error`), or when you design multi-agent orchestrations (coordinators that trust subagent text summaries over structured status).

---

## References

- OpenAI, "Agentic Loop — Codex CLI," developers.openai.com, 2026. Stop reason semantics, tool result format, retry behaviour.
- OpenAI, "Goal Mode — Codex CLI," developers.openai.com, 2026. Budget-based termination, maker-verifier separation.
- CCA-F exam anti-patterns (Domain 1). "Parsing natural-language signals to terminate the loop" and "Arbitrary iteration caps as the primary stop mechanism" are confirmed exam distractors.
- Anthropic, "Building Effective Agents," anthropic.com/research, 2025. The agentic loop pattern, error handling with `is_error`, structured tool results.
