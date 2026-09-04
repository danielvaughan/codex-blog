---
title: "Codex CLI for MongoDB Development: MCP Server, Agent Skills, and Document Modelling Workflows"
type: Technical Article
timestamp: 2026-05-26T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-26-codex-cli-mongodb-development-mcp-server-agent-skills-document-modelling-workflows"
tags: ["codex-cli", "mongodb", "mcp-server", "agent-skills", "document-database", "atlas", "vector-search", "schema-design"]
date: 2026-05-26T09:00:00+00:00
last_modified_at: 2026-09-04T08:18:49+01:00
---
# Codex CLI for MongoDB Development: MCP Server, Agent Skills, and Document Modelling Workflows


---

The MongoDB MCP Server reached general availability at v1.9.0 in March 2026 and has since grown to v1.11.0 with 41+ tools across six categories [^1][^2]. Combined with the MongoDB Agent Skills package — seven skills encoding schema design heuristics, indexing strategies, and query patterns — it gives Codex CLI the deepest document-database integration of any coding agent [^3]. This article covers the server, the skills, config.toml wiring, AGENTS.md conventions, and four production workflow patterns.

## The MongoDB MCP Server

The official server (`mongodb-js/mongodb-mcp-server`) ships as an npm package and Docker image, supporting both STDIO and HTTP transports [^1]. At 41+ tools it is roughly double the surface area of the Neon or Supabase MCP servers [^2].

### Tool Categories

| Category | Tool Count | Examples |
|---|---|---|
| Database Operations | 21 | `find`, `aggregate`, `insert-many`, `update-many`, `delete-many`, `create-collection`, `create-index`, `collection-schema`, `explain`, `export` |
| Atlas Cluster Management | 13 | `atlas-list-clusters`, `atlas-inspect-cluster`, `atlas-create-free-cluster`, `atlas-create-db-user`, `atlas-create-access-list` |
| Atlas Stream Processing | 4 | `atlas-streams-build`, `atlas-streams-discover`, `atlas-streams-manage`, `atlas-streams-teardown` |
| Atlas Local Deployments | 4 | `atlas-local-create-deployment`, `atlas-local-list-deployments`, `atlas-local-connect-deployment`, `atlas-local-delete-deployment` |
| Performance Advisory | 4 | `listClusterSuggestedIndexes`, `listDropIndexes`, `listSchemaAdvice`, `listSlowQueries` |
| Knowledge Search | 2 | `list-knowledge-sources`, `search-knowledge` |

The Performance Advisory tools surface Atlas recommendations directly inside the agent loop — no context switching to the Atlas console [^4]. The `insert-many` tool now generates embeddings automatically via Voyage AI when the target collection has a vector search index [^4].

### Configuration

```toml
# ~/.codex/config.toml  —  or  .codex/config.toml (project-scoped)

[mcp_servers.mongodb]
command = "npx"
args = ["-y", "mongodb-mcp-server@1"]

[mcp_servers.mongodb.env]
MDB_MCP_CONNECTION_STRING = "mongodb+srv://dev:${MONGO_PASSWORD}@cluster0.example.mongodb.net/myapp"
MDB_MCP_READ_ONLY = "true"
```

For Atlas API access (cluster management, performance advisory), add service account credentials:

```toml
[mcp_servers.mongodb.env]
MDB_MCP_API_CLIENT_ID = "${ATLAS_CLIENT_ID}"
MDB_MCP_API_CLIENT_SECRET = "${ATLAS_CLIENT_SECRET}"
```

Key environment variables [^1]:

| Variable | Purpose |
|---|---|
| `MDB_MCP_CONNECTION_STRING` | MongoDB connection URI |
| `MDB_MCP_READ_ONLY` | Restrict to read-only operations |
| `MDB_MCP_MAX_TIME_MS` | Query timeout in milliseconds |
| `MDB_MCP_IDLE_TIMEOUT_MS` | Connection idle timeout |
| `MDB_MCP_API_CLIENT_ID` | Atlas API service account ID |
| `MDB_MCP_API_CLIENT_SECRET` | Atlas API service account secret |

Use `--readOnly` or `MDB_MCP_READ_ONLY=true` in any environment where writes are unacceptable. The server also supports `--disableTools` to restrict the tool surface [^1].

### Transport Options

STDIO is the default and integrates directly with Codex CLI's config.toml. For shared or remote setups, the server supports HTTP transport:

```bash
npx mongodb-mcp-server@1 --transport http --httpHost 127.0.0.1 --httpPort 3000
```

Docker deployment is also available via `mongodb/mongodb-mcp-server` on Docker Hub [^1].

## MongoDB Agent Skills

The Agent Skills package (`mongodb/agent-skills`, v1.1.0) bundles seven skills that embed MongoDB engineering standards into the agent's context [^3][^5]. Skills cover:

- **Schema design** — document modelling heuristics, embedding vs. referencing decisions, validation rules
- **Indexing strategies** — compound index selection, covered queries, partial indexes, TTL indexes
- **Query patterns** — aggregation pipeline construction, `$lookup` optimisation, natural language to MongoDB query translation
- **Vector search** — index creation, embedding generation, semantic search pipelines
- **Stream processing** — Atlas Stream Processing pipeline design with Kafka, S3, and Lambda integrations
- **Operational safeguards** — production readiness checks, connection pooling, write concern selection

### Installation in Codex CLI

```bash
codex plugin marketplace add mongodb/agent-skills
```

This installs both the MCP server and the skills as a single plugin [^5]. Alternatively, clone the repository and point Codex at the skills directory:

```bash
git clone https://github.com/mongodb/agent-skills.git
```

The skills are also available as plugins for Claude Code, Cursor, Gemini CLI, and VS Code [^3].

## AGENTS.md for MongoDB Projects

An `AGENTS.md` file at repository root encodes project conventions that survive context compaction [^6]. For a MongoDB 8.x project:

```markdown
# AGENTS.md — MongoDB Project Conventions

## Stack
- MongoDB 8.3 (document database)
- Node.js 22 / TypeScript 5.8
- Mongoose 9.x ODM

## Schema Design Rules
- Prefer embedding over referencing when subdocuments are read together
- Never embed arrays expected to grow unboundedly — use the Bucket pattern
- All collections MUST have a JSON Schema validator in production
- Use `$$ROOT` sparingly in aggregation — prefer `$project` stages

## Indexing Rules
- Every query pattern MUST have a supporting index
- Compound indexes: ESR rule (Equality, Sort, Range)
- Never create single-field indexes that duplicate a compound index prefix
- TTL indexes for session/token collections — never application-level deletion

## Anti-Hallucination Rules
- MongoDB 8.3 is current stable, NOT 7.0 or 6.0
- `$lookup` supports `let`/`pipeline` syntax since 3.6 — always use pipeline form
- `$merge` replaces `$out` for incremental materialised views
- Atlas Vector Search indexes use `vectorSearch` type, NOT `search` type
- Mongoose 9.x uses ESM by default — no `require()` calls

## Testing
- Use mongodb-memory-server for unit tests
- Integration tests against Atlas local deployment via MCP
```

## Workflow Patterns

### Pattern 1: Schema Design with Agent Skills

```mermaid
flowchart TD
    A[Describe domain model] --> B[Agent loads schema design skill]
    B --> C[Agent proposes document schema]
    C --> D[create-collection with validator]
    D --> E[insert-many with sample documents]
    E --> F[collection-schema — verify inferred shape]
    F --> G{Schema matches intent?}
    G -- No --> C
    G -- Yes --> H[create-index for query patterns]
    H --> I[explain — verify index usage]
```

Prompt: *"Design a MongoDB schema for an e-commerce order system. Orders have line items, shipping addresses, and payment records. Use the Bucket pattern for order history."*

The agent, guided by the schema design skill, will prefer embedding line items and shipping addresses within the order document whilst bucketing historical orders into fixed-size documents. The `collection-schema` tool then validates that the inferred schema matches the proposed structure [^2].

### Pattern 2: Performance Audit with Atlas Advisory

```mermaid
flowchart TD
    A[codex 'Audit performance for orders collection'] --> B[listSlowQueries]
    B --> C[Agent analyses slow query patterns]
    C --> D[listClusterSuggestedIndexes]
    D --> E[Agent evaluates suggestions against existing indexes]
    E --> F[listDropIndexes — identify unused indexes]
    F --> G[Agent proposes index changes]
    G --> H[create-index / drop-index]
    H --> I[explain — verify improvement]
```

This loop replaces the manual cycle of checking Atlas console → analysing → switching to shell → creating indexes. The Performance Advisory tools surface the same data the Atlas UI shows, but inside the agent context [^4].

### Pattern 3: Vector Search Pipeline

MongoDB 8.x supports native vector search with indexes that build up to 30× faster than MongoDB 7.0 [^7]. The MCP server's `insert-many` tool with automatic embedding generation via Voyage AI eliminates the pre-processing step:

```bash
codex "Create a vector search index on the articles collection for the \
content_embedding field with 1536 dimensions and cosine similarity. \
Then insert these 50 articles with automatic embedding generation."
```

The agent will:
1. Use `create-index` with vector search type and the specified dimensions
2. Call `insert-many` — the server automatically generates embeddings for fields mapped to vector search indexes [^4]
3. Run a test `aggregate` with `$vectorSearch` to verify retrieval quality

### Pattern 4: Batch Collection Audit with `codex exec`

```bash
find . -name '*.model.ts' | codex exec \
  "Review this Mongoose model file. Use the MongoDB MCP server to check \
   collection-schema and collection-indexes for the corresponding collection. \
   Report: missing indexes for declared query patterns, unbounded array \
   embeddings, missing validators, and TTL candidates."
```

This pipes every model file through the agent, which cross-references the declared schema against the live database state via MCP tools [^8].

## Model Selection

| Task | Recommended Model | Rationale |
|---|---|---|
| Schema design, aggregation pipelines | `o3` | Strongest reasoning for complex document modelling decisions |
| Index creation, CRUD operations | `o4-mini` | Fast, cost-effective for straightforward database operations |
| Batch audits with `codex exec` | `o4-mini` | Lower cost per file across large model directories |
| Vector search pipeline design | `o3` | Multi-step reasoning for embedding + retrieval workflows |

## Sandbox and Security Considerations

- **Network access**: The MongoDB MCP server requires network connectivity to reach Atlas clusters. Use `full` network mode in Codex CLI, or configure the sandbox to allow MongoDB connection strings [^9]
- **Read-only mode**: Always set `MDB_MCP_READ_ONLY=true` for audit and exploration tasks. Use read-write only when the prompt explicitly requires mutations [^1]
- **Credential hygiene**: Use environment variable references (`${MONGO_PASSWORD}`) in config.toml rather than literal credentials. Atlas API credentials should use service accounts with minimum required permissions [^1]
- **Elicitation safety**: The GA server supports elicitation for destructive operations — the agent will confirm before dropping collections or databases [^2]
- **Local development**: Use `atlas-local-create-deployment` for sandboxed local clusters that require no Atlas credentials, reducing credential exposure during development [^4]

## Server Composition

The MongoDB MCP server covers database operations comprehensively, but production workflows often benefit from composing it with other servers:

```toml
[mcp_servers.mongodb]
command = "npx"
args = ["-y", "mongodb-mcp-server@1"]

[mcp_servers.mongodb.env]
MDB_MCP_CONNECTION_STRING = "mongodb+srv://dev:${MONGO_PASSWORD}@cluster0.example.mongodb.net/myapp"

[mcp_servers.github]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[mcp_servers.github.env]
GITHUB_PERSONAL_ACCESS_TOKEN = "${GITHUB_TOKEN}"
```

This gives the agent database context (MongoDB), repository context (GitHub), and project conventions (AGENTS.md) — the three layers needed for end-to-end feature development.

## Limitations

- **Training data lag**: Models may suggest MongoDB 6.0/7.0 patterns. The AGENTS.md anti-hallucination rules and agent skills mitigate this, but review generated aggregation pipelines for deprecated operators [^6]
- **Tool budget**: 41+ tools consume significant context. Use `--disableTools` to restrict to relevant categories (e.g., disable Stream Processing tools for a CRUD-only project) [^1]
- **No change streams via MCP**: The server provides Stream Processing tools for Atlas Streams but does not expose `watch()` for real-time change streams on collections ⚠️
- **Vector search maturity**: Automatic embedding generation requires Voyage AI integration and is behind a feature flag for vector search index creation [^4]
- **Atlas dependency for advanced features**: Performance Advisory, Stream Processing, and Knowledge Search tools require Atlas connectivity — they are unavailable with standalone/self-hosted MongoDB [^2]
- **Community alternative**: MongoDB Lens (200+ stars, 50+ tools) offers a community-maintained alternative with broader tool coverage but lacks the official Agent Skills integration [^2]

## Citations

[^1]: [MongoDB MCP Server — GitHub](https://github.com/mongodb-js/mongodb-mcp-server) — Official repository, configuration reference, v1.11.0 (May 2026)
[^2]: [The MongoDB MCP Server — ChatForest Review](https://chatforest.com/reviews/mongodb-mcp-server/) — Comprehensive review, 41+ tools, 4/5 rating (April 2026)
[^3]: [Introducing MongoDB Agent Skills and Plugins — MongoDB Blog](https://www.mongodb.com/company/blog/product-release-announcements/introducing-mongodb-agent-skills) — Agent Skills GA announcement, seven skills, plugin distribution (March 2026)
[^4]: [What's New in the MongoDB MCP Server: Winter 2026 Edition — MongoDB Blog](https://www.mongodb.com/company/blog/product-release-announcements/whats-new-mongodb-mcp-server-winter-2026-edition) — Performance Advisory, vector search, local clusters, automatic embeddings
[^5]: [MongoDB Agent Skills — GitHub](https://github.com/mongodb/agent-skills) — Skills repository, v1.1.0, installation instructions, supported clients
[^6]: [AGENTS.md Guide — OpenAI Codex Docs](https://developers.openai.com/codex/agents-md) — Project conventions, anti-hallucination rules, context compaction survival
[^7]: [MongoDB 8.0 Release Notes — MongoDB Docs](https://www.mongodb.com/docs/manual/release-notes/8.0/) — Vector index performance, bulkWrite, 36% read throughput improvement
[^8]: [Codex CLI `exec` — OpenAI Codex Docs](https://developers.openai.com/codex/cli) — Batch execution, stdin piping, structured output
[^9]: [Codex CLI Features — OpenAI Codex Docs](https://developers.openai.com/codex/features) — Sandbox modes, network access configuration, approval modes
