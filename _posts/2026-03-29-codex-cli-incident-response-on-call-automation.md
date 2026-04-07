---
title: "Codex CLI Incident Response: Automating On-Call with Agents"
description: "Wire PagerDuty/Datadog alerts into Codex CLI agentic workflows for AI-driven incident investigation and patch generation."
tags: [incident-response, pagerduty, datadog, on-call-automation, production-safety, mcp]
---

![Sketchnote: Codex CLI Incident Response: Automating On-Call with Agents](/sketchnotes/articles/2026-03-29-codex-cli-incident-response-on-call-automation.png)


*Published: 2026-03-29. Primary sources: [Datadog + OpenAI Codex CLI integration blog](https://www.datadoghq.com/blog/openai-datadog-ai-devops-agent/), [Datadog MCP Server docs](https://docs.datadoghq.com/bits_ai/mcp_server/), [Codex Advanced Config](https://developers.openai.com/codex/config-advanced).*

---

Incident response is one of the most compelling — and underexplored — use cases for Codex CLI. When your service is degraded at 3 a.m., the tools that help fastest are the ones already in your terminal. Codex CLI, connected to your observability stack via MCP, can move from "alert fired" to "patch proposed" without a human touching a keyboard.

This article covers the patterns that make this work in practice: how to connect observability tools, how to wire webhooks for trigger-based invocation, and what guardrails to put in place when agents act on production systems.

---

## The Core Pattern: Alert → Agent → Investigation → Draft Fix

The full incident response loop:

```
1. Alert fires (PagerDuty / Opsgenie / Datadog monitor)
2. Webhook triggers codex exec with incident context
3. Codex calls MCP tools to gather context:
   - Logs from the past 30 minutes (Datadog MCP)
   - APM traces for the affected service
   - Recent deploys (GitHub MCP)
   - Monitor status
4. Agent produces root cause hypothesis
5. Codex drafts a patch (configuration change, code fix, or rollback)
6. Inline diff shown to on-call engineer for approval before apply
7. Post-fix: metrics verified via MCP, incident updated, PR opened
```

The key insight: Codex doesn't replace the on-call engineer. It handles the *context gathering* phase — which is typically 60–80% of incident response time — so the human arrives at a decision point rather than a blank terminal.

---

## Setting Up the Datadog MCP Integration

The Datadog MCP Server launched **GA on March 10, 2026**. It connects Codex (and any MCP-compatible agent) to your full Datadog observability stack.

**Add to `~/.codex/config.toml`:**

```toml
[mcp.servers.datadog]
type = "streamable-http"
url = "https://mcp.datadoghq.com/mcp"
# Add toolsets as query params — only load what you need
# Available: core, alerting, apm, cases, synthetics, workflows
url = "https://mcp.datadoghq.com/mcp?toolsets=core,apm,alerting"

[mcp.servers.datadog.auth]
type = "oauth"
# First run opens browser for OAuth flow — credentials cached
```

**What `core` toolset gives you:**
- Query logs (natural language → Datadog log search)
- Fetch metrics for any service
- Access APM traces
- Update active incidents
- Check monitor statuses
- List dashboards, notebooks, events

**Audit trail:** Every tool call is recorded in Datadog Audit Trail with tool name, arguments, user identity, and the MCP client used. The metrics `datadog.mcp.session.starts` and `datadog.mcp.tool.calls` (tagged with `tool_name`, `user_id`, `client`) let you monitor MCP usage.

---

## Wiring PagerDuty as a Trigger

Codex CLI isn't natively event-driven — it needs an external process to kick off `codex exec` on alert. Two patterns:

### Pattern A: PagerDuty Webhook → Lambda → codex exec

```python
# lambda/on_alert.py
import subprocess, json, os

def handler(event, context):
    alert = json.loads(event["body"])
    service = alert["service"]["name"]
    incident_id = alert["id"]

    # Call codex exec with incident context
    result = subprocess.run([
        "codex", "exec",
        "--profile", "incident-responder",
        "--no-interactive",
        f"Investigate a production incident for service '{service}' (ID: {incident_id}). "
        f"Query Datadog for errors and high-latency in the last 30 minutes. "
        f"Check APM traces. Identify root cause and draft a fix. "
        f"Do not apply any changes — output a diff and summary only."
    ], capture_output=True, text=True, timeout=300)

    # Post summary to PagerDuty incident notes
    post_to_pagerduty(incident_id, result.stdout)
```

### Pattern B: `codex notify` → PagerDuty Completion Alert

The `notify` key fires when Codex completes a turn. Use it to post a completion webhook after Codex has finished an investigation:

```toml
# ~/.codex/config.toml — incident responder profile
[profiles.incident-responder]
model = "gpt-5-codex"
reasoning_effort = "high"
approval_policy = "on-failure"  # Only ask for approval on destructive actions

notify = [
  "bash", "-lc",
  "curl -s -X POST https://events.pagerduty.com/v2/enqueue -H 'Content-Type: application/json' -d '{\"routing_key\": \"$PD_INTEGRATION_KEY\", \"event_action\": \"resolve\", \"payload\": {\"summary\": \"Codex investigation complete\", \"severity\": \"info\", \"source\": \"codex-cli\"}}'"
]
```

---

## The `incident-responder` Profile

Define a dedicated profile with appropriate guardrails for production work:

```toml
# .codex/config.toml (project-level, for ops repos)

[profiles.incident-responder]
model = "gpt-5-codex"
reasoning_effort = "high"

# Safe approval policy: approve reads, ask for writes
approval_policy = "on-request"
approvals_reviewer = "guardian_subagent"  # Smart Approvals for reads; human for destructive ops

# MCP servers available in this profile
[profiles.incident-responder.mcp]
allowed_servers = ["datadog", "github"]

# Agent instructions scoped to incident work
developer_instructions = """
You are an incident response agent. Your job is to investigate production issues quickly and accurately.

Rules:
- ALWAYS gather context before proposing solutions
- NEVER restart services, scale infrastructure, or modify production configs without explicit approval
- ALWAYS show your working: quote the specific logs, traces, or metrics that support your hypothesis
- Output a structured report: (1) Timeline, (2) Root cause hypothesis, (3) Evidence, (4) Recommended fix
- Draft fixes as diffs — never apply them directly unless told to
- When in doubt, request human approval
"""
```

---

## AGENTS.md for Incident Response Repos

If your ops/infrastructure repo has a top-level AGENTS.md, tune it for incident work:

```markdown
# AGENTS.md — Ops Repository

## Incident Response Guidelines

When investigating an incident:
1. Check Datadog logs for the past 30 minutes FIRST — use natural language queries
2. Cross-reference with recent deploys: `git log --oneline -20` and GitHub deploys API
3. Check for open incidents in Datadog before creating new ones
4. Always output a structured report (see format below)

## Forbidden Actions

DO NOT:
- Restart Kubernetes pods or services
- Modify production database records
- Scale up/down infrastructure
- Merge PRs to main

All of the above require a `[HUMAN APPROVAL REQUIRED]` marker in your output.

## Output Format for Incident Investigation

```
## Incident Summary
**Service:** [name]
**Time range:** [start] – [end]
**Symptoms:** [what users/monitors observed]

## Root Cause
[1-paragraph hypothesis supported by evidence]

## Evidence
- Log sample: `[paste relevant log line]`
- APM trace: [trace ID or summary]
- Deploy correlation: [yes/no, details]

## Recommended Fix
[diff or config change, marked as DRAFT — human must approve]
```
```

---

## Guardrails for Production Safety

These patterns prevent agent actions that could worsen an incident:

### 1. Approval policy: reads free, writes gated

```toml
[profiles.incident-responder]
approval_policy = "on-request"
# MCP reads (logs, metrics, traces) don't trigger approval
# MCP writes (incident updates) and shell commands do
```

### 2. Network sandbox: allow observability, block broad internet

```toml
[profiles.incident-responder.sandbox]
network = "allowed-domains"
allowed_domains = [
  "mcp.datadoghq.com",
  "api.github.com",
  "api.pagerduty.com"
]
```

### 3. Hooks for audit logging

```toml
# Log every command Codex runs during incident response
[[hooks]]
event = "PreToolUse"
run = ["bash", "-lc", "echo \"[$(date -u +%Y-%m-%dT%H:%M:%SZ)] CODEX TOOL: $CODEX_TOOL_NAME $CODEX_TOOL_ARGS\" >> /var/log/codex-incident.log"]
```

### 4. Read-only mode for initial investigation

Start every incident investigation in a read-only profile:

```toml
[profiles.incident-investigate]
approval_policy = "always"  # Approve everything — forces Codex to propose, not act
model = "gpt-5-codex"
```

Switch to `incident-responder` (with Smart Approvals) only once you've confirmed the root cause and want Codex to draft a fix.

---

## The Datadog → Codex Workflow in Practice

Here's what a real investigation session looks like. You can run this interactively or via `codex exec`:

```
$ codex --profile incident-responder
> Service X is showing 40% error rate since 14:23 UTC.
  Investigate using Datadog. What changed?

[Codex calls Datadog MCP tools]
→ get_logs: "service:x status:error" last 30min → 1,847 errors, all NullPointerException in UserController.java:L234
→ get_apm_traces: service:x → P99 latency 8,200ms (up from 240ms baseline)
→ get_events: service:x last 2hr → deploy at 14:18 UTC (v2.4.1 → v2.4.2)

Hypothesis: Deploy v2.4.2 introduced a null reference in UserController.java:L234.
The APM trace shows the error originates in the `getUserPreferences()` method.

Recommended fix: Roll back to v2.4.1 OR apply this patch:
[diff shown inline]

Risk: LOW — isolated to UserController, no DB migrations in v2.4.2.
```

The terminal also shows inline ASCII metric plots (configurable by Datadog's MCP client) so you can see the error rate graph without leaving the terminal.

---

## When to Use Codex for Incidents vs When Not To

| Situation | Recommendation |
|-----------|---------------|
| Unknown root cause, large log volume | ✅ Use Codex — exactly what it's for |
| Suspected regression (narrow diff) | ✅ Use Codex to confirm hypothesis quickly |
| Database performance issue | ⚠️ Use Codex for diagnostics only; DB changes need DBA review |
| Active security incident | ❌ Do not use Codex — potential for log injection / prompt injection |
| Infrastructure scale-up needed | ❌ Too high-risk for agent action; manual only |
| Post-incident review / write-up | ✅ Excellent use case — Codex reads the incident timeline and drafts the postmortem |

---

## Post-Incident: Auto-Draft Postmortems

Once an incident is resolved, Codex can draft the postmortem doc from the investigation log:

```bash
codex exec --profile incident-responder \
  "Read the incident log at /var/log/codex-incident.log and the Datadog incident #INC-1234.
   Draft a postmortem in docs/postmortems/$(date +%Y-%m-%d)-service-x.md using our standard template."
```

---

## Key Resources

- [Datadog MCP Server](https://docs.datadoghq.com/bits_ai/mcp_server/) — GA March 10, 2026
- [Datadog + Codex CLI blog post](https://www.datadoghq.com/blog/openai-datadog-ai-devops-agent/)
- [Codex Advanced Config — `notify` hook](https://developers.openai.com/codex/config-advanced)
- [Codex config reference — profiles](https://developers.openai.com/codex/config-reference)
