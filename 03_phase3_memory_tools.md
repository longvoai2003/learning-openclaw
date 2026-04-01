# Phase 3: Memory, Sessions & Built-in Tools

## Phase Overview

**Big-picture goal:** By the end of this phase, you'll deeply understand OpenClaw's three most powerful systems: persistent memory (how your assistant remembers you), sessions (how conversations are managed), and built-in tools (how the assistant takes action). These are what transform OpenClaw from a chatbot into a personal agent.

**What you'll understand:**
- How memory is stored, retrieved, and injected into context
- The session lifecycle: creation, compaction, pruning, and reset
- The tool call architecture: how the model decides to use tools and how results flow back
- Browser control, shell execution, file system access, and web tools

**What you'll be able to do:**
- Teach your assistant persistent facts and retrieve them across sessions
- Manage sessions: create, reset, inspect, and clean up
- Use all built-in tools: browsing websites, running shell commands, reading/writing files
- Understand context windows and token usage

---

## What You Need Before This Phase

- [ ] Completed Phase 2 (working Gateway with at least one channel)
- [ ] A basic understanding of what "context window" means (the amount of text an AI model can process at once)

---

## 3.1 Persistent Memory

### Part A — Theory & Conceptual Understanding

#### What Memory Is (And Isn't)

**Memory** in OpenClaw is a system for storing **persistent facts** about you that survive across sessions. This is fundamentally different from conversation history.

| | Conversation History (Session) | Persistent Memory |
|---|---|---|
| **Lifespan** | Resets daily (by default) | Permanent until you delete it |
| **Content** | Every message and response | Key facts, preferences, decisions |
| **Size** | Can be very large (full transcript) | Compact (key-value style facts) |
| **Injection** | Loaded from session transcript | Injected into system prompt at start |

**Analogy:** Conversation history is like a phone call recording — everything said, verbatim. Memory is like the notes your assistant writes down in a notebook after the call: "Client prefers email over phone. Birthday: March 15. Allergic to peanuts."

#### How Memory Works Under the Hood

1. **Creation:** When you tell your assistant something important ("I'm a vegetarian"), the AI itself decides to store this as a memory. It uses a built-in memory tool to save the fact.
2. **Storage:** Memories are stored as structured records in the OpenClaw state directory (`~/.openclaw/`).
3. **Retrieval:** At the start of each agent turn, relevant memories are retrieved and injected into the system prompt before the AI sees your message.
4. **Context injection:** The memories appear in the AI's context as something like:
   ```
   ## User Memories
   - User's name is Alex
   - User is a vegetarian
   - User prefers dark mode interfaces
   - User works at AcmeCorp as a data engineer
   ```

This means the AI "knows" these things naturally — it doesn't need to look them up.

#### What Makes Good vs. Bad Memories

**Good memories** (worth storing):
- Your name, profession, location
- Preferences: communication style, dietary restrictions, favorite tools
- Ongoing projects: "Working on a Python ML pipeline for Kaggle"
- Key decisions: "We decided to use PostgreSQL, not MySQL"

**Bad memories** (don't store):
- Transient facts: "The weather today is 72°F"
- Obvious things: "User speaks English" (if all conversation is in English)
- Every detail ever discussed (that's what session history is for)

#### Memory and Token Impact

Memories are injected into every conversation turn, consuming tokens from your context window. 50 memories × ~20 tokens each = ~1,000 tokens of context per turn. This matters for:
- **Cost:** More tokens = more API cost per message
- **Context limit:** If memories + history + skills fill the context window, the AI loses the ability to see older messages

**Rule of thumb:** Keep your memory bank focused and pruned. Quality over quantity.

#### Theory Check

1. What is the key difference between persistent memory and conversation history?
2. How are memories injected into the AI's context? At what point during the agent loop does this happen?
3. Why should you be selective about what gets stored as a memory? What happens if you store too many?

---

### Part B — Hands-On: Working with Memory

**Step 1:** Tell your assistant something memorable:

> "Remember that my name is Alex, I work as a data scientist at TechCorp, and I prefer Python over R."

**Step 2:** Check that it was stored. Ask:

> "What do you know about me?"

The assistant should recall the facts you just shared.

**Step 3:** Start a new session to test persistence:

Type `/new` in the chat to reset the session, then ask:

> "What's my name and what do I do for a living?"

If memory is working correctly, it should still know — even though the conversation history was cleared.

**Step 4:** Manage memories via commands. Try:

> "List all the memories you have about me."

> "Forget that I prefer Python over R."

#### Common Mistakes
- **Expecting the AI to remember everything automatically:** The AI must actively decide to save a memory. Important facts stated casually might not be stored. Be explicit: "Please remember that..."
- **Confusing session context with memory:** If you don't reset the session, the AI remembers things from conversation history, not from persistent memory. The true test of memory is: does it survive `/new`?

---

## 3.2 The Session System

### Part A — Theory & Conceptual Understanding

#### What a Session Is

A **session** is a container for one continuous conversation. Think of it as a single "chat thread." Inside a session:
- All messages and responses are stored chronologically
- The AI has full context of everything said within that session
- Tool calls and their results are part of the transcript

#### The Session Lifecycle

```
Session Created ──→ Messages flow ──→ Context grows ──→ Compaction ──→ Eventually reset
       │                                                                      │
       │            Daily reset (4 AM default)                                │
       │            Idle reset (configurable minutes)                         │
       │            Manual reset (/new or /reset)                             │
       └──────────────────────────────────────────────────────────────────────┘
                                New session begins
```

**Three ways sessions reset:**
1. **Daily reset (default):** New session at 4:00 AM local time on the Gateway host
2. **Idle reset (optional):** New session after N minutes of inactivity (`session.reset.idleMinutes`)
3. **Manual reset:** Type `/new` or `/reset` in chat. `/new <model>` also switches the model

#### Compaction: Handling Long Conversations

When a conversation gets very long and approaches the context window limit, OpenClaw uses **compaction** — it summarizes older parts of the conversation to free up space while preserving the key information.

**Analogy:** Compaction is like taking meeting notes. Instead of keeping the full 2-hour recording, you keep a 1-page summary of key decisions and action items. You lose the verbatim details but retain the essential context.

#### Pruning: Trimming Tool Results

**Pruning** is a separate mechanism from compaction. When tool calls return very large results (e.g., a shell command that outputs 10,000 lines), pruning trims the tool output to save context space.

#### Where State Lives on Disk

- **Session index:** `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- **Transcripts:** `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`

Each transcript is a JSONL file (one JSON object per line), with each line representing a message, response, or tool call.

#### Session Maintenance Config

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

- `mode`: `"warn"` (default, just warns) or `"enforce"` (actually deletes old sessions)
- `pruneAfter`: Delete sessions older than this (e.g., `"30d"` = 30 days)
- `maxEntries`: Maximum number of session entries to keep

#### Inspecting Sessions

Useful commands:
- `openclaw status` — session store path and recent activity
- `openclaw sessions --json` — all sessions (filter with `--active <minutes>`)
- `/status` in chat — context usage, model, and toggles
- `/context list` — what is in the system prompt
- `openclaw sessions cleanup --dry-run` — preview what would be cleaned up

#### Theory Check

1. What are the three ways a session can be reset? Which is the default?
2. What is compaction, and why is it necessary? What does the AI lose when compaction happens?
3. Where are session transcripts stored on disk, and what format are they in?

---

### Part B — Hands-On: Mastering Sessions

**Step 1:** Check your current session status:

Type `/status` in the Dashboard. Note the context usage, model, and session ID.

**Step 2:** Inspect all sessions:

```bash
openclaw sessions --json
```

**Step 3:** Force a session reset:

Type `/new` in the chat. Then type `/status` again. Notice the new session ID.

**Step 4:** Examine a raw transcript:

```bash
ls ~/.openclaw/agents/*/sessions/
# Find a .jsonl file and view it:
head -20 ~/.openclaw/agents/*/sessions/*.jsonl
```

**Step 5:** Check what's in the system prompt:

Type `/context list` in chat. This shows you everything the AI sees before your message: system instructions, memories, skills, etc.

---

## 3.3 Built-in Tools: Browser, Shell, Files & Web

### Part A — Theory & Conceptual Understanding

#### The Tool Call Architecture

Tools are how the AI **takes action** in the real world. Here's how the architecture works:

1. The model receives your message plus the list of available tools (with descriptions)
2. Instead of responding with text, the model can respond with a **tool call** — a structured request like: `{"tool": "bash", "params": {"command": "ls -la ~/Documents"}}`
3. The Gateway **executes** the tool call on your machine
4. The result is sent back to the model as a new message
5. The model can then respond with text OR make another tool call
6. This loop continues until the model produces a final text response

**Analogy:** The AI is like a manager dictating to an assistant. The manager can't physically open a filing cabinet — but they can say "Please open the filing cabinet and read me the Johnson file." The assistant (the Gateway's tool executor) does the physical work and reports back.

#### Available Built-in Tools

| Tool | What It Does | Example Use |
|------|-------------|-------------|
| **bash/exec** | Runs shell commands on your machine | `ls`, `grep`, `git clone`, `python script.py` |
| **browser** | Controls a headless Chromium browser | Visit URLs, fill forms, click buttons, extract data |
| **file system** | Reads and writes files | Read configs, write reports, edit code |
| **web tools** | HTTP requests, URL fetching | Call APIs, scrape pages, download data |

#### Security and Sandboxing

These tools are POWERFUL — they give the AI the same capabilities as you sitting at your terminal. This is why security matters:

- **Default behavior:** Tools run with your user permissions. The AI can `rm -rf` just like you can.
- **Sandboxing (optional):** OpenClaw supports Docker-based sandboxing where tools run inside a container with limited permissions.
- **Elevated mode:** Certain dangerous operations prompt for explicit confirmation.

**Important:** The DM safety from Phase 2 is your first line of defense. If only you can talk to the assistant, only you can trigger tool calls.

#### Browser Control Deep Dive

OpenClaw's browser tool can:
- Navigate to any URL
- Read page content (converted to text/markdown)
- Click elements by selector
- Fill out forms
- Take screenshots
- Wait for elements to appear
- Execute JavaScript

This makes it a powerful tool for web scraping, form filling, and web automation.

#### Shell/Bash Execution

The bash tool can run any command your user account can run:
- File operations: `ls`, `cat`, `cp`, `mv`, `mkdir`
- System info: `df`, `top`, `uname -a`
- Development: `git`, `npm`, `python`, `docker`
- Networking: `curl`, `wget`, `ssh`

**Token cost awareness:** Large command outputs (e.g., `find / -type f`) consume many tokens. The AI should be taught to use focused commands. This is why pruning exists.

#### Theory Check

1. Describe the tool call loop. What happens when the AI model returns a tool call instead of text?
2. Why is sandboxing important? What could go wrong if the AI runs `rm -rf ~/` without any protection?
3. Name three things the browser tool can do that the simple web/HTTP tool cannot.

---

### Part B — Hands-On: Using Built-in Tools

**Step 1:** Test shell execution. Ask the Dashboard:

> "List all files in my Documents folder and tell me which is the largest."

Watch the AI run `ls` and `du` commands.

**Step 2:** Test file operations:

> "Create a file at ~/openclaw_test.txt with the content 'Hello from OpenClaw!' and then read it back to confirm."

**Step 3:** Test browser control:

> "Browse to https://news.ycombinator.com and tell me the top 3 headlines right now."

Watch the logs — you'll see the browser launching, navigating, and extracting content.

**Step 4:** Test web tools:

> "Make an HTTP request to https://api.github.com and tell me what the response looks like."

**Step 5:** Clean up:

> "Delete the file ~/openclaw_test.txt."

---

## Phase 3 Projects

---

### Project 1: "The Personal Profile" — Building Your Assistant's Memory of You

#### Theory Recap
Applies Sections 3.1 (memory) and 3.2 (sessions).

#### What You'll Build
A rich, multi-faceted personal profile stored in OpenClaw's memory that persists across sessions and demonstrates the memory system's full capabilities.

#### What You'll Learn
- Teaching the AI persistent facts strategically
- Testing memory persistence across session resets
- Managing and editing stored memories

#### Prerequisites
- Working OpenClaw with at least one channel

#### Step-by-Step Build Guide

1. **Teach your assistant about yourself across multiple messages:**

   > "Please remember: My name is Alex Chen, I'm a data scientist at TechCorp, and I live in San Francisco."

   > "Also remember: I prefer Python for data work, I use VS Code as my editor, and I'm currently learning Rust."

   > "Important preference: When I ask you to write code, always include comments explaining the logic."

2. **Verify the memories were stored:**

   > "List everything you know about me."

3. **Reset the session to test persistence:**

   Type `/new` to start a fresh session.

4. **Test recall in the new session:**

   > "What's my name, where do I work, and what programming languages do I use?"

   The assistant should recall all facts from memory.

5. **Test memory in action:**

   > "Write me a Python function that sorts a list of dictionaries by a specific key."

   Check: Does the response include comments? (You asked it to remember that preference.)

6. **Manage memories — delete one:**

   > "Forget that I'm learning Rust."

7. **Verify deletion:**

   > "What programming languages do I use?"

   Rust should no longer be mentioned.

#### How to Test It
- ✅ Assistant remembers your name, job, and preferences after `/new`
- ✅ Memory influences behavior (comments in code)
- ✅ You can delete specific memories
- ✅ Deleted memories don't resurface

#### Common Pitfalls
- **Not resetting the session before testing:** Without `/new`, you're testing conversation history, not persistent memory.
- **Vague memory requests:** "Remember stuff about me" is vague. Be specific about what to remember.

#### Stretch Goals
1. Store 20+ distinct facts and check the impact on `/status` context usage.
2. Ask the assistant to organize its memories about you into categories.
3. Check the raw memory storage files on disk to understand the storage format.

---

### Project 2: "The Research Assistant" — Web Browsing & Data Extraction

#### Theory Recap
Applies Section 3.3 (browser control, web tools) and 3.2 (sessions for conversational context).

#### What You'll Build
A multi-step research workflow where the AI browses multiple websites, extracts information, and compiles a structured report.

#### What You'll Learn
- Using the browser tool for real web data extraction
- Chaining multiple tool calls in a single conversation
- Handling large outputs and context management

#### Prerequisites
- Working OpenClaw installation with browser capabilities

#### Step-by-Step Build Guide

1. **Start with a research question.** Ask:

   > "I want you to research the current state of the Rust programming language. Please visit rust-lang.org and summarize what Rust is and its latest stable version."

2. **Chain to a second source:**

   > "Now browse to the Rust subreddit (reddit.com/r/rust) and tell me what the top 3 discussions are about right now."

3. **Request a comparison:**

   > "Browse to the Go programming language homepage (go.dev) and give me a comparison table: Rust vs Go on memory safety, performance, learning curve, and ecosystem size."

4. **Compile the research:**

   > "Based on everything you've found, write a 500-word summary report titled 'Rust vs Go in 2026' and save it to ~/research_report.md"

5. **Verify the output:**

   > "Read ~/research_report.md back to me and confirm it exists."

#### How to Test It
- ✅ The AI successfully browsed multiple websites
- ✅ Data was extracted and structured correctly
- ✅ A report file was created at the specified path
- ✅ The report contains information from multiple sources

#### Common Pitfalls
- **Websites blocking bots:** Some sites block headless browsers. The AI may need to use different approaches.
- **Context overflow:** Browsing multiple pages generates a LOT of text. The AI should summarize rather than dump raw page content.
- **Dynamic content:** JavaScript-heavy sites may require waiting for elements to load.

#### Stretch Goals
1. Ask the AI to browse a news site daily and summarize headlines (preview of automation in Phase 4).
2. Have the AI fill out a web form (e.g., a test form at `httpbin.org/forms/post`).
3. Chain browser + bash: browse for data, then create a Python script to visualize it.

---

### Project 3: "The System Administrator" — Shell Commands & File Management

#### Theory Recap
Applies Section 3.3 (bash/exec, file system tools).

#### What You'll Build
A series of system administration tasks performed by your AI assistant, demonstrating its ability to manage files, check system health, and automate routine tasks.

#### What You'll Learn
- Using the AI for system administration tasks
- Multi-step shell command chains
- Output parsing and reporting

#### Prerequisites
- Working OpenClaw installation
- Comfort with the idea of the AI running shell commands

#### Step-by-Step Build Guide

1. **System health check:**

   > "Run a full system health check: show me disk usage, memory usage, CPU info, and my operating system version."

2. **File organization:**

   > "List all .md files in my home directory (recursively). Create a summary listing each file with its size and last modified date."

3. **Create a utility script:**

   > "Write a bash script at ~/scripts/backup_configs.sh that copies my OpenClaw config (~/.openclaw/openclaw.json) to a timestamped backup in ~/backups/. Make the script executable."

4. **Run the script:**

   > "Run ~/scripts/backup_configs.sh and verify the backup was created."

5. **Clean up test artifacts:**

   > "Remove ~/scripts/backup_configs.sh and ~/openclaw_test.txt if it exists. Show me what was deleted."

#### How to Test It
- ✅ System health report was accurate
- ✅ Backup script was created and executed successfully
- ✅ Backup file exists with correct content
- ✅ Clean-up removed the correct files

#### Common Pitfalls
- **Destructive commands:** Be careful with `rm` commands. Always review what the AI plans to delete.
- **Permission errors:** Some system info commands may require `sudo`.
- **Script not executable:** `chmod +x` is needed before running scripts.

#### Stretch Goals
1. Ask the AI to monitor a log file in real-time and alert you (via the chat) when specific patterns appear.
2. Create a cron-style task that the AI sets up for you.
3. Have the AI write and run a Python script that analyzes your OpenClaw session transcripts and reports usage statistics.

---

## Phase 3 — End of Phase Review

### Conceptual Recap Questions

1. Explain persistent memory vs. session history. Give an analogy not used in this curriculum.
2. What is compaction? When does it trigger, and what tradeoff does it make?
3. Describe the complete tool call loop — from the AI deciding to use a tool to the result being incorporated into the response.
4. Why is session pruning important for long-running OpenClaw installations?
5. What security consideration is most important when the AI has shell access?

### Practical Consolidation Challenge

**"The Daily Briefing"**: Ask your assistant to do all of this in one conversation:
1. Check today's date and weather for your city (browser)
2. List your unread files modified today (shell)
3. Recall your name and job from memory
4. Write a personalized daily briefing document and save it to `~/daily_briefing.md`

This tests memory, browser, shell, and file tools all working together in a single agent loop.

---

**Phase 3 complete!** You now understand OpenClaw's core power systems. In Phase 4, you'll learn to extend these capabilities with custom skills, plugins, and automation.
