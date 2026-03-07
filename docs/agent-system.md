# Agent System

Clawd HQ operates a fleet of eight specialized AI agents. Each agent has a focused domain, its own memory scope, a dedicated Telegram topic for reporting, and access to the shared event bus.

## Agent Fleet

| Agent       | Domain               | Schedule  | Primary Model       |
|-------------|----------------------|-----------|---------------------|
| HAL         | Orchestrator         | Always-on | Claude (reasoning)  |
| Trader      | Prediction markets   | Every 4h  | Claude (reasoning)  |
| Recruiter   | Job automation       | Every 4h  | Claude (reasoning)  |
| Banker      | Personal finance     | Daily     | Claude (reasoning)  |
| Professor   | Education & research | On-demand | Claude (reasoning)  |
| CEO         | Strategy & planning  | On-demand | Claude (reasoning)  |
| Sentinel    | Infra monitoring     | Hourly    | Gemini Flash (fast) |
| Trainer     | Personal fitness     | On-demand | Gemini Flash (fast) |

## Agent Profiles

### HAL --- Orchestrator

The central coordinator. HAL runs the synthesis heartbeat every 6 hours, querying all agents' recent memories and events to identify cross-domain patterns. HAL is the only agent that reads broadly across all namespaces.

- **Responsibilities:** Cross-agent synthesis, conflict detection, resource allocation
- **Outputs:** Synthesis reports, strategic recommendations, alert escalation

### Trader --- Prediction Markets

Monitors and trades on Kalshi prediction markets. Tracks event probabilities, identifies mispriced contracts, and manages position sizing.

- **Responsibilities:** Market scanning, position management, P&L tracking
- **Outputs:** Position snapshots, trade decisions, market alerts

### Recruiter --- Job Automation

Automates job applications across LinkedIn and Indeed. Manages resume tailoring, application submission, and response tracking.

- **Responsibilities:** Job discovery, application automation, interview scheduling
- **Outputs:** Application logs, listing alerts, response tracking

### Banker --- Personal Finance

Ingests financial data, tracks spending patterns, monitors account balances, and generates financial reports.

- **Responsibilities:** Transaction categorization, budget tracking, bill monitoring
- **Outputs:** Daily balance snapshots, spending reports, anomaly alerts

### Professor --- Education & Research

Manages learning goals, tracks educational progress, and surfaces relevant research.

- **Responsibilities:** Curriculum planning, research synthesis, skill tracking
- **Outputs:** Study plans, research summaries, progress reports

### CEO --- Strategy & Planning

High-level strategic planning across all domains. Evaluates priorities, sets quarterly goals, and allocates attention across agents.

- **Responsibilities:** Goal setting, priority ranking, cross-domain strategy
- **Outputs:** Strategic plans, priority updates, decision rationale

### Sentinel --- Infrastructure Monitoring

Monitors system health across all five machines. Checks service availability, disk usage, network connectivity, and security.

- **Responsibilities:** Health checks, alerting, incident response
- **Outputs:** Health reports, infrastructure alerts, uptime metrics

### Trainer --- Personal Fitness

AI-powered personal trainer. Generates workout programs, tracks progress, provides nutrition guidance, and adapts training plans based on performance data.

- **Responsibilities:** Workout programming, progress tracking, nutrition planning, recovery monitoring
- **Outputs:** Training plans, progress reports, nutrition guidance

## Agent Anatomy

Every agent follows the same structural pattern:

```
agent/
  CLAUDE.md          # Agent personality, tools, constraints
  memory/
    MEMORY.md        # Persistent local memory (auto-managed)
  scripts/
    run.sh           # Entry point for LaunchAgent
  templates/
    report.md        # Output templates
```

### CLAUDE.md

Each agent's `CLAUDE.md` defines its identity, available tools, constraints, and standard operating procedures. This file is loaded as system context at the start of every invocation.

```markdown
# Sentinel Agent

## Role
Infrastructure monitoring and incident response.

## Tools
- SSH access to all Tailscale machines
- Uptime Kuma API for service status
- Beszel API for system metrics
- Event bus CLI for publishing alerts

## Constraints
- Never modify production services without logging to event bus
- Escalate critical alerts to Telegram immediately
- Health checks must complete within 60 seconds
```

### Mem0 Memory

Each agent writes memories tagged with its `agent_id`. This enables scoped queries (e.g., "all Trader memories from the last 24 hours") while allowing HAL to query across all agents during synthesis.

```python
# Agent writing a memory
mem0.add(
    messages="BTC-YES contract at $0.42, 3% below fair value estimate",
    user_id="clawd-hq",
    agent_id="trader",
    metadata={"type": "position_snapshot", "market": "btc-price"}
)
```

### Telegram Topic

Each agent posts to a dedicated topic in the Clawd HQ Telegram group. This provides a human-readable activity stream without requiring dashboard access.

| Agent      | Topic ID |
|------------|----------|
| Trader     | 5        |
| Recruiter  | 6        |
| Banker     | 7        |
| Professor  | 8        |
| CEO        | 9        |
| Sentinel   | 10       |
| Claude Code| 90       |

## Communication Patterns

Agents communicate through three mechanisms, each suited to different patterns:

### 1. Event Bus (Pub/Sub)

For structured, auditable inter-agent communication. An agent publishes an event; any other agent can read it on its next run.

```
Trader --[publishes: "large position opened"]--> Event Bus
                                                    |
HAL ----[reads on next synthesis]-------------------+
Banker --[reads on next financial check]------------+
```

### 2. Mem0 Shared Namespace

For persistent knowledge sharing. Agents write discoveries and decisions to Mem0; other agents query relevant memories during their runs.

```
Recruiter --[saves: "Company X hiring freeze"]--> Mem0
                                                    |
CEO ------[queries: "job market signals"]----------+
```

### 3. A2A Protocol

For synchronous, cross-machine queries. An agent on one machine can query another agent's knowledge in real time.

```
MacBook Air (Claude Code) --[A2A query]--> Mac Mini (HAL)
                                              |
                            [Mem0 search + Gemini synthesis]
                                              |
                           <--[structured answer]--
```

## Scheduling

Agents run on staggered schedules to avoid resource contention:

```
Hour:  0  1  2  3  4  5  6  7  8  9  10 11 12 ...
       |  |        |     |     |  |        |
       S  S     T/R/S  HAL    S  S     T/R/S  ...
       B                                B

S = Sentinel (hourly)
T = Trader (every 4h)
R = Recruiter (every 4h)
B = Banker (daily at midnight)
HAL = Synthesis (every 6h)
```

Schedules are defined in LaunchAgent plists and can be adjusted without code changes.
