# Appendix: Advanced openclaw.json Configuration

As your OpenClaw deployment grows in complexity, you may want to exert finer control over the system's runtime behavior. While the `openclaw onboard` wizard provides a stable baseline, `openclaw.json` exposes a much broader set of configuration options, as detailed in the official documentation at **https://docs.openclaw.ai/**.

This appendix covers three highly useful advanced fields that you can manually add to your configuration file to fundamentally alter how your local assistant behaves.

---

## 1. Customizing Agent Personality & Knowledge: Workspace Markdown Files

### How It Works
By default, OpenClaw agents have a baseline objective to be a helpful, honest personal assistant. However, you can radically change an agent's persona or embed immense, specialized context directly into the system prompt. Instead of using `openclaw.json`, OpenClaw dynamically reads system instructions from Markdown files located in your workspace directory (typically `~/.openclaw/workspace/`).

The primary files used for instruction injection are:
- `AGENTS.md`: Core operating instructions, persistent goals, and memory.
- `SOUL.md`: Persona, tone, boundaries, and behavioral constraints.
- `IDENTITY.md`: Name, general vibe, and emoji representation.

### Advanced Example: The Curriculum Designer
One powerful pattern is to give OpenClaw an incredibly robust set of "Setup Rules" or a specialized persona. For instance, you could configure your assistant to act as a senior university professor designing an OpenClaw curriculum. 

You can add a massive "Setup Rules" prompt like this directly into your `~/.openclaw/workspace/SOUL.md`:

> "You are an expert curriculum designer and OpenClaw specialist... Think of yourself as a senior university professor building a complete professional training program — not just a list of topics. Every single section of this curriculum must follow this strict two-part structure: Part A — Theory & Conceptual Understanding. Part B — Hands-On Tutorials & Projects..."

By injecting rules of this caliber into your workspace files, every single session inherits this specialized framework, turning your general-purpose Gateway into a domain-expert machine.

---

## 2. Tuning Session Persistence: `session.reset.idleMinutes`

### How It Works
By default, OpenClaw automatically resets idle conversational sessions to prevent the context window from filling up with old, irrelevant messages (usually a 24-hour cycle at 4 AM local time, or after prolonged inactivity). However, you may want a session to be cleared more aggressively for privacy, or kept open indefinitely for an ongoing project.

### The Field Structure
```json
{
  "session": {
    "reset": {
      "idleMinutes": 1440
    }
  }
}
```

### Why Use It?
- **Shorter windows (e.g., `60` minutes):** Ideal if you only use OpenClaw for quick, one-off questions and want to aggressively manage token usage by dumping history after an hour.
- **Longer windows (or disabling):** Better if you are collaborating on a software project over several days and need the assistant to maintain immediate conversational awareness without relying solely on long-term memory retrieval.

---

## 3. Developer Hot-Reloading: `gateway.reload.mode`

### How It Works
When you edit `openclaw.json`, the changes do not take effect immediately because the Gateway process reads the configuration file at startup. Traditionally, you must run `openclaw gateway restart`. For developers writing plugins and skills, this restart cycle slows down the feedback loop.

### The Field Structure
```json
{
  "gateway": {
    "reload": {
      "mode": "hybrid"
    }
  }
}
```

### Why Use It?
Setting `gateway.reload.mode` to `"hybrid"` tells the Gateway to watch its configuration files and local skill directories for changes. When a modification is detected, the Gateway attempts to hot-swap the specific changed components (like updating a system prompt) without tearing down the entire server or dropping active WebSocket connections.

---

## Finding More Configuration Fields

To see the complete schema for `openclaw.json`, you should always consult the formal documentation:

1. **Visit [docs.openclaw.ai](https://docs.openclaw.ai)** and navigate to the "Configuration Reference" section.
2. The schema is organized hierarchically, matching the JSON tree. If you want to know what can go inside `"channels": { ... }`, look for the "Channels Configuration" chapter.
3. Every field, its accepted type (string, boolean, integer), and its default behavior is listed there. You are encouraged to read the docs thoroughly, as new features are frequently exposed entirely through JSON flags before they gain a Control UI interface.
