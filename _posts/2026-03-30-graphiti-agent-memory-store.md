---
title: "Graphiti: Temporal Knowledge Graphs for Agent Memory — Should the Knowledge Flywheel Use It?"
layout: single
date: 2026-03-30
tags: memory, knowledge-graph, neo4j, agents, graphiti
---

## What Is Graphiti?

Graphiti is an open-source Python library (Apache 2.0 licensed, maintained by Zep) for building and querying temporal knowledge graphs designed specifically for AI agent memory. Unlike flat vector stores or static RAG pipelines, Graphiti models facts as entities and relationships with explicit time-bounded validity windows — it knows not just *what* is true, but *when* it became true and when it was superseded. This makes it well-suited to any agent workflow where the world changes and agents need to reason about the current state without hallucinating stale facts.

Graphiti is the open-source engine under Zep Cloud, a managed memory platform. You can use Graphiti directly against your own Neo4j (or FalkorDB, Kuzu) instance without subscribing to Zep Cloud.

## How It Works

### Memory Model

Graphiti structures memory into four layers:

- **Episodes** — raw ingested data (text or JSON), the ground truth. Every derived fact traces back to a source episode.
- **Entities (nodes)** — people, concepts, tools, configurations. Summaries evolve as new episodes arrive.
- **Facts/Relationships (edges)** — typed triplets linking entities, each carrying a temporal validity window (`valid_from`, `valid_until`).
- **Custom Types (ontology)** — developer-defined entity and edge types via Pydantic models, allowing domain-specific schemas.

When an agent session completes, a summary or structured observation is ingested as an episode. Graphiti extracts entities and edges via LLM-powered extraction (requires OpenAI, Gemini, or Anthropic API access), invalidates any facts that contradict new evidence, and stores everything in the graph database. On the next run, the agent queries Graphiti using hybrid retrieval — combining semantic vector search, BM25 keyword search, and graph traversal — to reconstruct relevant context without reading raw conversation history.

### Query Patterns

Agents query Graphiti using natural-language questions. The library returns ranked facts, entity summaries, and temporal context. Because retrieval is graph-traversal + vector hybrid rather than pure embedding similarity, it handles multi-hop questions well: "what did the pipeline process last week, and what was approved versus rejected?"

### Backend Requirements

Graphiti's primary and default backend is **Neo4j version 5.26 or higher**. Neo4j AuraDB is explicitly supported — it uses the `neo4j+s://` protocol and includes APOC procedures automatically, so no additional configuration is required. The connection is three environment variables: `NEO4J_URI`, `NEO4J_USER`, `NEO4J_PASSWORD`.

Other supported backends: FalkorDB, Kuzu, Amazon Neptune + OpenSearch.

### Ingestion Cost

LLM-powered entity extraction is the key tradeoff. Every episode ingestion makes LLM calls to extract entities and edges. Benchmarks from the Mem0 research paper suggest Graphiti's memory footprint can reach 600,000+ tokens per conversation compared to Mem0's 1,764 tokens — though Zep disputes the test configuration. For batch pipelines that ingest infrequently (daily or weekly), this is manageable. For high-frequency, low-latency pipelines it adds real cost.

### Performance

P95 retrieval latency is approximately 300ms. For the Knowledge Flywheel's batch-oriented, scheduled workload this is not a concern.

---

## Fit Assessment for the Knowledge Flywheel v2

The Knowledge Flywheel v2 already has a Neo4j AuraDB instance as its knowledge graph. The question is whether adding Graphiti on top of that Neo4j instance provides value that the system does not already have.

### What the System Already Has

- **Neo4j AuraDB** with a hand-authored knowledge graph of atomic notes (concepts, how-to, reference, community, changelog).
- **git as the durable backup** — the `knowledge/` directory is version-controlled.
- **Agent invocations via Codex CLI** — each run is stateless (a Cloud Run job spins up, clones the repo, runs Codex, commits output, exits).
- **No persistent episodic agent memory** — each agent run starts from the AGENTS.md, the repo state, and whatever context the Codex prompt supplies. There is no layer that remembers which URLs were already processed, which decisions were made in previous runs, or why a human reviewer rejected a particular draft.

### The Gap Graphiti Fills

The system's current weakness is agent amnesia. Each Codex CLI run is ephemeral. If the research job discovers a URL on Monday and processes it, and on Tuesday the same job runs again, it has no mechanism to remember that URL was already processed — other than writing a record back to git or Neo4j manually. Human decisions (approved vs. rejected in the oversight dashboard) are not automatically fed back to influence future agent behaviour.

Graphiti would give the Knowledge Flywheel a persistent, queryable episodic memory layer on top of the existing Neo4j AuraDB instance. Because Graphiti stores its data in Neo4j, there is no new database to manage — it writes entities and episodes into the same AuraDB instance the pipeline already uses (or a separate database within the same AuraDB cluster).

### Specific Use Cases Where Graphiti Would Add Value

**1. Deduplication across runs**
The research job, YouTube curator job, and sentiment job all discover and process URLs. Currently, deduplication requires explicit checks against Neo4j. With Graphiti, ingesting each processed URL as an episode (with metadata: source, date, outcome) would let agents ask: "Has this URL been processed? What was the result?" — and get a structured answer including temporal context.

**2. Decision memory**
The human oversight dashboard approves or rejects agent-generated content. Those decisions are currently not surfaced back to the agents in subsequent runs. Graphiti could store each approval/rejection as a typed episode with entities (article, topic, reason), so a future agent could query: "What kinds of content has the human reviewer rejected recently?" and adjust its output accordingly.

**3. Cross-run task continuity**
If a job fails mid-run or a chapter is partially processed, the next run has no awareness of partial state beyond what was committed to git. Graphiti episodes could track task state: "kb-update-job run 2026-03-29 processed notes 1–12 of 34 before timeout." The next run resumes at note 13.

**4. Agent self-awareness of prior quality**
The analytics job generates weekly performance data. If that data were ingested as Graphiti episodes, the newsletter and manuscript jobs could query: "What topics had the highest reader engagement last month?" and prioritise accordingly — without baking engagement logic into the agent prompts manually.

**5. Audit trail**
Graphiti's provenance model (every fact traces to a source episode) provides a natural audit log: "Why does the agent believe X?" traces to the specific run and source that established that belief.

---

## Integration Complexity

### What Would Change

Graphiti is a Python library. The Knowledge Flywheel's agents run as Node.js 22 containers using Codex CLI. This is the primary integration friction.

**Options:**

1. **Graphiti MCP Server** (lowest friction): Graphiti ships an official MCP (Model Context Protocol) server. Codex CLI already speaks MCP. Running Graphiti as an MCP server in a Cloud Run sidecar or a separate lightweight Cloud Run service would let Codex CLI agents call memory tools (`add_episode`, `search_memory`) over MCP without writing any Python. The Graphiti MCP server connects to the existing Neo4j AuraDB instance.

2. **Thin Python sidecar per job** (moderate friction): Each Cloud Run job container includes a small Python script that wraps Graphiti calls. The Codex CLI script calls it via shell exec. This keeps the agent in Node.js while using Graphiti's Python SDK.

3. **REST wrapper service** (highest friction, most robust): A dedicated Cloud Run service exposing a simple REST API over Graphiti. All jobs call it for memory reads and writes. Adds a network hop but centralises the memory layer cleanly.

### Schema Considerations

Graphiti writes its own graph schema into Neo4j (entities, edges, episodes). The existing Knowledge Flywheel schema uses custom node labels (Article, KnowledgeNote, Source, etc.). These do not conflict — Graphiti can use a separate Neo4j database within the same AuraDB instance (AuraDB supports multiple databases on Professional and Enterprise tiers), keeping schemas clean.

### What Does Not Change

- Neo4j AuraDB: already provisioned, no additional cost.
- Cloud Run jobs: no architectural changes required.
- Codex CLI: no changes if using MCP server approach.
- Pub/Sub event bus: Graphiti has no Pub/Sub dependency.

### Operational Overhead

If using the MCP server approach, one additional Cloud Run service is added (stateless, scales to zero, minimal cost). Graphiti's LLM calls for entity extraction add token cost per ingestion — at batch rates (one ingestion per job run per article/URL) this is small. No new managed databases.

---

## Comparison to Alternatives

| Approach | Fit for Flywheel | Notes |
|---|---|---|
| **Graphiti on Neo4j** | Strong | Reuses existing AuraDB, temporal reasoning, MCP integration available |
| **Mem0** | Moderate | Easier setup, Python SDK, but dual-store adds complexity and graph features cost $249/mo on cloud tier |
| **Raw Neo4j (manual)** | Partial | Already in place, but requires hand-authoring every memory schema and query; no temporal reasoning |
| **LangGraph Memory** | Weak | Tied to LangGraph framework; Flywheel uses Codex CLI, not LangGraph |
| **No memory layer** | Current state | Simple but agents are amnesiac; same work gets redone; human feedback not reused |

Graphiti's advantage over raw Neo4j is that the episodic extraction and temporal validity logic is done for you. The existing Neo4j knowledge graph stores curated facts about Codex CLI the *subject*. Graphiti would store facts about the *pipeline itself* — what ran, what was processed, what decisions were made. These are complementary uses of the same database.

---

## License and Maturity

- **License:** Apache 2.0 (Graphiti library). Zep Cloud is proprietary but not required.
- **GitHub:** github.com/getzep/graphiti — actively maintained, backed by Zep.
- **Maturity:** Production-grade as of 2025–2026. Used in CRM agents, compliance systems, and healthcare workflows. Peer-reviewed architecture (arXiv 2501.13956). Cited in the ICLR 2026 MemAgents Workshop.
- **LLM support:** OpenAI, Azure OpenAI, Google Gemini, Anthropic. The Flywheel's existing OpenAI key works immediately.

---

## Recommendation: Backlog — High Priority

Graphiti is a strong fit for the Knowledge Flywheel v2, but not a blocker for v2 launch. The system's core value comes from the knowledge graph and publishing pipeline. Graphiti enhances agent effectiveness; it does not enable the core use case.

**Adopt it after the pipeline is running**, targeting these specific wins in order of value:

1. **First**: Implement URL deduplication via Graphiti episodes (research job, YT curator job). This is the highest-frequency pain point.
2. **Second**: Feed human approval/rejection decisions from the oversight dashboard into Graphiti episodes. This closes the feedback loop between human oversight and agent behaviour.
3. **Third**: Implement cross-run task continuity for long-running jobs (kb-update-job, manuscript-job).

**Architecture recommendation**: Use the Graphiti MCP Server as a lightweight Cloud Run service connected to a separate database within the existing Neo4j AuraDB instance. This avoids Node.js/Python bridging, reuses existing infrastructure, and keeps integration minimal.

**Do not use Zep Cloud** — the Flywheel already has Neo4j AuraDB, and Graphiti self-hosted against it eliminates the need for Zep's managed service.

---

## Backlog and Article Ideas

- **Article**: "Why Your AI Pipeline Needs Episodic Memory (And How Graphiti Delivers It)" — explains the agent amnesia problem with the Flywheel as a concrete example.
- **Article**: "Graphiti MCP Server: Persistent Memory for Codex CLI Agents Without Writing Python" — practical integration walkthrough.
- **Backlog item**: Spike — deploy Graphiti MCP server against AuraDB, instrument the research job to log processed URLs as episodes, measure deduplication accuracy over two weeks.
- **Backlog item**: Extend oversight dashboard to write approval/rejection decisions to Graphiti as typed episodes.

---

## Sources

- [Graphiti README](https://raw.githubusercontent.com/getzep/graphiti/main/README.md)
- [Graphiti Neo4j Configuration Docs](https://help.getzep.com/graphiti/configuration/neo-4-j-configuration)
- [Top 6 AI Agent Memory Frameworks for Devs (2026)](https://dev.to/nebulagg/top-6-ai-agent-memory-frameworks-for-devs-2026-1fef)
- [Best AI Agent Memory Systems in 2026: 8 Frameworks Compared](https://vectorize.io/articles/best-ai-agent-memory-systems)
- [Mem0 vs Zep (Graphiti): AI Agent Memory Compared (2026)](https://vectorize.io/articles/mem0-vs-zep)
- [Graphiti MCP Server: Agentic Memory Guide](https://skywork.ai/skypage/en/graphiti-mcp-server-agentic-memory/1978662683507544064)
- [Implementing Agentic Memory with Graphiti — FalkorDB](https://www.falkordb.com/blog/implementing-agentic-memory-graphiti/)
- [Cognee AI Memory Tools Evaluation: Cognee, Mem0, Zep/Graphiti](https://www.cognee.ai/blog/deep-dives/ai-memory-tools-evaluation)
