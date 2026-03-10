# System Architecture

Clawd HQ is a multi-agent AI system running on a distributed infrastructure anchored by a Mac Mini (M2) home server. Eight specialized AI agents operate autonomously on scheduled loops, coordinated through a shared event bus, persistent memory layer, and Telegram-based communication.

## High-Level Overview

```
                          +------------------+
                          |   Tailscale Mesh |
                          |   (5 machines)   |
                          +--------+---------+
                                   |
          +------------+-----------+-----------+------------+
          |            |           |           |            |
     Mac Mini     MacBook Air  Gaming PC     VPS        Router
     (primary)    (mobile)     (GPU)     (production)  (edge)
          |
          +--------------------------------------------------+
          |                Mac Mini Services                  |
          |                                                   |
          |  +-------------+  +----------+  +-----------+    |
          |  | 8 AI Agents |  | OpenClaw |  |  Docker   |    |
          |  | (scheduled) |  | Gateway  |  |  Stack    |    |
          |  +------+------+  +----+-----+  +-----+-----+   |
          |         |              |              |           |
          |    +----+----+    +----+----+    +----+----+     |
          |    |Event Bus|    |Mem0 MCP |    |Langfuse |     |
          |    |(SQLite) |    | Server  |    |Uptime K.|     |
          |    +---------+    +---------+    |Beszel   |     |
          |                                  |n8n      |     |
          |                                  +---------+     |
          +--------------------------------------------------+
```

## Mac Mini (M2) --- Primary Compute Node

The Mac Mini serves as the central hub. All eight agents run here via LaunchAgent-scheduled Claude Code sessions. Key responsibilities:

| Component          | Purpose                              |
|--------------------|--------------------------------------|
| OpenClaw Gateway   | Unified LLM API routing              |
| Mem0 SSE Proxy     | Memory access for Claude.ai          |
| A2A Protocol Server| Cross-machine agent queries           |

## OpenClaw Gateway

A local proxy that normalizes requests across multiple LLM providers, allowing agents to switch models without code changes.

```
Agent Request
     |
     v
OpenClaw Gateway
     |
     +---> Claude (Anthropic API)     # Primary reasoning
     +---> Gemini 2.5 Flash (Google)  # Synthesis, high-throughput
     +---> Mistral Large (Mistral)    # Fallback, cost optimization
```

Configuration is provider-agnostic. Agents specify intent (e.g., `fast`, `reasoning`, `cheap`) and the gateway routes accordingly.

## Tailscale Mesh Network

Five machines connected via Tailscale for secure, zero-config networking:

| Machine      | Role                | Key Services                        |
|--------------|---------------------|-------------------------------------|
| Mac Mini     | Primary compute     | Agents, Docker, OpenClaw, A2A       |
| MacBook Air  | Mobile development  | Claude Code, Chrome debug profile   |
| Gaming PC    | GPU workloads       | Model training, heavy inference     |
| VPS          | Production runtime  | Job automation, public-facing bots  |
| Router       | Network edge        | AdGuard Home, Tailscale exit node   |

All inter-machine communication uses Tailscale, eliminating port forwarding and firewall complexity.

## Docker Services (via Colima)

Docker runs on the Mac Mini through Colima. Four services provide observability and workflow automation:

| Service      | Purpose                                   |
|--------------|-------------------------------------------|
| Uptime Kuma  | Service health monitoring, alerting       |
| Langfuse     | LLM call tracing, cost tracking          |
| Beszel       | System metrics (CPU, memory, disk)        |
| n8n          | Workflow automation, webhook integrations |

Langfuse runs as a 6-container stack (app, worker, PostgreSQL, Redis, ClickHouse, MinIO).

## LaunchAgent Scheduling

macOS LaunchAgents manage all recurring agent tasks. Each agent has a plist defining its schedule, working directory, and environment.

```xml
<!-- Example: HAL Synthesis Heartbeat -->
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>ai.clawd.hal-synthesis</string>
    <key>StartInterval</key>
    <integer>21600</integer> <!-- 6 hours -->
    <key>ProgramArguments</key>
    <array>
      <string>/usr/local/bin/claude</string>
      <string>--agent</string>
      <string>hal</string>
      <string>--task</string>
      <string>synthesis</string>
    </array>
  </dict>
</plist>
```

Agents are not long-running daemons. They spin up, execute their task, write results to the event bus and Mem0, then exit. This keeps resource usage minimal and avoids stale state.

## Design Principles

1. **Local-first.** All critical data (event bus, configs, logs) lives on local disk. Cloud services (Mem0, LLM APIs) are used for capabilities, not storage of record.

2. **Stateless agents.** Agents reconstruct context from Mem0 and the event bus on each run. No in-memory state persists between invocations.

3. **Zero external infrastructure.** No Kubernetes, no cloud VMs for orchestration. LaunchAgents and SQLite handle scheduling and coordination.

4. **Graceful degradation.** If Gemini is down, agents fall back to Mistral. If Mem0 is unreachable, agents log locally and sync later.

5. **Observable by default.** Every LLM call is traced in Langfuse, every agent decision is logged to the event bus, every service is monitored by Uptime Kuma.
