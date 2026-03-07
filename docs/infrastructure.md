# Infrastructure

Clawd HQ runs on a five-machine Tailscale mesh with the Mac Mini (M2) as the primary compute node. This document covers the physical topology, scheduling system, containerized services, and monitoring stack.

## Tailscale Mesh Network

All machines are connected via Tailscale, providing encrypted peer-to-peer connectivity without port forwarding or firewall configuration.

```
                    +---------------------------+
                    |       Tailscale Mesh      |
                    |     (WireGuard overlay)    |
                    +---------------------------+
                    |                           |
     +--------------+--+    +--+---------------+-----+
     |                 |    |                         |
+----+------+   +------+----+---+   +---------+------+--+
| Mac Mini  |   | MacBook Air   |   | Gaming PC          |
| (M2)      |   | (M3)          |   | (Windows)          |
| Primary   |   | Mobile Dev    |   | GPU Tasks          |
| Compute   |   |               |   |                    |
+-----------+   +---------------+   +--------------------+
     |                                    |
+----+------+                    +--------+-------+
| VPS       |                    | GL.iNet Router |
| (Digital  |                    | (Network Edge) |
|  Ocean)   |                    |                |
+-----------+                    +----------------+
```

### Machine Roles

| Machine      | Hardware          | Role                  | Key Services                           |
|--------------|-------------------|-----------------------|----------------------------------------|
| Mac Mini     | Apple M2, 16GB    | Primary compute       | 8 agents, Docker stack, OpenClaw, A2A  |
| MacBook Air  | Apple M3, 16GB    | Mobile development    | Claude Code, Chrome debug profile      |
| Gaming PC    | AMD/NVIDIA GPU    | GPU workloads         | Model training, batch inference        |
| VPS          | 2 vCPU, 4GB RAM   | Production automation | Job-legion, public-facing services     |
| Router       | GL-iNet GL-MT3000 | Network edge          | AdGuard Home, Tailscale subnet router  |

### Network Addressing

All inter-machine communication uses Tailscale IPs (`100.x.x.x`). Services bind to `0.0.0.0` and are accessible from any machine on the mesh. No ports are exposed to the public internet.

## LaunchAgent Scheduling

macOS LaunchAgents manage all recurring tasks. Each agent has a dedicated plist that defines its schedule, environment, and logging.

### Active Schedules

| LaunchAgent                   | Agent     | Interval | Purpose                          |
|-------------------------------|-----------|----------|----------------------------------|
| `ai.clawd.hal-synthesis`      | HAL       | 6 hours  | Cross-agent synthesis heartbeat  |
| `ai.clawd.sentinel-health`    | Sentinel  | 1 hour   | Infrastructure health checks     |
| `ai.clawd.trader-monitor`     | Trader    | 4 hours  | Market scanning and position mgmt|
| `ai.clawd.banker-ingest`      | Banker    | Daily    | Financial data ingestion         |
| `ai.clawd.recruiter-alert`    | Recruiter | 4 hours  | Job listing discovery            |

### LaunchAgent Structure

```xml
<!-- Template: ~/Library/LaunchAgents/ai.clawd.<agent>.plist -->
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>ai.clawd.<agent-name></string>

    <key>ProgramArguments</key>
    <array>
      <string>/bin/bash</string>
      <string>/Users/zabraham/clawd/<agent>/scripts/run.sh</string>
    </array>

    <key>StartInterval</key>
    <integer>3600</integer> <!-- seconds between runs -->

    <key>WorkingDirectory</key>
    <string>/Users/zabraham/clawd/<agent></string>

    <key>EnvironmentVariables</key>
    <dict>
      <key>PATH</key>
      <string>/usr/local/bin:/usr/bin:/bin</string>
      <key>ANTHROPIC_API_KEY</key>
      <string>$ANTHROPIC_API_KEY</string>
    </dict>

    <key>StandardOutPath</key>
    <string>/tmp/clawd-<agent>.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/clawd-<agent>.err</string>

    <key>RunAtLoad</key>
    <true/> <!-- Run once immediately on load -->
  </dict>
</plist>
```

### Management Commands

```bash
# Load/start an agent schedule
launchctl load ~/Library/LaunchAgents/ai.clawd.sentinel-health.plist

# Unload/stop an agent schedule
launchctl unload ~/Library/LaunchAgents/ai.clawd.sentinel-health.plist

# Check if running
launchctl list | grep ai.clawd

# View logs
tail -f /tmp/clawd-sentinel.log
```

## Docker Stack (Colima)

Docker runs on the Mac Mini via Colima, a lightweight container runtime for macOS.

```bash
# Start Colima with resource limits
colima start --cpu 4 --memory 8 --disk 60
```

### Services

| Service       | Containers | Port | Image                         |
|---------------|------------|------|-------------------------------|
| Uptime Kuma   | 1          | 3001 | `louislam/uptime-kuma:1`      |
| Beszel Hub    | 1          | 8090 | `henrygd/beszel`              |
| Langfuse      | 6          | 3000 | `langfuse/langfuse` + deps    |
| n8n           | 1          | 5678 | `n8nio/n8n`                   |

Langfuse is the most complex deployment, requiring:
- Application server (port 3000)
- Background worker
- PostgreSQL database
- Redis cache
- ClickHouse analytics
- MinIO object storage

All services use Docker volumes for persistent storage and restart automatically via `restart: unless-stopped`.

## A2A Protocol Server

The Agent-to-Agent (A2A) protocol server enables cross-machine agent queries. Any machine on the Tailscale mesh can query the Mac Mini's agent knowledge base.

```
Requester                          Mac Mini (port 9999)
    |                                     |
    +--[A2A request]--------------------->|
    |                              +------+------+
    |                              | Search Mem0 |
    |                              | for context |
    |                              +------+------+
    |                              +------+------+
    |                              | Gemini Flash|
    |                              | synthesizes |
    |                              +------+------+
    |<--[structured response]-------------+
```

**Agent Card** (discovery endpoint):

```
GET http://<tailscale-ip>:9999/.well-known/agent.json

{
  "name": "HAL",
  "description": "Clawd HQ orchestrator agent",
  "url": "http://<tailscale-ip>:9999",
  "capabilities": ["memory-search", "synthesis", "event-query"]
}
```

Built on the official Google A2A SDK (`a2a-sdk`).

## BlueBubbles (SMS Gateway)

BlueBubbles runs on the Mac Mini, providing API access to iMessage for SMS verification during automated signups.

```bash
# Read recent messages
curl -s -X POST "http://localhost:1234/api/v1/message/query?password=$BLUEBUBBLES_PASSWORD" \
  -H "Content-Type: application/json" \
  -d '{"limit": 5, "sort": "DESC"}'
```

Used by agents that need to verify accounts or read incoming codes as part of automated workflows.

## Monitoring Pyramid

Monitoring is layered, with each level providing a different view of system health:

```
                    +-------------------+
                    |   Event Bus       |
                    | (Agent Decisions) |
                    +--------+----------+
                             |
                    +--------+----------+
                    |    Langfuse       |
                    | (LLM Traces)     |
                    +--------+----------+
                             |
                    +--------+----------+
                    |     Beszel        |
                    | (System Metrics)  |
                    +--------+----------+
                             |
                    +--------+----------+
                    |   Uptime Kuma     |
                    | (Service Health)  |
                    +-------------------+
```

| Layer        | Tool        | Monitors                                    | Alert Channel     |
|--------------|-------------|---------------------------------------------|--------------------|
| Service      | Uptime Kuma | HTTP endpoints, TCP ports, Docker containers | Telegram, email    |
| System       | Beszel      | CPU, memory, disk, network across all nodes  | Dashboard          |
| LLM          | Langfuse    | API calls, token usage, latency, cost        | Dashboard          |
| Agent        | Event Bus   | Decisions, state changes, discoveries        | Mem0 + Telegram    |

### Health Check Flow

Sentinel runs hourly and queries each monitoring layer:

```
1. Uptime Kuma API  --> Are all 14 services responding?
2. Beszel API       --> Any machine above 85% disk/CPU/memory?
3. Langfuse API     --> Any LLM errors or unusual cost spikes?
4. Event Bus        --> Any critical events since last check?
       |
       v
   Publish health_report event
   (escalate if issues found)
```

## Disaster Recovery

| Component       | Backup Strategy                                     |
|-----------------|-----------------------------------------------------|
| Event Bus       | SQLite file copy (cron, daily)                      |
| Agent Configs   | Git repository (clawd-hq)                           |
| Docker Volumes  | Colima snapshot + volume tar archives                |
| Mem0 Memories   | Cloud-hosted (Mem0 manages durability)               |
| LaunchAgents    | Git repository (clawd-hq)                           |

Recovery procedure: clone repo, restore SQLite backup, reload LaunchAgents, restart Docker stack. Target recovery time: under 30 minutes.
