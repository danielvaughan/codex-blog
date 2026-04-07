# Claude Code Source Leak — What 163K Lines of TypeScript Reveal About Anthropic's Engineering

**Source:** [r/ClaudeCode discussion](https://www.reddit.com/r/ClaudeCode/comments/1s8ower/now_that_its_open_source_we_can_see_why_claude/) | [DEV Community analysis](https://dev.to/kolkov/we-reverse-engineered-12-versions-of-claude-code-then-it-leaked-its-own-source-code-pij) | [Axios](https://www.axios.com/2026/03/31/anthropic-leaked-source-code-ai) | [CNBC](https://www.cnbc.com/2026/03/31/anthropic-leak-claude-code-internal-source.html)
**Author:** Community analysis (multiple contributors)
**Published:** 2026-03-31
**Date saved:** 2026-04-01
**Content age:** Current — breaking news from March 31, 2026
**Tags:** claude-code, anthropic, source-leak, architecture, engineering-practices, competitive-analysis

---

## Summary

On March 31, 2026, security researcher Chaofan Shou discovered that Anthropic's entire Claude Code CLI source code (v2.1.88, ~163,318 lines of TypeScript) was exposed via a sourcemap file bundled into the published npm package. The `.map` file referenced the full, unobfuscated TypeScript source hosted on Anthropic's R2 storage bucket. An Anthropic employee subsequently made the source available in the public domain. This is Anthropic's second major security lapse, coming days after accidentally revealing internal details about their Mythos project.

---

## Key Points

### What Was Exposed
- Complete Claude Code v2.1.88 TypeScript source — 163,318 lines
- Discoverable via npm sourcemap file pointing to Anthropic's R2 storage bucket
- No authentication required to access the source archive

### Architectural Findings

**Code Structure:**
- 64,464 lines with **zero automated tests**
- A single React component (`REPL.tsx`) sprawls to 5,005 lines / 875 KB
- `print.ts` exhibits 12 levels of nesting and ~486 branch points of cyclomatic complexity
- 74 npm dependencies; React used for terminal rendering (causing constant re-renders)
- Both Axios and `fetch` used for HTTP — inconsistent approach

**Streaming & Reliability:**
- Watchdog timeout mechanism exists but is disabled by default, hidden behind an undocumented environment variable
- Watchdog initialises too late, missing the initial connection phase where hangs occur
- Abort functions target undefined variables during critical phases, rendering them ineffective

**Silent Model Degradation:**
- System silently downgrades users from Opus to Sonnet after three consecutive server errors — with no notification
- Users paying for Opus may unknowingly receive Sonnet responses during high-load periods

**Attestation Bug:**
- A `cch=00000` sentinel corrupts conversation content when certain strings appear
- This invalidates prompt cache keys, consuming 10–20× more tokens than expected

### Hidden Features Found in Source

**KAIROS Mode:**
- An entire persistent assistant mode: always-running Claude that watches, logs, and proactively acts on things it notices
- Gated behind compile-time feature flags — absent from external builds
- References April 1–7, 2026 as a teaser window, full launch gated for May 2026

**Buddy Companion Feature:**
- A "buddy" companion system referenced in the code
- Appears to be an always-on pair programmer concept

### Community Reaction
- Developers began porting core features to Python within hours
- Architecture analyses published in multiple languages by April 1
- Issue #39755 tagged 17 Claude Code team members asking why it wasn't open-sourced — zero responses from Anthropic
- The author's assessment: "maybe Anthropic can't open source Claude Code — not because of competitive advantage (there is none — it's a CLI wrapper), but because the code quality is so poor that publishing it would be embarrassing"

---

## Relevance to Codex CLI

This leak is significant competitive intelligence for the Codex CLI ecosystem:

1. **Testing discipline**: Claude Code has zero tests across 64K lines. Codex CLI (written in Rust) has a fundamentally different engineering approach. This validates the "engineering-first" narrative.

2. **Architecture comparison**: Claude Code uses React for terminal rendering with 74 npm deps. Codex CLI uses a compiled Rust binary with minimal dependencies. The performance and reliability implications are stark.

3. **Silent degradation**: Claude Code silently downgrades models. Codex CLI's `--model` flag is explicit and transparent. This is a trust differentiator.

4. **KAIROS as a signal**: Anthropic is building always-on persistent agents. This aligns with the "agentic pod" concept (Chapter 32) — persistent, context-aware agents that watch and act proactively.

5. **Security posture**: The "safety-first" AI lab leaking its own source via an npm sourcemap is ironic. For enterprise buyers evaluating Claude Code vs Codex CLI, this raises supply-chain security questions.

---

## Notable Commands / Code

```typescript
// Silent model degradation (from leaked source)
// After 3 consecutive 529 errors, falls back to sonnet
// No user notification
if (consecutiveErrors >= 3) {
  model = "claude-sonnet-4"; // was claude-opus-4
}
```

```typescript
// REPL.tsx — 5,005 lines, single component
// Uses React for terminal rendering
// Constant re-renders on every keystroke
```

---

## Personal Notes

This is a landmark moment for the agentic coding tools space. The leak confirms several things Daniel has been writing about:

- The importance of engineering discipline in agentic tools (Ch23 testing-evaluation)
- The trust implications of silent model switching (Ch11 model-selection)
- The emergence of always-on persistent agents (Ch32 agentic-pod / KAIROS)
- The competitive landscape shifting rapidly (Ch05 competing-tools)

The KAIROS discovery is particularly noteworthy — it validates the "persistent agent" pattern that the Knowledge Flywheel v2 architecture is built around.

---

## Citations

[^1]: Chaofan Shou, security researcher, discovered the sourcemap exposure on March 31, 2026
[^2]: [DEV Community analysis](https://dev.to/kolkov/we-reverse-engineered-12-versions-of-claude-code-then-it-leaked-its-own-source-code-pij) — "We Reverse-Engineered 12 Versions of Claude Code"
[^3]: [Axios](https://www.axios.com/2026/03/31/anthropic-leaked-source-code-ai) — "Anthropic leaked its own Claude source code"
[^4]: [CNBC](https://www.cnbc.com/2026/03/31/anthropic-leak-claude-code-internal-source.html) — Claude Code run-rate revenue >$2.5B as of February 2026
[^5]: [Fortune](https://fortune.com/2026/03/31/anthropic-source-code-claude-code-data-leak-second-security-lapse-days-after-accidentally-revealing-mythos/) — "Second major security breach"
