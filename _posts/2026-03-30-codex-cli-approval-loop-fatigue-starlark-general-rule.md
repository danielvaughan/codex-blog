---
title: "Codex CLI Approval Loop Fatigue: The prefix_rule Limitation and the Coming general_rule Fix"
date: 2026-03-30T09:00:00+00:00
description: "Why Codex's current prefix_rule-only rules engine causes approval loop fatigue in multi-agent sessions — and what the community is building to fix it."
tags:
  - security
  - approval-modes
  - subagents
  - sandbox
  - execpolicy
  - starlark
  - rules-engine
  - approval-fatigue
---

![Sketchnote diagram for: Codex CLI Approval Loop Fatigue: The prefix_rule Limitation and the Coming general_rule Fix](/sketchnotes/articles/2026-03-30-codex-cli-approval-loop-fatigue-starlark-general-rule.png)

*Published: 2026-03-30. Sources: [github.com/openai/codex/issues/15214](https://github.com/openai/codex/issues/15214) · [developers.openai.com/codex/rules](https://developers.openai.com/codex/rules) · community gist by @vertti*

---

## The Problem

In a busy Codex session, the approval dialog appears many times. Most of the time it's for commands that differ only in a trailing argument:

```
cat src/auth.ts | head -5      → prompts
cat src/auth.ts | head -10     → prompts again
cat src/main.ts | head -5      → prompts again
```

This is expected: `prefix_rule` matches on exact command prefix — `["cat"]` matches *every* cat invocation, but if you only want to allow specific patterns you have no expressive choice beyond listing prefixes literally. For inline shell scripts (`bash -c "..."`) the problem is worse: the entire script body must be an exact prefix match, making `bash -c` rules useless in practice.

The core issue was filed as **GitHub issue #15214** on March 19, 2026, by `hktonylee`, referencing four prior issues (#11298, #8537, #13175, and more) as evidence of longstanding community frustration.

> *"In many cases (over 80% of the time), commands differ from each other only in a final argument… the non-stop approval steps are actually not safe because people will get tired and approve something that should not be approved."*

**Approval fatigue → safety regression.** When developers are prompted too frequently for clearly-safe variations, they stop reading the approval dialogs and hit Accept reflexively. This undermines the entire purpose of the approval model.

---

## The Current Workaround: Shared Community Rule Sets

Before `general_rule()` arrives, the best available mitigation is to pre-populate `~/.codex/rules/default.rules` with a comprehensive allow-list of safe commands that will never need approval in normal development.

Community member **@vertti** published 79 `prefix_rule()` definitions covering the most common safe commands ([gist](https://gist.github.com/vertti/ce82aa9e2fe8679b82746294f61a4875)):

```python
# ~/.codex/rules/default.rules
# Safe file/text inspection
prefix_rule(pattern=["cat"], decision="allow")
prefix_rule(pattern=["head"], decision="allow")
prefix_rule(pattern=["tail"], decision="allow")
prefix_rule(pattern=["grep"], decision="allow")
prefix_rule(pattern=["rg"], decision="allow")
prefix_rule(pattern=["find"], decision="allow")
prefix_rule(pattern=["ls"], decision="allow")
prefix_rule(pattern=["tree"], decision="allow")
prefix_rule(pattern=["stat"], decision="allow")
prefix_rule(pattern=["wc"], decision="allow")
prefix_rule(pattern=["diff"], decision="allow")
prefix_rule(pattern=["jq"], decision="allow")

# Safe git read operations
prefix_rule(pattern=["git", "status"], decision="allow")
prefix_rule(pattern=["git", "log"], decision="allow")
prefix_rule(pattern=["git", "diff"], decision="allow")
prefix_rule(pattern=["git", "show"], decision="allow")
prefix_rule(pattern=["git", "branch", any_of(["--list", "-a", "-r"])], decision="allow")
prefix_rule(pattern=["git", "remote", any_of(["-v", "show"])], decision="allow")
prefix_rule(pattern=["git", "stash", "list"], decision="allow")
prefix_rule(pattern=["git", "ls-files"], decision="allow")

# Lockfile-respecting installs (safe — can't change versions)
prefix_rule(pattern=["npm", "ci"], decision="allow")
prefix_rule(pattern=["pnpm", "install", "--frozen-lockfile"], decision="allow")
prefix_rule(pattern=["uv", "sync", "--frozen"], decision="allow")
prefix_rule(pattern=["cargo", "fetch"], decision="allow")

# Test runners
prefix_rule(pattern=["uv", "run", "pytest"], decision="allow")
prefix_rule(pattern=["cargo", "test"], decision="allow")
prefix_rule(pattern=["go", "test"], decision="allow")
prefix_rule(pattern=["npm", "test"], decision="allow")
```

**Test your rules before deploying:**

```bash
codex execpolicy check 'git log --oneline -10'
codex execpolicy check 'rm -rf dist/'
```

---

## The Proposed Fix: `general_rule()`

The community proposal (issue #15214) extends the rules engine with a new `general_rule()` built-in that accepts any Starlark callable as a predicate:

```python
# Allow any safe read-only bash one-liners
def is_safe_read_only(cmd):
    """Accept bash -c scripts that only use read-only utilities."""
    safe_prefixes = ("cat ", "head ", "tail ", "grep ", "rg ", "ls ", "find ",
                     "echo ", "wc ", "diff ", "jq ", "stat ")
    script = " ".join(cmd)
    return any(script.startswith(p) for p in safe_prefixes)

def is_inline_bash(predicate):
    def _inner(args):
        return (
            len(args) >= 3
            and args[0] in ("bash", "zsh")
            and args[1] == "-c"
            and predicate(args[2:])
        )
    return _inner

# Apply: allow bash -c "cat ..." style commands
general_rule(predicate=is_inline_bash(is_safe_read_only), decision="allow")
```

This enables:
- **Class-based allows** — match entire categories of safe commands with a single predicate
- **Shell-script parsing** — safely inspect the content of `bash -c "..."` scripts
- **Community-shared predicates** — import rule libraries, share via skills or gists
- **`&&`/`||` chain analysis** — examine compound commands for unsafe segments

**Status as of 2026-03-30:** Open enhancement — not yet shipped. Monitor [github.com/openai/codex/issues/15214](https://github.com/openai/codex/issues/15214) and the changelog for `general_rule` to appear.

---

## Impact on Agentic Pod Workflows

In a multi-agent session with four subagents running in parallel, approval fatigue is multiplied. Each subagent issues its own tool calls independently — the approval surface grows linearly with agent count.

**Before `general_rule()`:** The safest option is to front-load `~/.codex/rules/default.rules` with the community allow-list. Invest 30 minutes writing test cases with `codex execpolicy check` to catch gaps before they surface in sessions.

**After `general_rule()`:** Replace the 79-rule list with 3-5 higher-level predicates. Write one predicate per command class (read-only inspection, test runners, locked installs, git-read) and maintain them as a versioned Starlark library checked into the repo.

**The deeper principle:** The rules engine should encode the *invariants* of your development workflow — the commands that are always safe regardless of arguments — so that approval dialogs are reserved for genuinely ambiguous decisions. If you're approving more than 2-3 commands per session, your rule set is undersized.

---

## Practical Checklist

- [ ] Copy @vertti's community rule set as a starting point: [gist](https://gist.github.com/vertti/ce82aa9e2fe8679b82746294f61a4875)
- [ ] Add project-specific safe commands to `.codex/rules/project.rules`
- [ ] Test every rule with `codex execpolicy check` before committing
- [ ] Include `match`/`not_match` test vectors in every rule definition
- [ ] Watch [issue #15214](https://github.com/openai/codex/issues/15214) for `general_rule()` — migrate when it ships
- [ ] For CI pipelines: use `[approval_policy.execpolicy] rules = false` to auto-allow `"prompt"` rules without blocking on `"forbidden"`
