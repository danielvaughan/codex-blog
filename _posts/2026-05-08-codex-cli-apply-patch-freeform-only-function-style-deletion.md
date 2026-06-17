---
title: "Codex CLI Deletes Function-Style apply_patch: Freeform Is Now the Only Way"
description: "PR #21651, merged on 8 May 2026, removes the JSON/function-style apply_patch tool entirely. Freeform is now the sole supported invocation path."
date: 2026-05-08T00:00:00+00:00
last_modified_at: 2026-06-17T22:07:53+01:00
tags: [codex-cli, apply_patch, freeform, architecture, tools, v0.130]
category: deep-dive
parent: "Articles"
nav_order: 931
---

![Sketchnote diagram for: Codex CLI Deletes Function-Style apply_patch: Freeform Is Now the Only Way](/sketchnotes/articles/2026-05-08-codex-cli-apply-patch-freeform-only-function-style-deletion.png)


# Codex CLI Deletes Function-Style apply_patch

PR [#21651](https://github.com/openai/codex/pull/21651), merged on 8 May 2026, removes the JSON/function-style `apply_patch` tool entirely. Freeform is now the sole supported invocation path.

## What Changed

Before v0.130-alpha.7, Codex maintained two parallel ways for models to invoke `apply_patch`:

1. **Function-style (JSON)** -- the model emits a structured JSON tool call with a schema-conformant payload.
2. **Freeform (custom tool)** -- the model emits unstructured patch text as a custom tool call, and Codex parses it directly.

The function-style variant has now been deleted:

- `ApplyPatchToolType::Function` variant removed from the enum
- JSON `apply_patch` spec and handler dropped
- All tests and SSE fixtures migrated to freeform calls
- `apply_patch_tool_type = freeform` remains as the only configuration value

## Why Freeform Won

The PR author (@pakrym-oai) states the rationale plainly: "Keeping the old JSON/function-style registration and parsing path left another way for models and tests to invoke apply_patch, which made the tool surface harder to reason about."

Three factors drove this:

1. **Model behaviour** -- modern LLMs (GPT-5, o3, codex-mini) naturally emit diff-like patch text rather than structured JSON payloads. Freeform aligns with how models actually want to write patches.
2. **Surface simplification** -- two invocation paths meant two code paths, two sets of tests, two potential failure modes. Deleting one halves the maintenance burden.
3. **Bedrock compatibility** -- Amazon Bedrock catalog metadata already used freeform. The function-style path was an extra translation layer for cross-provider support.

## Impact

- **Enterprise users** with custom toolchains that relied on JSON-style `apply_patch` calls must migrate to freeform.
- **Hook authors** who intercepted `apply_patch` function calls in `PreToolUse` hooks should verify their hooks still fire for custom tool types.
- **The draft PR #21687** ("Enable apply_patch freeform by default") is now moot -- there is no alternative to enable.

## Broader Pattern

This follows a pattern in Codex's architecture: start with multiple approaches, let usage data pick the winner, then delete the loser. The same happened with sandbox transport (WebSocket vs stdio -- stdio won) and environment providers (env-var vs config-file -- config-file is now primary with env-var as fallback).

For agentic workflows, the freeform-only apply_patch means patch generation is more predictable and easier to govern via hooks -- there is exactly one code path to audit.

## Source

- PR: [github.com/openai/codex/pull/21651](https://github.com/openai/codex/pull/21651)
- Release: [v0.130.0-alpha.7](https://github.com/openai/codex/releases/tag/rust-v0.130.0-alpha.7) (8 May 2026)
