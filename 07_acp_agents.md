# Phase 7: ACP Agents — Delegating Work to Specialist AI Agents

> All commands, config keys, and behavior in this document verified against **docs.openclaw.ai/tools/acp-agents** and **docs.openclaw.ai/tools/slash-commands** (fetched live, April 2026).

## Phase Overview

**Big-picture goal:** By the end of this phase, you will understand the Agent Client Protocol (ACP) — OpenClaw's system for spawning, binding, and orchestrating specialist AI agents — and be able to delegate research, coding, and other tasks to purpose-built agents directly from Telegram or Discord.

**What you'll understand:**
- What ACP is and how it differs from just switching models
- The difference between `oneshot` and `persistent` session modes
- How session binding (`--bind here`, `--thread auto`) works across channels
- How OpenClaw routes messages to a bound ACP session
- The full ACP lifecycle: spawn → steer → cancel/close

**What you'll be able to do:**
- Spawn a `deepseek-expert` ACP session from inside Telegram
- Bind ACP sessions to your current conversation or to a thread
- Manage active ACP sessions with `/acp status`, `/acp steer`, and `/acp close`
- Configure persistent ACP routing for Topic-based Telegram groups
- Troubleshoot common ACP errors with `/acp doctor`

---

## What You Need Before This Phase

- [ ] OpenClaw installed and configured (Phase 1 complete)
- [ ] At least one channel active — Telegram or Discord (Phase 2 complete)
- [ ] Agents configured in `openclaw.json` (e.g., `deepseek-expert`, `coder-bot`) with distinct models
- [ ] Running Gateway (`openclaw gateway status` shows healthy)

---

## 7.1 What Is ACP and Why Does It Exist?

### Part A — Theory & Conceptual Understanding

#### The Problem: One Agent, All Tasks

By default, your OpenClaw setup works like a star topology — every message you send goes to one primary agent (e.g., `main-tele-bot`), no matter how different the task is. Quick question? `main-tele-bot`. Deep research? `main-tele-bot`. Code review? `main-tele-bot`.

This works, but it is inefficient. A general-purpose agent backed by a free Qwen model handles quick questions cheaply. But when you need to do hours of research or write complex code, you want to bring in a specialist with a more capable — and sometimes more expensive — model. You also might want that specialist to work in a separate workspace without polluting the context of your daily conversation.

**That's what ACP solves.**

#### What ACP Is

**Agent Client Protocol (ACP)** is an open protocol (not invented by OpenClaw, but used by it) for structured communication between AI agents. In OpenClaw, "ACP agents" refers to the system that lets OpenClaw:

1. **Spawn** a separate specialist AI session (running a different agent, model, or harness like Codex or Claude Code)
2. **Route** messages from a specific conversation directly to that specialist session
3. **Collect** the specialist's output and deliver it back through your chat channel
4. **Control** the specialist's lifecycle from your chat with `/acp` slash commands

Think of your main bot (`main-tele-bot`) as a **project manager**: it takes your request, understands it, then delegates it to the right specialist. The specialist does the heavy work and returns results through the same channel. Your project manager stays available for your next request.

#### ACP vs. Sub-Agents

It's worth distinguishing these two things:

| Feature | ACP Agent | Sub-Agent |
|---|---|---|
| Session ID format | `agent:<id>:acp:<uuid>` | `agent:<id>:subagent:<uuid>` |
| Interface | `/acp ...` commands | `/subagents ...` commands |
| Persistence | Yes — can survive multiple turns | Typically ephemeral |
| Memory | Owns its own session context | Borrows parent context |
| Use case | Long-running specialist work | Quick parallel tasks |

For research and coding tasks that span multiple messages, ACP is the right tool.

#### ACP vs. `/model` Switching

| | `/model` switch | ACP spawn |
|---|---|---|
| Changes model? | Yes | Yes |
| Separate context? | ❌ Same context window | ✅ Isolated context |
| Conversation continues? | In-place, replaces model | Parallel session |
| Persists across turns? | Yes (session override) | Yes (bound session) |
| Closes cleanly? | `/model` back | `/acp close` |

**Rule of thumb:** Use `/model` for a quick "upgrade" when you want a better model for a single question. Use `/acp spawn` when you want a dedicated expert with its own workspace and memory for a prolonged task.

#### The Analogy: The Project Manager and Specialists

Think of it this way:

- **Your main bot** (`main-tele-bot`) = a project manager sitting at your desk
- **`deepseek-expert`** = a researcher in a separate office
- **`coder-bot`** = a developer in a separate office
- **`/acp spawn deepseek-expert --bind here`** = calling the researcher and saying "take all future messages in this chat until I say stop"
- **`/acp close`** = telling the researcher "you're done, put your project manager back on"

The researcher works in their own office (separate context). They don't see the project manager's unrelated conversations. When finished, the project manager takes back the channel.

#### Theory Check

1. In your own words, what is the key architectural difference between switching models with `/model` and spawning an ACP agent with `/acp spawn`? When would you choose each?
2. If you send a message to Telegram after running `/acp spawn deepseek-expert --bind here`, which agent processes it — `main-tele-bot` or `deepseek-expert`? Why?
3. What does the ACP session ID format `agent:deepseek-expert:acp:<uuid>` tell you about how OpenClaw identifies this session?

---

### Part B — Hands-On: Your First ACP Inspection

Before spawning any session, understand what ACP harnesses are available.

**Step 1:** Check your ACP system health from your terminal.

```bash
openclaw status
```
Displays a high-level summary including active sessions and channel connections.

**Step 2:** List all agents configured in your setup.

```bash
openclaw agents list --bindings
```
Lists all agents and their current channel bindings.

You should see your agents (`main-tele-bot`, `deepseek-expert`, `coder-bot`, etc.) with their configured models.

**Step 3:** Open the Dashboard in your browser.

```bash
openclaw dashboard
```
Opens the local Dashboard UI at `http://127.0.0.1:18789`.

Type `/status` in the Dashboard chat. Observe the response — note which agent and model are active.

#### Common Mistakes to Watch For

- **Confusing agents with models:** Agents are named configurations in `agents.list[]`. Models are the AI backends they use. An agent is the "role"; the model is the "capability."
- **Expecting ACP to work without configuration:** By default, ACP is available for `/acp spawn` from chat. But for persistent bindings in config, you need additional setup (covered in Section 7.3).

---

## 7.2 Spawning and Managing ACP Sessions

### Part A — Theory & Conceptual Understanding

#### The `/acp spawn` Lifecycle

Every ACP session goes through a lifecycle:

```
/acp spawn <agent> [flags]
        │
        ▼
    Session Created (unique UUID, own context)
        │
        ▼
    Messages route to this session
    (you talk, it works, it responds)
        │
        ▼
    /acp steer <instruction>    ← nudge without replacing context
    /acp status                 ← check what it's doing
    /acp cancel                 ← stop current turn
        │
        ▼
    /acp close                  ← remove binding, close session
```

#### Session Modes

When you spawn an ACP session, you choose a **mode**:

- **`persistent`** (default when `--bind here` is used): The session stays open across multiple messages. Follow-up messages continue the same conversation with the specialist. This is what you want for research or coding projects.
- **`oneshot`**: The session runs once for a single task and then closes itself. Good for one-time delegations where you don't need follow-up.

#### Binding Modes

The binding flag controls **how your channel conversation connects** to the ACP session:

| Flag | What It Does |
|---|---|
| `--bind here` | Pins the **current conversation** to the ACP session. All future messages in this chat route to the specialist. No new thread is created. |
| `--thread auto` | OpenClaw may create a **child thread or topic** and bind the ACP session there. The specialist lives in that thread. |
| `--thread here` | Binds the ACP session to the **currently active thread** (Telegram topic, Discord thread). |
| `--thread off` | No thread binding — the session runs but isn't bound to any conversation. |

> **Important:** `--bind here` and `--thread auto` are mutually exclusive — you can only use one.

#### The `/acp` Command Suite

These are the verified `/acp` commands available in chat (from docs.openclaw.ai/tools/slash-commands):

| Command | Purpose |
|---|---|
| `/acp spawn <harness> [flags]` | Create and optionally bind an ACP session |
| `/acp status` | Show active ACP sessions and their state |
| `/acp steer <instruction>` | Nudge an active session with a new instruction without resetting context |
| `/acp cancel` | Stop the current turn of the active session |
| `/acp close` | Close the session and remove its binding |
| `/acp sessions` | List all ACP sessions by key/ID/label |
| `/acp model <provider/model>` | Change the model for the active ACP session |
| `/acp permissions <profile>` | Change the permission profile |
| `/acp timeout <seconds>` | Set a timeout for the session's operations |
| `/acp doctor` | Diagnose ACP configuration issues |
| `/acp install` | Install the ACP backend (acpx) if not present |

#### Theory Check

1. You spawn an ACP session with `--bind here`. You then send 3 follow-up messages. How many ACP sessions exist? Where do those 3 messages go?
2. What is the difference between `/acp cancel` and `/acp close`? In what scenario would you use each one?
3. If you want DeepSeek to run a long research task in a separate Telegram topic so it doesn't interfere with your daily conversations, which combination of flags would you use?

---

### Part B — Hands-On: Spawning DeepSeek for Research

**Step 1:** Open your Telegram conversation with your `main-tele-bot`.

**Step 2:** Spawn `deepseek-expert` and bind it to the current conversation.

```
/acp spawn deepseek-expert --bind here
```
Creates a persistent ACP session using `deepseek-expert` and routes all future messages in this conversation to it.

You should receive a confirmation message showing the session ID (format: `agent:deepseek-expert:acp:<uuid>`).

**Step 3:** Ask a research question. Since the session is now bound, your message goes directly to `deepseek-expert`:

```
Research the top 5 most popular open-source computer vision datasets in 2025 and compare their size, license, and typical use cases.
```

Watch `deepseek-expert` respond with its DeepSeek model. Notice the response style may differ from your usual Qwen responses.

**Step 4:** Check what's active.

```
/acp status
```
Shows the current ACP session state, which agent is active, and how long it has been running.

**Step 5:** Steer the session without resetting context.

```
/acp steer focus only on datasets suitable for edge device deployment
```
Sends a steering instruction to refine the session's direction without losing the conversation history.

**Step 6:** Close the session when done.

```
/acp close
```
Closes the `deepseek-expert` session and returns routing to your default `main-tele-bot`.

**Step 7:** Verify the main bot is back by sending a normal message:

```
What agent are you using right now?
```

The response should confirm it's back to `main-tele-bot` with its Qwen model.

#### Common Mistakes to Watch For

- **`/acp spawn` without agent name:** You must specify the `agentId`. `/acp spawn` alone will fail. Always use `/acp spawn deepseek-expert` (or your target agent's exact ID).
- **Forgetting to close before spawning another:** You can have multiple ACP sessions, but `--bind here` can only bind one at a time per conversation. Use `/acp close` before re-binding to a different agent.
- **Expecting the main bot to respond while bound:** When you use `--bind here`, your main bot is bypassed. ALL messages go to the ACP session. If you need the main bot, close the ACP session first.

---

## 7.3 Persistent ACP Routing with Topic Bindings

### Part A — Theory & Conceptual Understanding

#### The Problem With Manual Spawning

Using `/acp spawn` manually is great for ad-hoc work. But what if you want **permanent routing**? For example:

- Every message in your Telegram "Research" topic → always routes to `deepseek-expert`
- Every message in your Telegram "Coding" topic → always routes to `coder-bot`
- Your main DMs → always routes to `main-tele-bot`

For this, you configure **persistent ACP bindings** in `openclaw.json`. These are different from the regular `bindings` you already have — they use `type: "acp"` and match specific channel peers (threads, topics, or channels by ID).

#### The Binding Model

A persistent ACP binding in config looks like this:

```json
{
  "type": "acp",
  "agentId": "deepseek-expert",
  "match": {
    "channel": "telegram",
    "accountId": "main-tele-bot",
    "peer": {
      "kind": "group",
      "id": "<chatId>:topic:<topicId>"
    }
  },
  "acp": {
    "mode": "persistent",
    "label": "research-session"
  }
}
```

Key parts (all verified against docs.openclaw.ai/tools/acp-agents):
- `type: "acp"` — marks this as an ACP binding (not a regular route binding)
- `agentId` — which agent in your `agents.list` handles this
- `match.peer.id` — the specific conversation to bind (format: `<chatId>:topic:<topicId>` for Telegram topics)
- `acp.mode` — `persistent` (session survives multiple turns) or `oneshot`
- `acp.label` — human-readable name for the session

#### Analogy: Permanent Phone Extension

Think of this like a phone switchboard: instead of manually transferring calls, you configure the switchboard so that calls from "Research Room" permanently ring to the researcher's desk, and calls from "Dev Room" permanently ring to the developer's desk. You set it up once, and it routes automatically forever.

#### How to Find a Telegram Topic ID

To bind a Telegram Forum Topic to an ACP session, you need both the **Chat ID** and the **Topic ID**:

1. Add your Telegram bot to a Forum-enabled Telegram Group
2. Create a topic in the group (e.g., "Research")
3. Send a message in that topic and observe your OpenClaw logs — the log will show the full peer ID in the format `-1001234567890:topic:42`

> **⚠️ Verify at docs.openclaw.ai:** Topic IDs are numeric (e.g., `42`), not topic names. Always use the numeric ID from logs.

#### Theory Check

1. What is the difference between a `type: "route"` binding and a `type: "acp"` binding in `openclaw.json`?
2. Why would you use a persistent config binding instead of manually running `/acp spawn` every time you start a Telegram session?
3. What happens when you send `/new` or `/reset` inside a conversation that is bound to a persistent ACP session?

---

### Part B — Hands-On: Configuring a Research Topic

**Step 1:** Ensure your Telegram Group has Topics enabled.

In Telegram, open your group → Group Info → Edit → Enable Topics (requires group admin).

**Step 2:** Create a "Research" topic in the group.

**Step 3:** Find the Topic ID.

Send a test message in the new topic. Then check your OpenClaw logs:

```bash
openclaw logs --follow
```
Streams Gateway logs live. Look for the peer ID in incoming message logs.

Find a log line containing your chat's peer ID — it will look like `-1001234567890:topic:5`. Note both numbers.

**Step 4:** Update `~/.openclaw/openclaw.json`.

Back up your config first:
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup
```
Creates a backup before editing.

Add the ACP binding under `"bindings"` *after* your existing route bindings:

```json
{
  "type": "acp",
  "agentId": "deepseek-expert",
  "match": {
    "channel": "telegram",
    "accountId": "main-tele-bot",
    "peer": {
      "kind": "group",
      "id": "YOUR_CHAT_ID:topic:YOUR_TOPIC_ID"
    }
  },
  "acp": {
    "mode": "persistent",
    "label": "research-topic"
  }
}
```

Replace `YOUR_CHAT_ID` and `YOUR_TOPIC_ID` with the values from Step 3.

**Step 5:** Restart the Gateway to apply the config.

```bash
openclaw gateway restart
```
Applies the updated configuration and reconnects to channels.

**Step 6:** Test by sending a message in your Research topic.

Send any message in that Telegram topic. The response should come from `deepseek-expert` (you'll notice it uses DeepSeek's characteristic response style, or you can ask "which model are you?").

**Step 7:** Verify the binding is active.

```bash
openclaw agents bindings --agent deepseek-expert
```
Lists all active bindings for the `deepseek-expert` agent.

#### Common Mistakes to Watch For

- **Wrong peer ID format:** Telegram topic IDs are `<chatId>:topic:<topicId>`. Using just the chat ID, or using the topic name instead of number, will silently fail to bind. **Fix:** Copy the exact format from your logs.
- **Forgetting `type: "acp"` in the binding:** A binding without `type: "acp"` is treated as a regular `route` binding, which routes to the agent but without ACP session management. **Fix:** Always include `"type": "acp"` explicitly.
- **Config JSON syntax errors:** A trailing comma or missing bracket will prevent the Gateway from starting. **Fix:** Validate with `python3 -m json.tool ~/.openclaw/openclaw.json` before restarting.

---

## Phase 7 Projects

---

### Project 1: "Research Delegation" — On-Demand DeepSeek Research Pipeline

#### Theory Recap
This project applies Sections 7.1 and 7.2. You'll practice spawning ACP sessions on-demand, steering them, and cleanly closing them.

#### What You'll Build
A repeatable workflow where you use your daily `main-tele-bot` for general tasks and summon `deepseek-expert` on demand for specific research questions, then return cleanly to your main bot.

#### What You'll Learn
- Spawning and closing ACP sessions with `/acp spawn` and `/acp close`
- Using `/acp steer` to refocus a running session
- Recognizing which agent is responding based on style and `/acp status`
- The before/after context separation between agents

#### Prerequisites
- `deepseek-expert` agent configured in `agents.list` with `openrouter/deepseek/deepseek-v3.2`
- Active Telegram channel with `main-tele-bot` as default

#### Step-by-Step Build Guide

1. **Start in your normal Telegram conversation.** Send a casual message to confirm `main-tele-bot` is active:
   ```
   What's the current date and time?
   ```
   Confirm it responds as `main-tele-bot`.

2. **Spawn DeepSeek for research:**
   ```
   /acp spawn deepseek-expert --bind here
   ```
   Routes all future messages to `deepseek-expert`.

3. **Submit a research task:**
   ```
   Research the best open-source tools for automating GPU rental and monitoring for machine learning workloads. List the top 5 with pros, cons, and GitHub stars.
   ```
   DeepSeek processes the request with its more capable research model.

4. **Steer the session to refine output:**
   ```
   /acp steer now focus only on tools that support spot instances and have active maintenance as of 2025
   ```
   Refines the session without starting over.

5. **Save the output** by asking DeepSeek to write a summary to a file:
   ```
   Write this summary to ~/research_output.md
   ```

6. **Close the session:**
   ```
   /acp close
   ```

7. **Confirm return to main-tele-bot:**
   ```
   What agent are you?
   ```

#### How to Test It
- ✅ `/acp spawn` confirms a session ID is returned
- ✅ Research response style differs noticeably from main-tele-bot
- ✅ `/acp steer` produces a refined response without restarting
- ✅ `~/research_output.md` exists with the summary
- ✅ After `/acp close`, main-tele-bot responds to subsequent messages

#### Common Pitfalls
- **Session not closing:** If you get "unable to resolve session target," it may already be closed. **Fix:** Run `/acp sessions` to see what's active.
- **File not saved:** The agent needs filesystem access. **Fix:** Ensure `tools.profile: "full"` is set in your `openclaw.json` and that the path is within your workspace or home directory.

#### Stretch Goals
1. Run a second `/acp spawn coder-bot --bind here` immediately after DeepSeek closes and give it the research output to turn into a Python script scaffold.
2. Use `/acp timeout 120` before the research task to enforce a 120-second timeout, then send a very complex query to see what happens when the timeout triggers.
3. Use `openclaw logs --follow` in a second terminal while running through this project and map each log line to the corresponding step in the ACP lifecycle.

---

### Project 2: "Specialist Routing Board" — Telegram Topic-Based Agent Farm

#### Theory Recap
This project applies Sections 7.1, 7.2, and 7.3. You'll configure permanent ACP routing so each Telegram topic always connects to a specific specialist agent.

#### What You'll Build
A Telegram Forum Group with three topics — "General", "Research", and "Coding" — each permanently bound to a different agent via `openclaw.json` config.

#### What You'll Learn
- Configuring `type: "acp"` bindings in `openclaw.json`
- Finding Telegram Topic IDs from logs
- Verifying multi-agent routing works independently per topic
- The difference between agent-level routing and topic-level routing

#### Prerequisites
- Telegram group with Topics enabled (admin access)
- `deepseek-expert` and `coder-bot` configured in `agents.list`
- `main-tele-bot` as Telegram default account

#### Step-by-Step Build Guide

1. **Create three Telegram Forum Topics:** "General", "Research", "Coding".

2. **Find the Topic IDs** by sending a test message to each and running:
   ```bash
   openclaw logs --follow
   ```
   Streams live logs to reveal peer IDs. Note the `chatId:topic:topicId` for each.

3. **Back up your config:**
   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup
   ```

4. **Add ACP bindings to your config's `"bindings"` array** for Research and Coding topics:
   ```json
   {
     "type": "acp",
     "agentId": "deepseek-expert",
     "match": {
       "channel": "telegram",
       "accountId": "main-tele-bot",
       "peer": { "kind": "group", "id": "CHAT_ID:topic:RESEARCH_TOPIC_ID" }
     },
     "acp": { "mode": "persistent", "label": "research" }
   },
   {
     "type": "acp",
     "agentId": "coder-bot",
     "match": {
       "channel": "telegram",
       "accountId": "main-tele-bot",
       "peer": { "kind": "group", "id": "CHAT_ID:topic:CODING_TOPIC_ID" }
     },
     "acp": { "mode": "persistent", "label": "coding" }
   }
   ```

5. **Restart the Gateway:**
   ```bash
   openclaw gateway restart
   ```

6. **Test each topic** by sending "Which model are you?" in each one:
   - General → `main-tele-bot` (Qwen free)
   - Research → `deepseek-expert` (DeepSeek V3)
   - Coding → `coder-bot` (Qwen Flash)

7. **Verify bindings from the terminal:**
   ```bash
   openclaw agents list --bindings
   ```

#### How to Test It
- ✅ "General" topic responds with `main-tele-bot`
- ✅ "Research" topic responds with `deepseek-expert`
- ✅ "Coding" topic responds with `coder-bot`
- ✅ Model names confirm correct routing (ask each "What model are you using?")
- ✅ `openclaw agents list --bindings` shows the ACP bindings

#### Common Pitfalls
- **Topic IDs are wrong:** Names won't work — only numeric IDs. **Fix:** Re-check logs for the exact `chatId:topic:topicId` format.
- **All topics still route to main-tele-bot:** The Gateway may not have restarted properly. **Fix:** Run `openclaw gateway status`. If it shows the old config is still loaded, run `openclaw gateway restart` again.
- **JSON syntax errors block startup:** One missing comma breaks everything. **Fix:** Validate with `python3 -m json.tool ~/.openclaw/openclaw.json` before restarting.

#### Stretch Goals
1. Add a 4th topic "Thinker" bound to `thinker-bot` (MiniMax) and use it for philosophical or multi-step reasoning questions. Compare output quality.
2. Configure `acp.cwd` for your `coder-bot` binding to point to a specific project directory so it always runs in your coding workspace.
3. Add a `channels.telegram.groups.<chatId>.topics.<topicId>.requireMention: false` override so you don't have to @mention the bot in topics.

---

## Phase 7 — End of Phase Review

### Theory Quiz

Test your conceptual understanding before moving on. These questions require synthesis, not just recall.

1. **Architecture:** OpenClaw's default setup routes all messages from a channel to one agent. ACP breaks this. Describe, in your own words, how ACP changes the message routing architecture — specifically where the message goes after it hits the Gateway.

2. **Modes:** A colleague tells you "I always use `oneshot` mode for ACP because I don't want lingering sessions." Is this always the right choice? Describe a scenario where `oneshot` is correct and another where `persistent` is necessary.

3. **Binding conflict:** You have a regular `type: "route"` binding for `main-tele-bot` on `telegram/main-tele-bot`, AND a `type: "acp"` binding for `deepseek-expert` on the same account but a specific topic. What happens when a message arrives in that topic? Which binding wins?

4. **`--bind here` vs `--thread auto`:** A user spawns with `--thread auto` and later complains "I can't find my conversation with DeepSeek." What likely happened, and how would you fix it?

5. **Lifecycle:** You spawn an ACP session but then need to ask your main bot a quick question without closing the DeepSeek session. Is this possible? If so, how?

6. **Troubleshooting:** A user sends `/acp spawn deepseek-expert --bind here` and receives `ACP runtime backend is not configured`. What is the most likely cause, and what command should they run to diagnose it?

7. **Config vs. chat:** When would you use a persistent ACP binding in `openclaw.json` instead of using `/acp spawn` manually every time? What are the trade-offs?

---

### Practical Consolidation Challenge

**"The Research-to-Code Pipeline"**: Build a fully automated multi-agent workflow entirely from within Telegram. The pipeline must:

1. Start in General (main-tele-bot): Ask the main bot to "research the top Python libraries for graph neural networks and write a summary."
2. Spawn `deepseek-expert` with `--bind here` to do the actual research.
3. Use `/acp steer` to focus the research on PyTorch-based libraries only.
4. Have DeepSeek save the research to `~/gnn_research.md`.
5. Close the DeepSeek session cleanly with `/acp close`.
6. Spawn `coder-bot` with `--bind here` and instruct it to read `~/gnn_research.md` and generate a Python starter template for the top recommended library.
7. Have coder-bot save the code to `~/gnn_starter.py`.
8. Close coder-bot.
9. Return to main-tele-bot and ask it to summarize both files and tell you next steps.

**Success criteria:**
- ✅ Both files exist on disk
- ✅ Three distinct agents were used (main-tele-bot, deepseek-expert, coder-bot)
- ✅ No agent contaminated another's context
- ✅ All sessions closed cleanly
- ✅ The final main-tele-bot summary references content from both files

---

## Future Development: What to Explore Next

Once you've mastered Phase 7, here are directions you can expand into:

### Near-Term (You Can Do This Now)
- **Automate ACP triggers:** Use OpenClaw cron jobs or webhooks to auto-spawn an ACP research session every morning at 8 AM with a daily research prompt
- **ACP + Memory:** After a DeepSeek research session, explicitly save key findings to `~/.openclaw/workspace/USER.md` so future sessions can reference them
- **Per-agent SOUL.md:** Give each agent (deepseek-expert, coder-bot) its own `agentDir` with its own persona and behavior instructions

### Intermediate
- **Multi-hop ACP chains:** Have `main-tele-bot` automatically spawn `deepseek-expert` to research, then spawn `coder-bot` to implement, by writing this logic as an OpenClaw skill
- **ACP + kaggle-monitor plugin:** Have your kaggle-monitor plugin automatically spawn `deepseek-expert` when it detects a new competition, to research the competition and generate a strategy brief
- **Discord thread-based routing:** Configure `channels.discord.threadBindings.spawnAcpSessions: true` and use ACP threads per coding task in Discord

### Advanced
- **Custom ACP harnesses:** Beyond the built-in patterns, explore how OpenClaw integrates with external ACP backends (requires understanding of the `acpx` plugin)
- **ACP + Task scheduling:** Combine `openclaw tasks` with ACP sessions to build fully autonomous research pipelines that run without human prompts
- **ACP streaming relay:** Use `streamTo: "parent"` in `sessions_spawn` to have the main bot relay live progress from a background ACP session back to the user
