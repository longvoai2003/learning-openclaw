# OpenClaw Session and Memory Management Specifications

This document outlines the specifications for Session Management and Memory in OpenClaw, based on the official documentation.

## Session Management

OpenClaw organizes conversations into **sessions**. Each incoming message is routed to a specific session based on its source.

### 1. How Messages are Routed
| Source | Behavior |
| :--- | :--- |
| **Direct messages (DMs)** | Shared session by default (can be isolated). |
| **Group chats** | Isolated per group. |
| **Rooms/channels** | Isolated per room. |
| **Cron jobs** | Fresh session per run. |
| **Webhooks** | Isolated per hook. |

### 2. DM Isolation
To prevent different users from sharing the same conversation context in DMs, scope can be configured in the agent configuration:
*   **`main`**: All DMs share one session (default for single-user).
*   **`per-peer`**: Isolated by sender.
*   **`per-channel-peer`**: Isolated by channel and sender (recommended).
*   **`per-account-channel-peer`**: Isolated by account, channel, and sender.

**Example Configuration:**
```json
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

### 3. Session Lifecycle
Sessions are managed through several reset mechanisms:
*   **Daily Reset**: Automatically resets at 4:00 AM local time daily by default to keep context fresh.
*   **Idle Reset**: Can be configured via `session.reset.idleMinutes` to reset after a period of inactivity.
*   **Manual Reset**: Users can type `/new` or `/reset` in the chat. Typing `/new <model>` (e.g., `/new gpt-4o`) resets the session and switches the model simultaneously.

### 4. Where State Lives
All session state is owned by the gateway and persisted in the agent's workspace:
*   **Session Metadata**: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
*   **Transcripts**: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

### 5. Session Maintenance
OpenClaw automatically bounds session storage over time. It runs in `warn` mode by default. You can enable automatic cleanup by setting the mode to `enforce`:
```json
{
  "session": {
    "maintenance": {
      "mode": "enforce",
      "pruneAfter": "30d",
      "maxEntries": 500
    }
  }
}
```

### 6. Inspecting Sessions
You can monitor and manage sessions using the CLI or chat commands:
*   `openclaw status`: Shows the session store path and recent activity.
*   `openclaw sessions --json`: Lists all active sessions.
*   `/status`: A chat command that returns the session ID and current model.
*   `/context list`: A chat command showing what is in the system prompt.

---

## Memory

Memory allows agents to persist and retrieve information across sessions using a multi-engine architecture. It is accessed via the `memory_search` and `memory_store` tools.

### 1. Memory Engines
OpenClaw supports multiple engines for different use cases:
*   **Builtin Engine**: Uses a local SQLite database with hybrid search (combining vector similarity and keyword matching).
*   **QMD Engine**: A local-first sidecar engine that supports reranking and can index external directories for RAG.
*   **Honcho Engine**: An AI-native engine designed for complex user modeling and cross-session memory.

### 2. Search Behavior
The memory system uses **Hybrid Search**, which ranks results based on both:
1.  **Semantic Similarity**: How well the meaning matches the query.
2.  **Keyword Matching**: Identifying exact matches for specific terms.
