# Event Bus

The event bus is the central nervous system of Clawd HQ. It provides structured, auditable communication between agents using a SQLite-backed publish/subscribe model.

## Design Philosophy

The event bus was designed around three constraints:

1. **Zero external dependencies.** No Redis, no Kafka, no message broker to maintain. SQLite is embedded, battle-tested, and requires no running service.
2. **Local-first.** All events are stored on local disk with instant read/write. No network latency for inter-agent communication.
3. **Concurrent-safe.** WAL (Write-Ahead Logging) mode enables multiple agents to read and write simultaneously without blocking.

### Why SQLite Over Alternatives

| Requirement          | Redis          | Kafka           | SQLite + WAL     |
|----------------------|----------------|-----------------|------------------|
| Zero ops overhead    | No (server)    | No (cluster)    | Yes              |
| Survives reboot      | Configurable   | Yes             | Yes              |
| Concurrent access    | Yes            | Yes             | Yes (WAL mode)   |
| Query flexibility    | Limited        | Limited         | Full SQL         |
| Backup               | RDB/AOF       | Topic retention | File copy        |
| Dependencies         | redis-server   | JVM + ZooKeeper | None             |

For a system where eight agents produce dozens of events per day (not thousands per second), SQLite is the right tool.

## Schema

### `events` Table

Stores all published events with structured metadata.

```sql
CREATE TABLE events (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp   TEXT    NOT NULL DEFAULT (datetime('now')),
    agent       TEXT    NOT NULL,
    category    TEXT    NOT NULL CHECK (category IN (
                    'decision', 'alert', 'state_change', 'discovery'
                )),
    severity    TEXT    NOT NULL DEFAULT 'info' CHECK (severity IN (
                    'info', 'warn', 'critical'
                )),
    title       TEXT    NOT NULL,
    body        TEXT,
    metadata    TEXT    -- JSON blob for arbitrary structured data
);

CREATE INDEX idx_events_agent    ON events(agent);
CREATE INDEX idx_events_category ON events(category);
CREATE INDEX idx_events_severity ON events(severity);
CREATE INDEX idx_events_ts       ON events(timestamp);
```

### `event_reads` Table

Tracks which agents have consumed which events, enabling "unread" queries.

```sql
CREATE TABLE event_reads (
    event_id  INTEGER NOT NULL REFERENCES events(id),
    agent     TEXT    NOT NULL,
    read_at   TEXT    NOT NULL DEFAULT (datetime('now')),
    PRIMARY KEY (event_id, agent)
);
```

## Event Categories

| Category       | Purpose                                      | Example                                    |
|----------------|----------------------------------------------|--------------------------------------------|
| `decision`     | An agent made a consequential choice         | Trader opened a new position               |
| `alert`        | Something requires attention                 | Sentinel detected a service outage         |
| `state_change` | A tracked entity changed state               | Recruiter received an interview invite     |
| `discovery`    | New information that may affect other agents | Professor found relevant research          |

## Severity Levels

| Level      | Behavior                                         |
|------------|--------------------------------------------------|
| `info`     | Logged to event bus only                         |
| `warn`     | Logged + broadcast to Mem0 shared namespace      |
| `critical` | Logged + broadcast to Mem0 + Telegram alert      |

The escalation chain ensures that high-severity events reach both persistent memory and human attention without requiring agents to coordinate directly.

## CLI Interface

The event bus is operated through a CLI tool that agents invoke during their runs.

### Publishing Events

```bash
eventbus publish \
    --agent sentinel \
    --category alert \
    --severity warn \
    --title "Disk usage above 85% on Mac Mini" \
    --body "$(df -h / | tail -1)" \
    --metadata '{"machine": "mac-mini", "disk": "/", "usage_pct": 87}'
```

### Reading the Feed

```bash
# All recent events (last 24 hours)
eventbus recent --hours 24

# Unread events for a specific agent
eventbus feed --agent hal

# Events from a specific category
eventbus recent --category alert --severity warn,critical
```

### Marking Events as Read

```bash
# Mark specific events as read
eventbus mark-seen --agent hal --events 42,43,44

# Mark all current events as read
eventbus mark-seen --agent hal --all
```

### Example Output

```
$ eventbus recent --hours 12

ID  | TIME              | AGENT     | CAT          | SEV   | TITLE
----|-------------------|-----------|--------------|-------|----------------------------------
47  | 2026-03-06 08:00  | sentinel  | alert        | warn  | Disk usage above 85% on Mac Mini
46  | 2026-03-06 06:00  | hal       | discovery    | info  | Synthesis: 3 cross-domain links
45  | 2026-03-06 04:12  | trader    | decision     | info  | Closed BTC-YES at $0.58 (+$32)
44  | 2026-03-06 04:00  | trader    | state_change | info  | Market scan complete: 12 contracts
43  | 2026-03-06 01:00  | sentinel  | state_change | info  | All 14 services healthy
42  | 2026-03-06 00:00  | banker    | state_change | info  | Daily balance: $X,XXX.XX
```

## Lifecycle of an Event

```
1. Agent runs on schedule
       |
2. Agent performs its task (market scan, health check, etc.)
       |
3. Agent publishes event(s) to the bus
       |
4. If severity >= warn:
       +---> Broadcast to Mem0 shared namespace
       +---> If critical: send Telegram alert
       |
5. Other agents read events on their next run
       |
6. HAL reads ALL events during synthesis heartbeat
       |
7. Events persist indefinitely (queryable history)
```

## Concurrency Model

SQLite WAL mode allows multiple readers and a single writer concurrently. In practice, agents rarely write simultaneously due to staggered scheduling, but WAL ensures correctness if they do.

```
Agent A (reading)  ----[READ]----[READ]----[READ]---->
Agent B (reading)  ------[READ]------[READ]---------->
Agent C (writing)  ----------[WRITE]----------------->
                   All operations succeed concurrently
```

The database file is opened with:

```python
conn = sqlite3.connect("eventbus.db")
conn.execute("PRAGMA journal_mode=WAL")
conn.execute("PRAGMA busy_timeout=5000")
```

The `busy_timeout` ensures that if a write lock is briefly held, other operations wait rather than fail.

## Retention and Maintenance

Events are never automatically deleted. The database grows slowly (each event is ~500 bytes), so a year of operations produces roughly 5 MB of data. If pruning is ever needed:

```sql
-- Archive events older than 90 days
DELETE FROM events WHERE timestamp < datetime('now', '-90 days');
DELETE FROM event_reads WHERE event_id NOT IN (SELECT id FROM events);
VACUUM;
```
