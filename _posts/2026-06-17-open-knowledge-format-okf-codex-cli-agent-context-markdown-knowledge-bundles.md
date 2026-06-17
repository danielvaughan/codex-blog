---
title: "Open Knowledge Format and Codex CLI: Giving Your Agent a Knowledge Base It Can Actually Read"
description: "Google has published an open specification for packaging knowledge as markdown files with YAML frontmatter. It maps directly to patterns Codex CLI already supports — and it formalises what many teams have been doing informally with AGENTS.md, skills, and MCP servers."
date: 2026-06-17T09:00:00+00:00
last_modified_at: 2026-06-17T07:23:31+01:00
tags:
  - context-engineering
  - mcp
  - agents-md
  - codex-cli
  - ecosystem
---

![Sketchnote diagram for: Open Knowledge Format and Codex CLI: Giving Your Agent a Knowledge Base It Can Actually Read](/sketchnotes/articles/2026-06-17-open-knowledge-format-okf-codex-cli-agent-context-markdown-knowledge-bundles.png)

# Open Knowledge Format and Codex CLI: Giving Your Agent a Knowledge Base It Can Actually Read


---

There is a problem that every team running Codex CLI on a real codebase encounters within the first week. The agent can read your code. It can read your tests. It can even read your README. But it cannot read the knowledge that lives in your senior engineer's head — the business rules, the naming conventions, the reason that column is called `amt_gbp_net` and not `amount`, the fact that the payments service was rewritten in 2024 and half the old documentation is wrong.

That knowledge exists somewhere. It is scattered across Confluence pages, Slack threads, onboarding documents, and comments in pull requests that were merged eighteen months ago. None of it is in a format an agent can reliably consume.

Google has now published an answer to this: the Open Knowledge Format, or OKF.[^1] It is a specification — version 0.1, published 12 June 2026 — for representing knowledge as a directory of markdown files with YAML frontmatter. No proprietary SDK. No cloud account. No schema registry. If you can `cat` a file, you can read OKF. If you can `git clone` a repo, you can ship it.

The format is worth understanding because it maps directly to patterns Codex CLI already supports. More than that, many teams are already doing something very close to OKF without knowing it. This article walks through what OKF is, how it connects to Codex CLI's existing context mechanisms, and how to use the two together.

---

## What OKF Actually Is

OKF is a directory of markdown files. Each file represents a single "concept" — a table, an API endpoint, a metric, a business process, a playbook, anything that constitutes a unit of knowledge. Every file has YAML frontmatter with one required field: `type`.[^2]

Here is a minimal example:

```markdown
---
type: API Endpoint
title: Payment Processing
description: Handles card payments via Stripe integration.
resource: https://api.internal.acme.com/v2/payments
tags: [payments, stripe, pci]
timestamp: 2026-06-10T09:00:00Z
---

# Overview

The `/v2/payments` endpoint processes card payments through our
Stripe integration. It replaced the legacy `/v1/charge` endpoint
in March 2024.

## Important constraints

- All requests MUST include the `X-Idempotency-Key` header.
- Amounts are in minor units (pence, not pounds).
- The `currency` field defaults to `GBP` if omitted.

## Related concepts

- [Refunds](/endpoints/refunds.md) — reverse a completed payment
- [Customers](/entities/customers.md) — the payer record
```

The frontmatter is deliberately minimal. Only `type` is required. The recommended fields — `title`, `description`, `resource`, `tags`, `timestamp` — are optional. Producers can add any additional keys they like. Consumers must tolerate unknown fields gracefully.[^2]

A collection of these files is called a **knowledge bundle**. Bundles are organised as directory trees:

```
payments-knowledge/
├── index.md
├── endpoints/
│   ├── index.md
│   ├── payments.md
│   └── refunds.md
├── entities/
│   ├── index.md
│   └── customers.md
└── playbooks/
    ├── index.md
    └── pci-audit.md
```

Files link to each other using standard markdown links. An `index.md` at each directory level provides a listing for progressive disclosure — agents can read the top-level index first, then drill into specific directories as needed.

That is the entire format. No binary files, no custom parsers, no platform dependencies. A knowledge bundle is a git repository that happens to follow a small set of conventions.

---

## Three Design Principles

OKF is built on three principles that matter for how it interacts with Codex CLI.[^1]

**Minimally opinionated.** The spec prescribes almost nothing. Only the `type` field is required. Type values are not registered centrally — producers pick values that make sense for their domain. This means a team can start writing OKF documents tomorrow without waiting for anyone to define a taxonomy.

**Producer/consumer independence.** The team that writes the knowledge bundle does not need to know which agent will consume it. The team running Codex CLI does not need to know which tool produced the bundle. This decoupling is important in organisations where the domain experts and the developers using coding agents are different people.

**Format not platform.** OKF does not require a Google Cloud account, a specific database, or a proprietary SDK. It is files in a directory. This makes it compatible with any tool that can read markdown — which includes Codex CLI.

---

## How This Maps to Codex CLI

Codex CLI already has several mechanisms for providing context to the agent. OKF does not replace any of them. Instead, it provides a standardised format for the knowledge that feeds into them.

### AGENTS.md — the convention file pattern

Codex CLI reads `AGENTS.md` files from the repository root and subdirectories. These files contain instructions, constraints, and context that shape the agent's behaviour. OKF generalises this pattern: instead of one instruction file per directory, you have a structured knowledge base of interconnected concepts.

The two patterns complement each other. `AGENTS.md` tells the agent how to behave. An OKF bundle tells the agent what things mean.

A practical setup:

```
my-project/
├── AGENTS.md                    # Behavioural instructions
├── .codex/knowledge/            # OKF bundle
│   ├── index.md
│   ├── entities/
│   │   ├── customers.md
│   │   └── orders.md
│   ├── endpoints/
│   │   └── payments.md
│   └── playbooks/
│       └── deployment.md
└── src/
    └── ...
```

Reference the knowledge bundle from `AGENTS.md`:

```markdown
# AGENTS.md

## Domain knowledge

Before making changes to the payments module, read the knowledge
bundle in `.codex/knowledge/`. Start with `index.md` for an overview,
then consult the relevant concept documents.

Key references:
- `.codex/knowledge/endpoints/payments.md` — payment processing rules
- `.codex/knowledge/playbooks/deployment.md` — deployment procedure
```

The agent now has both behavioural guardrails (from `AGENTS.md`) and domain knowledge (from the OKF bundle) available in its context.

### Skills and the `@` mention system

Codex CLI 0.140.0 introduced unified `@` mentions for files, plugins, and skills. You can reference OKF documents directly in prompts:

```
@.codex/knowledge/endpoints/payments.md Refactor the payment handler
to use the idempotency pattern described in the knowledge base.
```

This pulls the OKF concept document into the agent's context alongside the code it needs to modify. The structured frontmatter (`type`, `tags`, `description`) helps the agent understand what it is reading without parsing free-form prose.

For teams using skills, an OKF bundle can serve as the knowledge backing for a custom skill:

```markdown
# .codex/skills/domain-expert.md

You are a domain expert for the payments platform. When answering
questions about business rules, consult the OKF knowledge bundle
in `.codex/knowledge/`.

Always check the `timestamp` field in the frontmatter — if a concept
document is older than 90 days, flag it as potentially stale.
```

### MCP — serving knowledge bundles to the agent

The most powerful integration point is MCP. An OKF bundle stored in a git repository can be exposed to Codex CLI through an MCP server, making the knowledge available as a searchable tool rather than a static file reference.

A simple MCP server for OKF might expose three tools:

1. **`search_knowledge`** — takes a query, searches concept titles, descriptions, and tags, returns matching documents
2. **`get_concept`** — takes a concept ID (e.g., `endpoints/payments`), returns the full document
3. **`list_concepts`** — takes an optional type filter, returns all concepts of that type

Configure it in `codex.toml`:

```toml
[mcp_servers.domain_knowledge]
command = "npx"
args = ["okf-mcp-server", "--bundle", ".codex/knowledge"]
```

With this setup, the agent can search the knowledge base dynamically rather than relying on static file references. When it encounters an unfamiliar table name or business rule, it calls `search_knowledge` to find the relevant concept document. The 0.140.0 release added MCP transient failure retries and encrypted OAuth credential storage, making MCP servers more reliable in production.[^3]

No official OKF MCP server exists yet — the specification is five days old. But the format is simple enough that building one is a weekend project. The concept ID is the file path minus `.md`; the search index is the frontmatter fields; the content is the markdown body. There is nothing exotic to parse.

### Hooks — validating knowledge references

Codex CLI's hook system can enforce knowledge bundle integrity. A `PreToolUse` hook can verify that any OKF concept referenced in generated code actually exists in the bundle:

```toml
[[hooks]]
event = "PreToolUse"
tool = "write_file"
command = "python3 .codex/hooks/validate-okf-refs.py $INPUT"
```

The validation script checks that markdown links to OKF concepts (`/entities/customers.md`, `/endpoints/payments.md`) resolve to actual files in the bundle. This catches stale references before they reach the codebase — the same pattern used for import validation, but applied to knowledge links.

---

## Building a Knowledge Bundle for Your Codebase

Starting an OKF bundle does not require a large upfront investment. The minimum viable bundle is three files.

### Step 1: Create the bundle directory

```bash
mkdir -p .codex/knowledge
```

### Step 2: Write the index

```markdown
---
type: Index
title: Project Knowledge Base
description: Domain knowledge for the payments platform.
okf_version: "0.1"
---

# Payments Platform Knowledge Base

This bundle contains domain knowledge for the payments platform,
organised by concept type.

- [Entities](entities/index.md) — core business objects
- [Endpoints](endpoints/index.md) — API reference and constraints
- [Playbooks](playbooks/index.md) — operational procedures
```

### Step 3: Document one concept properly

Pick the concept your team explains most often to new starters. Write it as an OKF document. Get the frontmatter right — `type`, `title`, `description`, `tags`. Write the body with structured markdown: headings, tables, code blocks. Link to related concepts even if those documents do not exist yet (you will fill them in later).

### Step 4: Reference it from AGENTS.md

Add a line to your `AGENTS.md` pointing the agent at the knowledge bundle. From that moment, every Codex CLI session in the repository has access to the knowledge.

### Step 5: Grow the bundle over time

Every time someone explains a business rule in a code review, writes a design decision in a pull request description, or answers a question in Slack that they have answered before — that is a candidate for an OKF concept document. The bundle grows organically, one concept at a time.

Google's own enrichment agent can accelerate this process. It walks a BigQuery dataset and drafts OKF documents automatically, using an LLM to generate descriptions, infer relationships, and crawl documentation URLs for supporting detail.[^1] The same approach works for any structured data source — database schemas, API specifications, configuration files.

---

## What OKF Means for Context Engineering

Context engineering — the discipline of assembling the right information for an agent at the right time — is solidifying as a core practice.[^4] OKF provides a missing piece: a standard interchange format for the knowledge layer.

Today, teams package context for Codex CLI in ad hoc ways: long `AGENTS.md` files, custom MCP servers that query internal wikis, skills that embed domain knowledge as prose. All of these work, but they are bespoke. When knowledge needs to move between teams, between tools, or between organisations, there is no shared format.

OKF offers that shared format. A knowledge bundle authored for Codex CLI can be consumed by Claude Code, Cursor, Google ADK, or any other agent framework that can read markdown. A bundle authored by a data team using Google's enrichment agent can be consumed by a development team using Codex CLI. The knowledge is decoupled from the tool.

This matters for the harness engineering thesis: the value is not in the model but in the infrastructure around it. OKF is pure infrastructure — a format specification, not an AI capability. It makes every agent that consumes it more effective, regardless of which model powers the agent.

---

## Practical Considerations

**Version control.** OKF bundles belong in git. They diff cleanly, support blame for attribution, and can be reviewed in pull requests alongside the code they describe. Store the bundle in the repository it documents, or in a dedicated knowledge repository that multiple projects reference.

**Staleness.** The `timestamp` field in frontmatter tracks when a concept was last updated. Build a simple CI check that flags concepts older than a threshold — 90 days, 180 days, whatever suits the domain. Stale knowledge is worse than missing knowledge because the agent trusts it.

**Size.** OKF documents should be concise. The spec recommends structured markdown — headings, tables, lists — over long-form prose. Agents parse structure more reliably than paragraphs. A concept document that exceeds a few hundred lines probably needs splitting into multiple concepts.

**Discoverability.** The `index.md` convention provides progressive disclosure. An agent reading the top-level index gets an overview; it can drill into subdirectories as needed. This is the same pattern as `AGENTS.md` files in subdirectories — give the agent a map, let it navigate.

**Security.** Knowledge bundles may contain sensitive domain knowledge. The same access controls that apply to the repository apply to the bundle. For bundles served via MCP, the 0.140.0 encrypted credential storage ensures that OAuth tokens for knowledge APIs are not stored in plaintext.[^3]

---

## Where This Goes Next

OKF is version 0.1 — a draft specification from Google, five days old at the time of writing. The GitHub repository has 3,200 stars and 206 forks already, which suggests the developer community recognises the need.[^5]

The specification explicitly references patterns that Codex CLI teams already use: convention files like `AGENTS.md`, hierarchical markdown knowledge bases, metadata-as-code repositories.[^2] It is not introducing a new idea so much as formalising an existing practice and giving it a name.

For Codex CLI users, the practical takeaway is straightforward. If you are already writing `AGENTS.md` files, you understand the pattern. OKF extends it from behavioural instructions to domain knowledge, with a minimal structure that makes the knowledge portable and machine-readable. Start with one concept document. Reference it from `AGENTS.md`. Build from there.

The knowledge your agent needs is already in your organisation. OKF gives it a format.

---

## References

[^1]: McVeety, S. and Hormati, A. (2026) 'Introducing the Open Knowledge Format', *Google Cloud Blog*, 12 June. Available at: https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing (Accessed: 17 June 2026).

[^2]: Google Cloud Platform (2026) 'Open Knowledge Format Specification v0.1', *GitHub*. Available at: https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md (Accessed: 17 June 2026).

[^3]: OpenAI (2026) 'Codex CLI v0.140.0 Release Notes', *GitHub*. Available at: https://github.com/openai/codex/releases/tag/v0.140.0 (Accessed: 16 June 2026).

[^4]: Willison, S. (2026) 'Context Engineering', *simonwillison.net*. Available at: https://simonwillison.net/2026/Jun/Context-Engineering/ (Accessed: 17 June 2026).

[^5]: Google Cloud Platform (2026) 'knowledge-catalog', *GitHub*. Available at: https://github.com/GoogleCloudPlatform/knowledge-catalog (Accessed: 17 June 2026).
