# Synthesis System

The synthesis system is Clawd HQ's mechanism for cross-domain intelligence. While individual agents are domain experts, synthesis is where emergent insights surface --- connections that no single agent would detect in isolation.

## Overview

Every 6 hours, the HAL Synthesis Heartbeat collects recent activity from all eight agents, feeds it to a fast LLM, and produces a structured analysis of cross-domain patterns, risks, and opportunities.

```
+---------+  +---------+  +---------+  +---------+
| Trader  |  |Recruiter|  | Banker  |  |Sentinel |
| memories|  | memories|  | memories|  | memories|
+----+----+  +----+----+  +----+----+  +----+----+
     |            |            |            |
     +------+-----+-----+-----+-----+------+
            |           |           |
       +----+----+ +----+----+ +----+----+
       |  Mem0   | |  Event  | |  Local  |
       |  Query  | |   Bus   | | Reports |
       +----+----+ +----+----+ +----+----+
            |           |           |
            +-----+-----+-----------+
                  |
          +-------+--------+
          | Context Assembly|
          +-------+--------+
                  |
          +-------+--------+
          | Gemini 2.5 Flash|
          | (Synthesis LLM) |
          +-------+--------+
                  |
     +------------+------------+
     |            |            |
+----+----+ +----+----+ +----+----+
|  Mem0   | |Telegram | |Markdown |
| Storage | |  Post   | | Report  |
+---------+ +---------+ +---------+
```

## Synthesis Pipeline

### Step 1: Collect Agent Memories

HAL queries Mem0 for recent memories across all eight agent IDs. Each agent's memories are retrieved with a time window matching the synthesis interval.

```python
AGENT_IDS = [
    "hal", "trader", "recruiter", "banker",
    "professor", "ceo", "sentinel", "trainer"
]

all_memories = []
for agent_id in AGENT_IDS:
    memories = mem0.search(
        query="recent activity and decisions",
        user_id="clawd-hq",
        agent_id=agent_id,
        limit=20
    )
    all_memories.extend(memories)
```

### Step 2: Read Event Bus

HAL reads all events published since the last synthesis run. This captures structured decisions and alerts that may not have been written to Mem0.

```python
recent_events = eventbus.recent(hours=6)
unread_events = eventbus.feed(agent="hal")
```

### Step 3: Assemble Context

Memories and events are organized by agent and formatted into a structured prompt:

```
## Agent Activity (Last 6 Hours)

### Trader
- Opened BTC-YES position at $0.42 (3% below fair value)
- Market scan: 12 contracts evaluated, 3 flagged

### Recruiter
- 4 applications submitted (2 ML Engineer, 2 Backend)
- Interview scheduled: Company X, Thursday 2pm CST

### Banker
- Daily balance: $X,XXX.XX (+$XXX from yesterday)
- Unusual charge flagged: $XXX at merchant Y

### Sentinel
- All 14 services healthy
- Disk usage warning: Mac Mini at 87%
...
```

### Step 4: LLM Analysis

The assembled context is sent to Gemini 2.5 Flash with a structured prompt requesting three types of analysis:

```python
synthesis_prompt = """
Analyze the following multi-agent activity report. Identify:

1. CROSS-DOMAIN CONNECTIONS
   Patterns or relationships between different agents' activities
   that individual agents would not detect.

2. RISKS AND CONFLICTS
   Potential issues: resource conflicts, contradictory decisions,
   missed dependencies, or emerging problems.

3. ACTIONABLE INSIGHTS
   Specific recommendations for any agent based on the combined
   context. Reference which agents and data points support each.

Be concrete and specific. Avoid generic observations.
"""

response = gemini.generate(
    model="gemini-2.5-flash",
    prompt=synthesis_prompt + context,
    temperature=0.3
)
```

Gemini 2.5 Flash is used instead of Claude for synthesis because:
- Lower cost for high-context analysis (synthesis prompts can reach 10K+ tokens)
- Faster response times for a scheduled background task
- Sufficient quality for pattern-matching across structured data

### Step 5: Output Distribution

Synthesis results are distributed to three destinations:

#### Mem0 Storage

The full synthesis is saved as a persistent memory, queryable by any agent on future runs.

```python
mem0.add(
    messages=synthesis_report,
    user_id="clawd-hq",
    agent_id="hal",
    metadata={"type": "synthesis", "timestamp": now}
)
```

#### Telegram Post

A condensed summary is posted to the Clawd HQ Telegram group, providing a human-readable digest.

```
HAL Synthesis | 2026-03-06 06:00 CST

Cross-Domain:
- Trader's BTC position correlates with macro data
  Banker flagged as relevant to portfolio exposure

Risks:
- Mac Mini disk at 87% may impact Langfuse traces
  (Sentinel alert + Langfuse 6-container stack)

Actions:
- Sentinel: Schedule disk cleanup before next Trader run
- Banker: Flag BTC exposure in next financial report
```

#### Local Markdown Report

A detailed report is written to disk for archival and offline review.

```
~/clawd/reports/synthesis/
  2026-03-06T06:00.md
  2026-03-06T12:00.md
  2026-03-06T18:00.md
  ...
```

## Scheduling

The synthesis heartbeat runs via a macOS LaunchAgent:

```xml
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>ai.clawd.hal-synthesis</string>
    <key>StartInterval</key>
    <integer>21600</integer>
    <key>WorkingDirectory</key>
    <string>$PROJECT_ROOT/hal</string>
    <key>EnvironmentVariables</key>
    <dict>
      <key>GEMINI_API_KEY</key>
      <string>$GEMINI_API_KEY</string>
      <key>MEM0_API_KEY</key>
      <string>$MEM0_API_KEY</string>
    </dict>
    <key>StandardOutPath</key>
    <string>/tmp/hal-synthesis.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/hal-synthesis.err</string>
  </dict>
</plist>
```

## Example Synthesis Output

```markdown
# HAL Synthesis Report
**Generated:** 2026-03-06 06:00 CST
**Window:** Last 6 hours (00:00 - 06:00)
**Events processed:** 12 | **Memories queried:** 47

## Cross-Domain Connections

1. **Job market + prediction markets overlap**
   Recruiter flagged 3 AI/ML roles at fintech companies.
   Trader holds positions in AI-regulation markets.
   Connection: regulatory outcomes may affect both hiring
   trends and contract prices.

2. **Infrastructure capacity + trading frequency**
   Sentinel reports Mac Mini disk at 87%.
   Trader's market scan evaluates 12 contracts per cycle.
   Risk: if disk fills, Langfuse traces drop, losing
   trade decision audit trail.

## Risks

1. **Disk pressure (Medium)**
   87% usage with 6-container Langfuse stack actively writing.
   Recommendation: Sentinel should run cleanup before next cycle.

## Actionable Insights

1. Trader: Consider AI-regulation contract exposure given
   Recruiter's signal on fintech hiring trends.
2. Sentinel: Prioritize disk cleanup; set alert threshold to 80%.
3. Banker: Cross-reference trading P&L with overall portfolio view.
```

## Design Decisions

**Why periodic synthesis instead of real-time?** Agents produce information at different cadences. A 6-hour window collects enough signal for meaningful pattern detection without drowning in noise. Real-time synthesis would fire on every minor event.

**Why a separate LLM call instead of agent-to-agent queries?** Individual agents are domain experts. Synthesis requires a generalist perspective that holds all domains in context simultaneously. A dedicated LLM call with full cross-domain context produces better insights than agents querying each other pairwise.

**Why Gemini Flash over Claude?** Synthesis is a background task optimized for throughput and cost, not maximum reasoning depth. Flash handles the structured pattern-matching well at a fraction of the cost.
