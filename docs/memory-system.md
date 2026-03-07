# Memory System

The memory system gives Clawd HQ's agents persistent, queryable knowledge that survives across sessions and devices. Built on Mem0 Cloud, it provides two namespaces: a personal namespace for cross-device synchronization and a shared namespace for multi-user collaboration.

## Architecture

```
+------------------+     +------------------+     +------------------+
|   MacBook Air    |     |    Mac Mini      |     |   Claude.ai      |
|   Claude Code    |     |   Claude Code    |     |   (browser)      |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         |    MCP Server          |    MCP Server          |  SSE Proxy
         |    (local)             |    (local)             |  + Cloudflare
         |                        |                        |  Tunnel
         +------------+-----------+------------+-----------+
                      |                        |
              +-------+--------+       +-------+--------+
              | Mem0 Cloud     |       | Mem0 Cloud     |
              | Personal NS   |       | Shared NS      |
              | user: zabraham |       | user: clawd-hq |
              +----------------+       +----------------+
```

## Dual Namespace Design

### Personal Namespace

The personal namespace stores cross-device knowledge for a single user. Any device running Claude Code can read and write to this namespace.

| Property       | Value                          |
|----------------|--------------------------------|
| API Key        | `$MEM0_PERSONAL_API_KEY`       |
| user_id        | `zabraham`                     |
| Access         | All personal devices via MCP   |
| Content        | Preferences, discoveries, debugging solutions, environment details |

**Use cases:**
- Debugging solution found on MacBook Air, needed later on Mac Mini
- Project context that should follow the user across machines
- Architectural decisions and their rationale

### Shared Namespace

The shared namespace enables collaboration between multiple users or agents. It includes DLP (Data Loss Prevention) sanitization to prevent accidental credential leakage.

| Property       | Value                          |
|----------------|--------------------------------|
| API Key        | `$MEM0_SHARED_API_KEY`         |
| user_id        | `clawd-hq`                     |
| Access         | Authorized agents and users    |
| Content        | Agent outputs, synthesis, shared knowledge |

**DLP Sanitization:**

Before any memory is written to the shared namespace, a sanitization layer strips sensitive patterns:

```python
DLP_PATTERNS = [
    r'[A-Za-z0-9+/=]{32,}',      # API keys (base64-like strings)
    r'[\w.+-]+@[\w-]+\.[\w.]+',    # Email addresses
    r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b',  # Phone numbers
    r'postgres://\S+',             # Database URIs
    r'mongodb://\S+',              # Database URIs
    r'sk-[a-zA-Z0-9]{20,}',       # OpenAI-style keys
    r'm0-[a-zA-Z0-9]{20,}',       # Mem0-style keys
]

def sanitize(text: str) -> str:
    for pattern in DLP_PATTERNS:
        text = re.sub(pattern, '[REDACTED]', text)
    return text
```

## Agent Memory Scoping

Each agent writes memories with its `agent_id`, enabling fine-grained queries.

```python
# Trader writes a market observation
mem0.add(
    messages="BTC-YES contract mispriced at $0.42 vs model estimate $0.48",
    user_id="clawd-hq",
    agent_id="trader",
    metadata={"type": "position_snapshot", "market": "btc-price"}
)

# HAL queries only Trader's recent memories
trader_context = mem0.search(
    query="recent trading activity",
    user_id="clawd-hq",
    agent_id="trader",
    limit=10
)

# HAL queries ALL agents (synthesis mode)
all_context = mem0.search(
    query="recent activity and decisions",
    user_id="clawd-hq",
    limit=50
)
```

### Agent IDs

| Agent     | agent_id     |
|-----------|--------------|
| HAL       | `hal`        |
| Trader    | `trader`     |
| Recruiter | `recruiter`  |
| Banker    | `banker`     |
| Professor | `professor`  |
| CEO       | `ceo`        |
| Sentinel  | `sentinel`   |
| Trainer   | `trainer`    |

## Memory Types

Memories are categorized using the `type` field in metadata for structured retrieval:

| Type                | Agent(s)          | Purpose                                     |
|---------------------|-------------------|---------------------------------------------|
| `health_report`     | Sentinel          | Infrastructure health check results         |
| `position_snapshot` | Trader            | Current market positions and P&L            |
| `daily_balance`     | Banker            | End-of-day financial summary                |
| `listing_alert`     | Recruiter         | New job listings matching criteria           |
| `synthesis`         | HAL               | Cross-domain synthesis report               |
| `decision`          | Any               | Record of a consequential decision and why  |
| `discovery`         | Any               | New information relevant to the system      |
| `preference`        | Any               | User preference or configuration change     |

### Querying by Type

```python
# Get all health reports from the last 24 hours
health = mem0.search(
    query="infrastructure health status",
    user_id="clawd-hq",
    agent_id="sentinel",
    limit=24
)

# Get recent synthesis reports
synthesis = mem0.search(
    query="synthesis cross-domain analysis",
    user_id="clawd-hq",
    agent_id="hal",
    limit=5
)
```

## Access Topology

### MCP Server (Local Devices)

Claude Code on both the MacBook Air and Mac Mini connects to Mem0 via a locally running MCP server. The MCP server exposes five tools:

| Tool              | Purpose                        |
|-------------------|--------------------------------|
| `add_memory`      | Store a new memory             |
| `search_memories` | Semantic search across memories|
| `get_memories`    | Retrieve all memories for a user|
| `update_memory`   | Modify an existing memory      |
| `delete_memory`   | Remove a memory                |

Configuration in `~/.claude.json`:

```json
{
  "mcpServers": {
    "mem0": {
      "command": "npx",
      "args": ["-y", "mem0-mcp"],
      "env": {
        "MEM0_API_KEY": "$MEM0_PERSONAL_API_KEY"
      }
    }
  }
}
```

### SSE Proxy (Claude.ai)

Claude.ai (browser) cannot run local MCP servers. Instead, a Server-Sent Events (SSE) proxy runs on the Mac Mini and is exposed via a Cloudflare tunnel.

```
Claude.ai (browser)
     |
     v
Cloudflare Tunnel (dynamic URL)
     |
     v
Mac Mini SSE Proxy (port 8051)
     |
     v
Mem0 Cloud API
```

Two LaunchAgents maintain this:
- `ai.mem0.mcp-sse` --- SSE proxy server on port 8051
- `com.cloudflare.mem0-tunnel` --- Cloudflare quick tunnel (URL logged to `/tmp/cloudflared-mem0.log`)

## Memory Lifecycle

```
1. Agent runs and produces output
       |
2. Significant findings are written to Mem0
   (tagged with agent_id and memory type)
       |
3. If shared namespace: DLP sanitization runs first
       |
4. Memory stored in Mem0 Cloud
   (vector-indexed for semantic search)
       |
5. Other agents query relevant memories on their next run
       |
6. HAL queries ALL agent memories during synthesis
       |
7. Stale memories naturally decay in relevance ranking
```

## Design Decisions

**Why Mem0 over a local vector DB?** Cross-device access is a hard requirement. Running a local vector database (Chroma, Qdrant) would require replication or a centralized server. Mem0 Cloud provides managed vector search with an API that works from any device.

**Why two namespaces?** Personal memories (SSH configs, debugging solutions, preferences) should not leak into shared context. Shared memories (agent decisions, synthesis reports) need DLP protection. Separation enforces these boundaries architecturally.

**Why not use the event bus for memory?** The event bus captures discrete events with structured fields. Mem0 captures unstructured knowledge that benefits from semantic search. An event says "Trader opened a position"; a memory says "BTC prediction markets tend to misprice around Fed announcements." Different access patterns, different storage.
