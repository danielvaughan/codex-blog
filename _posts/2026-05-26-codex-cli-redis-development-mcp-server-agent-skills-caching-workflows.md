---
title: "Codex CLI for Redis Development: MCP Server, Agent Skills, and Production Caching Workflows"
type: Technical Article
timestamp: 2026-05-26T00:00:00+00:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-05-26-codex-cli-redis-development-mcp-server-agent-skills-caching-workflows"
tags: ["codex-cli", "redis", "mcp", "caching", "agent-skills", "valkey", "data-structures"]
date: 2026-05-26T09:00:00+00:00
last_modified_at: 2026-09-05T10:23:50+01:00
---
# Codex CLI for Redis Development: MCP Server, Agent Skills, and Production Caching Workflows



---

Redis remains the dominant in-memory data store, with Redis 8.6 integrating JSON, time series, probabilistic structures, and vector sets directly into the core[^1]. The ecosystem now ships two official MCP servers — one for data operations and one for cloud subscription management — plus a dedicated agent skill that encodes current best practices across eight use cases[^2]. This article covers how to wire all three into Codex CLI, write an AGENTS.md that prevents the most common Redis anti-patterns, and run practical workflows for caching, rate limiting, and semantic search.

## The Redis MCP Landscape

Three integration points are available as of May 2026:

| Layer | Project | Transport | Purpose |
|-------|---------|-----------|---------|
| Data operations | `redis/mcp-redis` | STDIO | Read, write, query across all Redis data types |
| Cloud management | `redis/mcp-redis-cloud` | STDIO | Subscription management, database provisioning |
| Knowledge | `redis/agent-skills` | Skill file | Production patterns for caching, rate limiting, vector search |

### mcp-redis: The Data Operations Server

The official `redis-mcp-server` package exposes tools across every Redis data type[^3]:

- **Strings** — set/get with expiration for caching and configuration
- **Hashes** — field-value pairs including vector embeddings
- **Lists** — append/pop for queues and activity tracking
- **Sets** — membership and set operations (union, intersection, difference)
- **Sorted sets** — leaderboards, priority queues, sliding-window rate limiters
- **Pub/Sub** — publish/subscribe for real-time notifications
- **Streams** — event sourcing with consumer group support
- **JSON** — nested document storage and manipulation (Redis 8+)
- **Vector search** — create and query indexed vectors for RAG
- **Documentation search** — query Redis docs via HTTP API
- **Server management** — database information and diagnostics

### mcp-redis-cloud: Subscription Management

The companion server provides natural-language control over Redis Cloud subscriptions[^4]. You can create databases, query subscription metrics, and manage capacity — useful for infrastructure-as-code workflows where Codex provisions ephemeral Redis instances for testing.

### Redis Agent Skills

Redis published an official agent skill in February 2026 that loads on-demand when agents encounter Redis-related tasks[^2]. The skill covers eight domains: caching, rate limiting, session management, vector search, semantic caching, agent memory, pub/sub, and streams. Crucially, it encodes anti-pattern guardrails — no `KEYS` in loops, no unbounded key growth, no large values — that prevent the stale Redis 6 patterns most LLMs default to.

## Codex CLI Configuration

### config.toml for STDIO Servers

Add the Redis MCP server to `~/.codex/config.toml` or your project-scoped `.codex/config.toml`:

```toml
[mcp_servers.redis]
command = "uvx"
args = ["--from", "redis-mcp-server@latest", "redis-mcp-server", "--url", "redis://localhost:6379/0"]
env = {}
startup_timeout_sec = 15
tool_timeout_sec = 30
enabled = true
```

For Redis Cloud management, add the cloud server alongside:

```toml
[mcp_servers.redis-cloud]
command = "uvx"
args = ["--from", "redis-mcp-cloud@latest", "redis-mcp-cloud"]
env = { REDIS_CLOUD_API_KEY = "${REDIS_CLOUD_API_KEY}", REDIS_CLOUD_SECRET = "${REDIS_CLOUD_SECRET}" }
startup_timeout_sec = 15
tool_timeout_sec = 30
enabled = true
```

For authenticated Redis instances with TLS:

```toml
[mcp_servers.redis]
command = "uvx"
args = [
  "--from", "redis-mcp-server@latest", "redis-mcp-server",
  "--url", "rediss://default:password@redis-host:6380/0",
  "--ssl"
]
```

The `--cluster-mode` flag enables Redis Cluster support, and Azure EntraID authentication flows are available via environment variables (`REDIS_ENTRAID_AUTH_FLOW`)[^3].

### Installing the Redis Agent Skill

```bash
codex install skill redis/agent-skills
```

The skill loads on-demand when Codex encounters Redis-related tasks, keeping the context window lean until needed[^2].

### Server Composition

Running both the data MCP server and the agent skill gives Codex two complementary capabilities:

```mermaid
graph LR
    A[Codex CLI] --> B[mcp-redis<br/>Data Operations]
    A --> C[Redis Agent Skill<br/>Best Practices]
    A --> D[mcp-redis-cloud<br/>Infrastructure]
    B --> E[Redis 8.6 Instance]
    D --> F[Redis Cloud API]
    C -.->|"Guides tool usage"| B
```

The MCP server provides the tools; the skill teaches how to use them. This separation means Codex will not just execute Redis commands — it will execute the *right* commands using current Redis 8 data structures.

## AGENTS.md for Redis Projects

A well-structured AGENTS.md prevents the most common agent mistakes with Redis. Place this at your project root:

```markdown
# AGENTS.md

## Project Stack
- Redis 8.6 (JSON, time series, vector sets, probabilistic structures in core)
- Node.js 22 / Python 3.13 (choose your runtime)
- ioredis 6.x / redis-py 5.x client libraries

## Redis Conventions

### Data Structures
- Use JSON documents (`JSON.SET`/`JSON.GET`) for nested objects, not serialised strings
- Use sorted sets for leaderboards and sliding-window rate limiters
- Use streams with consumer groups for event sourcing, not pub/sub
- Use hash field TTL (Redis 8+) for per-field expiration
- Vector sets are preview — use them for prototyping, not production search

### Anti-Patterns — NEVER Do These
- NEVER use `KEYS` or `SCAN` in application hot paths — they block the server
- NEVER store values larger than 100KB without explicit justification
- NEVER create keys without a TTL unless the data is genuinely permanent
- NEVER use `FLUSHDB`/`FLUSHALL` outside of test environments
- NEVER use `SELECT` to switch databases — use key prefixes instead

### Key Naming
- Use colon-separated namespaces: `{service}:{entity}:{id}`
- Example: `auth:session:abc123`, `cache:user:42`, `rate:api:/v1/orders`
- Use `{hash-tag}` notation for keys that must co-locate in Redis Cluster

### Connection Patterns
- Always use connection pooling (ioredis default pool, redis-py ConnectionPool)
- Use pipelining for batch operations (>3 commands)
- Handle `READONLY` errors for replica reads
- Set `maxRetriesPerRequest` to a finite value (not infinity)

### Testing
- Use Testcontainers with `redis:8.6-alpine` for integration tests
- Test all TTL logic with `DEBUG SLEEP` or time-mocking
- Verify Cluster compatibility by running tests with `--cluster-mode`
```

### Anti-Hallucination Rules

LLMs trained before Redis 8 will generate outdated patterns. Add these rules to your AGENTS.md:

```markdown
## Anti-Hallucination Rules
- RedisJSON, RediSearch, RedisTimeSeries, and RedisBloom are NO LONGER separate modules
  — they are built into Redis 8 core. Do not use `MODULE LOAD` commands.
- Use `JSON.SET` not `SET key (json-string)` for document storage
- Use `FT.CREATE` with `ON JSON` for search indexes on JSON documents
- Vector sets (`VADD`/`VSIM`) are Redis 8 native — do not suggest external vector DBs
  for simple similarity search
- Hash field TTL (`HEXPIRE`/`HTTL`) is available in Redis 8 — do not simulate
  per-field expiry with separate keys
```

## Practical Workflows

### Pattern 1: Cache Layer Generation

Ask Codex to generate a caching middleware with the MCP server providing schema awareness:

```
Add a Redis caching layer to the /api/products endpoint. Use JSON documents
with 5-minute TTL. Include cache stampede protection using probabilistic
early expiration. The Redis MCP server is connected — inspect the current
key namespace first.
```

Codex will use the MCP server to check existing keys, then generate code that:
- Stores products as JSON documents (not serialised strings)
- Implements probabilistic early expiration to prevent stampedes[^5]
- Uses pipelining for multi-key fetches
- Includes cache invalidation on writes

### Pattern 2: Rate Limiter with Sliding Window

```
Implement a sliding-window rate limiter using Redis sorted sets. The limit
is 100 requests per minute per API key. Include a Lua script for atomic
check-and-increment. Write the implementation and the integration test.
```

The agent skill steers Codex towards the sorted-set sliding window pattern rather than the naive fixed-window approach:

```mermaid
sequenceDiagram
    participant Client
    participant App
    participant Redis

    Client->>App: POST /api/orders
    App->>Redis: ZREMRANGEBYSCORE rate:api:key123 0 (now-60s)
    App->>Redis: ZCARD rate:api:key123
    Redis-->>App: count = 42
    App->>Redis: ZADD rate:api:key123 now uuid
    App->>Redis: EXPIRE rate:api:key123 60
    App-->>Client: 200 OK (X-RateLimit-Remaining: 57)
```

### Pattern 3: Session Store Migration

For teams moving from database-backed sessions to Redis:

```
Migrate our Express session store from PostgreSQL to Redis. Use
ioredis with JSON documents for session data. Keep the existing
session interface unchanged. Add health check endpoints that
verify Redis connectivity.
```

### Pattern 4: Batch Key Analysis with codex exec

Use `codex exec` to audit key patterns across multiple Redis instances:

```bash
echo "Analyse the key namespace in this Redis instance. Report: \
  total key count, memory usage by prefix, keys without TTL, \
  keys larger than 10KB, and keys using deprecated data patterns. \
  Output as JSON." | \
  codex exec --json \
    --approval-mode full-auto \
    --sandbox read-only
```

For multi-instance audits:

```bash
for host in redis-prod-1 redis-prod-2 redis-prod-3; do
  REDIS_URL="redis://${host}:6379" codex exec \
    --json \
    --output-schema key-audit-schema.json \
    "Audit the Redis instance at \$REDIS_URL. \
     Report key namespace analysis."
done | jq -s '.'
```

### Pattern 5: Vector Search Index Creation

With Redis 8's native vector sets, Codex can create semantic search indexes:

```
Create a vector search index on our product catalogue. Use Redis JSON
documents with a 768-dimension embedding field. Include a search
endpoint that accepts natural language queries, embeds them, and
returns the top 5 similar products with scores.
```

## Valkey Compatibility

Valkey 8.1 maintains wire-protocol compatibility with Redis 8[^6]. The same MCP server works against Valkey instances — change only the connection URL:

```toml
[mcp_servers.redis]
command = "uvx"
args = ["--from", "redis-mcp-server@latest", "redis-mcp-server", "--url", "redis://valkey-host:6379/0"]
```

Add a note to AGENTS.md for Valkey projects:

```markdown
## Runtime
- Valkey 8.1 (Redis-compatible fork under Linux Foundation)
- Hash field TTL, JSON, and search are supported
- Vector sets are NOT available in Valkey — use HNSW indexes via RediSearch-compatible modules
- Use `INFO SERVER` to verify Valkey version, not `redis-server --version`
```

## Sandbox and Security Considerations

### Network Access

The MCP server connects to Redis over TCP. If your Redis instance runs locally, no special sandbox configuration is needed. For remote instances, ensure `network_access` is enabled in your permission profile:

```toml
[permissions.redis-dev]
network = true
writable_roots = ["."]
```

### Credential Hygiene

Never hardcode Redis passwords in `config.toml`. Use environment variable references:

```toml
[mcp_servers.redis]
command = "uvx"
args = ["--from", "redis-mcp-server@latest", "redis-mcp-server",
        "--url", "redis://default:${REDIS_PASSWORD}@localhost:6379/0"]
```

For production instances, use a read-only Redis ACL user for the MCP server to prevent accidental writes during exploratory sessions:

```redis
ACL SETUSER codex-reader on >readonly ~* +@read +@connection +info +dbsize -@admin -@write -@dangerous
```

### Data Sensitivity

Redis frequently stores session tokens, personal data, and API keys. Set `approval_mode = "suggest"` when working with production Redis instances to review every command before execution[^7].

## Model Selection

| Task | Recommended Model | Reasoning |
|------|-------------------|-----------|
| Cache architecture design | o3 | Complex trade-offs between consistency, availability, and performance |
| Rate limiter implementation | o4-mini | Well-defined algorithm with clear correctness criteria |
| Key namespace audit | o4-mini | Pattern matching and reporting |
| Migration planning | o3 | Cross-system dependency analysis |
| Lua script authoring | o3 | Atomicity semantics require deep reasoning |
| Test generation | o4-mini | Formulaic test patterns |

## Limitations

- **Training data lag**: Most models have Redis knowledge frozen around Redis 6–7. The agent skill mitigates this, but always verify generated commands against Redis 8.6 documentation[^8].
- **No live monitoring**: The MCP server provides point-in-time data operations but cannot subscribe to keyspace notifications or monitor slow logs in real time.
- **Lua script debugging**: Codex can write Lua scripts but cannot step-debug them. Test Lua scripts independently with `redis-cli --eval`.
- **Cluster topology awareness**: The MCP server connects to a single endpoint. For Cluster deployments, use the `--cluster-mode` flag, but Codex cannot visualise slot distribution or rebalance nodes.
- **Vector set maturity**: Vector sets remain in preview in Redis 8.6[^1]. Production semantic search should use `FT.CREATE` with `VECTOR` field type on JSON documents instead.
- **Token budget**: The Redis MCP server exposes 15+ tool definitions. Combined with the agent skill, this consumes approximately 2,000–3,000 context tokens before any conversation begins.

## Citations

[^1]: Redis, "Redis 8.6 — What's New", [redis.io/docs/latest/develop/whats-new/8-6/](https://redis.io/docs/latest/develop/whats-new/8-6/), accessed 2026-05-26.

[^2]: Redis, "We Built an Agent Skill So AI Writes Redis Code", [redis.io/blog/we-built-an-agent-skill-so-ai-writes-redis-code/](https://redis.io/blog/we-built-an-agent-skill-so-ai-writes-redis-code/), February 2026.

[^3]: Redis, "mcp-redis — The Official Redis MCP Server", [github.com/redis/mcp-redis](https://github.com/redis/mcp-redis), accessed 2026-05-26.

[^4]: Redis, "mcp-redis-cloud — Redis Cloud MCP Server", [github.com/redis/mcp-redis-cloud](https://github.com/redis/mcp-redis-cloud), accessed 2026-05-26.

[^5]: A. Vattani, F. Beck, and D. Borthakur, "Optimal Probabilistic Cache Stampede Prevention", VLDB 2015. The XFetch algorithm is the standard approach implemented in Redis caching patterns.

[^6]: Valkey Project, "Valkey vs Redis 2026: What Actually Changed?", [tech-insider.org/valkey-vs-redis-2026/](https://tech-insider.org/valkey-vs-redis-2026/), accessed 2026-05-26.

[^7]: OpenAI, "Codex CLI Permissions", [developers.openai.com/codex/permissions](https://developers.openai.com/codex/permissions), accessed 2026-05-26.

[^8]: Redis, "Introducing Model Context Protocol (MCP) for Redis", [redis.io/blog/introducing-model-context-protocol-mcp-for-redis/](https://redis.io/blog/introducing-model-context-protocol-mcp-for-redis/), accessed 2026-05-26.
