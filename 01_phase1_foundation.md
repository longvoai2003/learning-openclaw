# Phase 1: Foundation — Understanding and Installing OpenClaw

## Phase Overview

**Big-picture goal:** By the end of this phase, you will understand *what* OpenClaw is, *how* its architecture works at a conceptual level, and have a fully running local installation that you can talk to through the Dashboard. You will not just follow commands — you will understand every piece of the system you are setting up.

**What you'll understand:**
- The mental model of a self-hosted AI assistant vs. cloud-hosted chatbots
- Why OpenClaw uses a Gateway architecture and what that means
- The role of Node.js, the daemon, and the config file
- How to install, configure, and verify OpenClaw
- How to use the Dashboard and Control UI

**What you'll be able to do:**
- Install OpenClaw on your machine using any of the three methods
- Run the onboarding wizard understanding what each step does under the hood
- Start, stop, and monitor the Gateway
- Hold your first real conversation through the Dashboard

---

## What You Need Before This Phase

- [ ] A computer running macOS, Windows, or Linux
- [ ] An internet connection
- [ ] Basic ability to open a terminal and type commands
- [ ] An API key from Anthropic, OpenAI, or Google (you'll get this during onboarding if you don't have one)

---

## 1.1 What is OpenClaw?

### Part A — Theory & Conceptual Understanding

#### Definition and Purpose

OpenClaw is an **open-source personal AI assistant** that runs entirely on your own machine. Unlike ChatGPT, Google Gemini, or other cloud-hosted AI chatbots where your conversations go to someone else's server, OpenClaw runs a server *on your hardware*. Your data stays yours.

But OpenClaw is much more than a local chatbot. It is a **platform** — a programmable layer that:

1. **Connects to your chat apps** — WhatsApp, Telegram, Discord, Slack, Signal, iMessage. You text your AI assistant using the apps you already use daily.
2. **Remembers you** — It has persistent memory across conversations. It learns your preferences, your name, your projects, your habits.
3. **Takes action** — It can browse the web, run shell commands, read and write files on your computer, and control your system.
4. **Is extensible** — Through a skills and plugins system, you (or the community) can teach it new abilities.

#### The Problem It Solves

Consider the typical AI chatbot experience:

- You open a website, type something, get an answer, close the tab. Next time, it has forgotten everything.
- You can't talk to it from WhatsApp. You can't ask it to check your files. You can't automate it.
- Your conversations are stored on someone else's server.

OpenClaw solves all of this. It is designed to be your **persistent, private, capable AI assistant** that lives on your machine and reaches you wherever you are.

#### The Analogy: Your Personal Butler's Office

Think of OpenClaw like hiring a personal butler and giving them an office in your house:

- **The office** = your computer, where the Gateway (OpenClaw's server process) runs
- **The phone lines** = the chat channels (Telegram, WhatsApp, etc.) that connect the outside world to your butler's desk
- **The filing cabinet** = persistent memory, where your butler remembers everything about you
- **The toolbox** = built-in tools (browser, shell, files) that let your butler get things done
- **The training manual** = skills that teach your butler new abilities
- **The brain** = the AI model (Claude, GPT, etc.) that powers your butler's intelligence

The key insight: **OpenClaw is the office and infrastructure. The AI model is the brain you plug into it.**

#### How It Connects to Everything Else

OpenClaw is the **central hub** of everything you'll learn in this curriculum:

```
           ┌─────────────┐
           │  AI Model    │  ← Claude, GPT, local models
           │  (the brain) │
           └──────┬───────┘
                  │
    ┌─────────────┴─────────────┐
    │      OpenClaw Gateway      │  ← The always-on server
    │   (the office/platform)    │
    └─┬───┬───┬───┬───┬───┬───┬─┘
      │   │   │   │   │   │   │
      │   │   │   │   │   │   └── Skills & Plugins
      │   │   │   │   │   └────── Memory
      │   │   │   │   └────────── Tools (browser, shell, files)
      │   │   │   └────────────── Sessions
      │   │   └────────────────── Dashboard / Control UI
      │   └────────────────────── Chat Channels
      └────────────────────────── Automation (cron, webhooks)
```

#### Key Terms

| Term | Definition |
|------|-----------|
| **Gateway** | The always-running server process at the heart of OpenClaw |
| **Channel** | A connection to a chat app (Telegram, WhatsApp, etc.) |
| **Session** | A conversation context with history; resets daily by default |
| **Memory** | Persistent facts stored across sessions (your name, preferences, etc.) |
| **Skill** | A markdown instruction file that teaches the AI new behaviors |
| **Plugin** | A code extension (Node.js) that adds new tools or API endpoints |
| **Dashboard** | The web-based chat interface at `http://127.0.0.1:18789` |
| **Daemon** | A background system service that keeps the Gateway running even after you close the terminal |

#### Common Misconceptions

1. **"OpenClaw IS an AI model."** Wrong. OpenClaw is the *infrastructure* that connects AI models to your life. It uses Claude, GPT, or local models as its brain, but OpenClaw itself is the platform, the routing, the memory, the tools.

2. **"I need a powerful GPU."** Not usually. The AI model runs in the cloud (Anthropic/OpenAI). Your machine just runs the lightweight Gateway server. If you want to use a local model, then yes, you need GPU power — but that's optional.

3. **"It's just another chatbot."** No. A chatbot answers questions. OpenClaw takes *actions* — browsing the web, running code, managing files. It's an agent, not just a chat interface.

4. **"My data goes to OpenClaw's servers."** No. OpenClaw is self-hosted. Your data stays on your machine. API calls go to the model provider you choose (Anthropic, OpenAI, etc.), but OpenClaw as a project never receives your data.

#### Theory Check

1. In your own words, what is the relationship between OpenClaw and an AI model like Claude? Which is the "brain" and which is the "body"?
2. If OpenClaw runs on your machine, where do your conversations actually get processed — on your machine or somewhere else? (Hint: there are two parts to this answer.)
3. Why does OpenClaw exist as a separate process (the Gateway) rather than being built into each chat app directly?

---

### Part B — Hands-On: Your First Encounter

There is no hands-on work for this section — it is purely conceptual. Your first hands-on work begins with installation in the next section.

Before moving on, make sure you can explain to a friend what OpenClaw is, why it exists, and how it differs from ChatGPT in at least three specific ways.

---

## 1.2 Installation Methods & Prerequisites

### Part A — Theory & Conceptual Understanding

#### Why Multiple Installation Methods Exist

OpenClaw offers three ways to install, each designed for a different type of user. Understanding why these exist helps you choose the right one and troubleshoot issues later.

| Method | Who it's for | What you get | Tradeoff |
|--------|-------------|-------------|----------|
| **Installer script** (`curl ... \| bash`) | Most users, fastest start | One command does everything: installs Node.js if missing, installs OpenClaw, sets up the daemon | Least control over what happens |
| **npm global install** (`npm install -g openclaw`) | Developers who already have Node.js | Clean npm package install | You manage Node.js yourself |
| **Git clone (hackable mode)** | Contributors or tinkerers who want to modify OpenClaw's source code | Full source code, development workflow | Most complex, requires git and dev tools |

#### The Node.js Dependency

OpenClaw is built with Node.js (JavaScript runtime). This is a critical prerequisite:

- **Required version:** Node.js 22.14+ (Node 24 recommended)
- **Why Node.js?** It's excellent for the always-on, event-driven server that OpenClaw needs to be. It handles multiple simultaneous chat connections efficiently.

**Analogy:** Node.js is to OpenClaw what an engine is to a car. You don't interact with the engine directly — you interact with the car (OpenClaw). But the engine (Node.js) must be there and must be the right size.

#### What the Installer Script Actually Does

When you run `curl -fsSL https://openclaw.ai/install.sh | bash`, here's what happens under the hood:

1. Downloads a shell script from openclaw.ai
2. Checks if Node.js is installed and if it's the right version
3. If Node.js is missing or too old, installs it (typically via nvm or fnm)
4. Runs `npm install -g openclaw@latest` to install OpenClaw globally
5. Verifies the installation succeeded

Understanding this lets you troubleshoot when things go wrong — usually, it's a Node.js version issue.

#### What `npm install -g` Means

`npm install -g openclaw@latest` breaks down as:
- `npm` — Node Package Manager, the tool that downloads JavaScript packages
- `install` — download and set up
- `-g` — install **globally** (available as a command from anywhere, not just in one project folder)
- `openclaw@latest` — the package name and version tag

After this runs, the `openclaw` command becomes available in your terminal.

#### The Config File: `~/.openclaw/openclaw.json`

Everything about your OpenClaw setup is configured in a single JSON file. The `~` means your home directory:

- **macOS/Linux:** `/home/yourname/.openclaw/openclaw.json`
- **Windows:** `C:\Users\yourname\.openclaw\openclaw.json`

This file controls:
- Which AI model to use
- Which chat channels are active
- Security settings (who can talk to your assistant)
- Memory and session behavior
- Skills and plugin configuration

You don't need to edit this file manually for basic use — the onboarding wizard creates it for you. But understanding it exists and what it controls is essential for later phases.

#### Environment Variables

OpenClaw also reads some environment variables:
- `OPENCLAW_HOME` — overrides the home directory for internal path resolution
- `OPENCLAW_STATE_DIR` — overrides where state data is stored
- `OPENCLAW_CONFIG_PATH` — overrides the config file location

These are useful for advanced setups (like running multiple instances) but you can ignore them for now.

#### Theory Check

1. What is the minimum Node.js version required for OpenClaw, and why does OpenClaw need Node.js at all?
2. If you already have Node.js 24 installed, which installation method would you choose and why?
3. Where does OpenClaw store its configuration file, and what role does this file play in the system?

---

### Part B — Hands-On: Installing OpenClaw

#### Step-by-Step Installation (Installer Script Method — Recommended)

**Step 1:** Open your terminal.
- **macOS:** Press `Cmd + Space`, type "Terminal", press Enter
- **Linux:** Press `Ctrl + Alt + T` or find Terminal in your applications
- **Windows:** Open PowerShell as Administrator

**Step 2:** Check if Node.js is already installed.

```bash
node --version
```

If you see `v22.14.0` or higher, you're good. If not, the installer script will handle it.

**Step 3:** Run the installer.

macOS / Linux:
```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Windows (PowerShell):
```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

**Step 4:** Verify the installation.

```bash
openclaw --version
```

You should see the version number printed. If you see "command not found," your terminal may need to be restarted to pick up the new PATH entry.

#### Alternative: npm Method

If you already have Node.js 22.14+ installed:

```bash
npm install -g openclaw@latest
openclaw --version
```

#### Common Mistakes to Watch For

1. **Node.js version too old:** The most common issue. Run `node --version` to check. If it shows v18 or v20, you need to upgrade.
2. **Permission errors on npm install:** On Linux/macOS, you may see `EACCES` errors. Fix: use `sudo npm install -g openclaw@latest` or better, configure npm to use a different directory.
3. **"command not found" after install:** Your shell hasn't reloaded its PATH. Close and reopen your terminal, or run `source ~/.bashrc` (Linux) / `source ~/.zshrc` (macOS).
4. **Firewall blocking the download:** Corporate networks may block the download. You can download manually or use the npm method behind a proxy.

---

## 1.3 The Onboarding Wizard

### Part A — Theory & Conceptual Understanding

#### What Onboarding Actually Configures

The onboarding wizard (`openclaw onboard`) is not just a "welcome screen." It is a systematic setup process that configures **four critical subsystems**:

1. **Authentication with a Model Provider**
   - You choose your AI "brain" — Anthropic (Claude), OpenAI (GPT), Google (Gemini), or a local model
   - The wizard securely stores your API key in the config file
   - This is what lets OpenClaw actually think — without it, you have a server with no brain

2. **Gateway Configuration**
   - Sets the port (default: 18789)
   - Configures default security settings
   - Establishes the Gateway auth token (for remote access later)

3. **Daemon Installation** (with `--install-daemon` flag)
   - A daemon is a background process that runs automatically and restarts if it crashes
   - The daemon keeps your Gateway alive even after you close the terminal
   - On macOS: uses `launchd` (macOS's built-in service manager)
   - On Linux: uses `systemd` (Linux's standard service manager)
   - On Windows: uses Windows Services or a startup task

4. **Initial Channel Setup** (optional during onboarding)
   - You can optionally connect your first chat channel (e.g., Telegram)
   - This involves providing a bot token (Telegram) or scanning a QR code (WhatsApp)

#### Why `--install-daemon` Matters

Without the daemon flag, you'd have to manually start the Gateway every time you restart your computer. The daemon makes OpenClaw **always available** — like how your phone doesn't need to be manually "started" every morning.

**Analogy:** The Gateway is the butler; the daemon is the building that keeps the butler's office open 24/7, even after you go home. Without the daemon, the butler goes home when you close the terminal.

#### Theory Check

1. Name the four things the onboarding wizard configures. Why is each one necessary?
2. What happens if you run `openclaw onboard` without the `--install-daemon` flag? How would your experience differ?
3. What is the relationship between your API key and the AI model? What happens if you don't have one?

---

### Part B — Hands-On: Running the Onboarding Wizard

**Step 1:** Run onboarding with daemon installation.

```bash
openclaw onboard --install-daemon
```

**Step 2:** Follow the prompts carefully.

The wizard will ask you:
1. **Model provider** — Choose Anthropic (recommended for beginners; Claude is excellent for agentic tasks)
2. **API key** — Paste your API key when prompted. It's stored locally in `~/.openclaw/openclaw.json`
3. **Daemon** — Say yes to install and start the background service

**Step 3:** Verify the Gateway is running.

```bash
openclaw gateway status
```

You should see output indicating the Gateway is running, with `Runtime: running` and `RPC probe: ok`.

**Step 4:** Check the full system status.

```bash
openclaw status
```

This shows you: the Gateway version, listening port, connected channels, and active sessions.

#### Common Mistakes

- **Pasting the API key with extra whitespace:** Copy-paste carefully. Leading/trailing spaces in your API key will cause authentication failures with the model provider.
- **Choosing a model you don't have a key for:** If you pick "OpenAI" but paste an Anthropic key, nothing will work. Match the provider to the key.
- **Skipping the daemon:** If you skip it, you'll need to run `openclaw gateway` manually every time. You can add the daemon later with `openclaw install-daemon`.

---

## 1.4 The Gateway Architecture

### Part A — Theory & Conceptual Understanding

#### What the Gateway Is

The Gateway is the **central nervous system** of OpenClaw. It is a single, always-on Node.js process that:

1. **Receives messages** from all your connected channels (Telegram, WhatsApp, etc.) and the Dashboard
2. **Routes them** through sessions, memory injection, and the AI model
3. **Executes tool calls** when the AI decides to take action (browse, run commands, etc.)
4. **Sends responses** back through the channel the message came from

It is a **server** running on your machine, listening on a port (default: `18789`).

#### Why a Separate Server Process?

This is a key architectural decision that beginners often wonder about: "Why can't OpenClaw just be a browser extension or a simple script?"

The answer is: **because it needs to be always available from multiple places simultaneously.**

Think about it:
- You want to message your assistant from Telegram on your phone
- AND from WhatsApp on your tablet
- AND from the Dashboard on your laptop
- AND have it run scheduled tasks at 3 AM while you're asleep

The only way to serve all of these is an always-on server process. That's the Gateway.

**Analogy:** The Gateway is like a web server (Apache, Nginx). Your website needs a server to be available to anyone who visits. Similarly, your AI assistant needs a server to be available from any chat channel at any time.

#### The Runtime Model (How It Actually Works)

The Gateway runs as:

- **One always-on process** for routing, control plane, and channel connections
- **Single multiplexed port** that handles everything:
  - WebSocket connections for real-time control and RPC (Remote Procedure Calls)
  - HTTP APIs — including OpenAI-compatible endpoints (`/v1/chat/completions`, `/v1/models`, etc.)
  - The Control UI and Dashboard
  - Webhook endpoints for channels and automation

This is elegant: **one port, one process, everything multiplexed.** No need to manage multiple services.

#### The Agent Loop

When a message arrives, here's what happens inside the Gateway:

```
1. Message arrives (from Telegram, Dashboard, etc.)
       │
2. Gateway identifies the sender and session
       │
3. Memory is retrieved and injected into context
       │
4. Skills relevant to the conversation are loaded
       │
5. The full context (system prompt + memory + skills + history + new message)
   is sent to the AI model (Claude, GPT, etc.)
       │
6. The model responds — either with text or a tool call
       │
   ┌───┴────────────────────────┐
   │ Text response?             │ Tool call?
   │ Send it back to the user   │ Execute the tool (browser, shell, etc.)
   │ via the original channel   │ Send the result back to the model
   └────────────────────────────┘ Loop back to step 5
```

This loop — send context to model → get response → execute tools → loop — is the **agent loop**, and it's the fundamental operation of OpenClaw.

#### Default Bind Mode: Loopback

By default, the Gateway binds to `127.0.0.1` (loopback / localhost). This means:
- Only your machine can access the Dashboard
- External devices cannot reach it directly
- This is a **security feature** — your assistant isn't exposed to the internet

If you want remote access later (Phase 5), you'll learn about Tailscale and remote deployment.

#### Authentication

The Gateway requires authentication by default:
- `gateway.auth.token` or `gateway.auth.password` in config
- Or environment variables: `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`

This prevents anyone on your local network from accessing your assistant.

#### Theory Check

1. What is the Gateway, and why does it need to be a separate always-on process rather than a simple script?
2. Describe the agent loop in your own words. What happens when the AI model responds with a tool call vs. plain text?
3. What does "loopback binding" mean, and why is it the default for security?

---

### Part B — Hands-On: Exploring the Gateway

**Step 1:** Check the Gateway status.

```bash
openclaw gateway status
```

Note the output. You should see the port, runtime status, and RPC probe result.

**Step 2:** View the Gateway logs in real-time.

```bash
openclaw logs --follow
```

Leave this running in one terminal and open a new terminal for the next steps. Watch how log entries appear as you interact.

**Step 3:** Start the Gateway manually (verbose mode) to see what's happening.

```bash
# Stop the daemon first
openclaw gateway stop

# Start manually with verbose output
openclaw gateway --port 18789 --verbose
```

You'll see the initialization sequence: channel connections being established, skills being loaded, and the server binding to port 18789. Press `Ctrl+C` to stop when done, then restart the daemon:

```bash
openclaw gateway start
```

---

## 1.5 The Dashboard & Control UI

### Part A — Theory & Conceptual Understanding

#### Two Interfaces, Two Purposes

OpenClaw provides two web-based interfaces, both served from the same Gateway port:

1. **Dashboard** — The primary chat interface. This is where you talk to your assistant. Think of it as the "WhatsApp" inside your browser, but connected directly to your local Gateway.

2. **Control UI** — The administrative interface. This is where you configure channels, view sessions, manage skills, check system health, and monitor logs. Think of it as the "admin panel."

#### Why a Web Interface?

You might wonder: "If I'm going to talk to it through Telegram, why does the Dashboard exist?"

Good reasons:
- **Testing:** The Dashboard is the fastest way to test your setup before connecting any channels
- **Full control:** Some things (like config changes, session inspection, and debugging) are easier in a rich web UI
- **Local-only access:** The Dashboard works even when no external channels are connected
- **Development:** When writing skills or plugins, the Dashboard gives you the quickest feedback loop

#### Access Points

- **Local (default):** `http://127.0.0.1:18789`
- **Remote:** Via web surfaces or Tailscale (covered in Phase 5)

#### Theory Check

1. What is the difference between the Dashboard and the Control UI?
2. Why would you use the Dashboard even if you plan to primarily use Telegram?
3. What URL do you use to access the Dashboard by default?

---

### Part B — Hands-On: Using the Dashboard

**Step 1:** Open the Dashboard.

```bash
openclaw dashboard
```

This opens your default browser to `http://127.0.0.1:18789`. Alternatively, just navigate to that URL manually.

**Step 2:** Send your first message.

Type something simple:

> "Hello! What's your name, and what can you do?"

Watch the response. The AI will introduce itself and describe its capabilities. Notice the response appears in real-time (streaming).

**Step 3:** Test a tool call.

Ask something that requires a tool:

> "What time is it right now?"

The assistant may use a tool to check the system clock. In the logs (if you have them open), you'll see the tool call.

**Step 4:** Explore the Control UI.

Navigate through the settings and configuration panels. Find:
- Active sessions
- Connected channels (probably none yet, just the Dashboard)
- Model configuration
- System status

Don't change anything yet — just observe.

---

## Phase 1 Projects

---

### Project 1: "Hello World Agent" — Your First Functioning Assistant

#### Theory Recap
This project applies: Section 1.1 (what OpenClaw is), 1.2 (installation), 1.3 (onboarding), 1.4 (Gateway architecture), and 1.5 (Dashboard).

#### What You'll Build
A fully functional OpenClaw installation where you hold a multi-turn conversation through the Dashboard, testing basic agent capabilities.

#### What You'll Learn
- Confirming the Gateway is running and healthy
- Holding a multi-turn conversation
- Observing the agent loop in action through logs
- Using the `/status` command

#### Prerequisites
- OpenClaw installed and onboarded (completed in section 1.2–1.3)

#### Step-by-Step Build Guide

1. **Open two terminal windows.** In the first, run:
   ```bash
   openclaw logs --follow
   ```

2. **Open the Dashboard** in your browser:
   ```bash
   openclaw dashboard
   ```

3. **Have a structured conversation.** Send these messages in order:

   **Message 1:** "Hi! My name is Alex and I'm learning OpenClaw. Can you remember that?"

   *Expected response:* The AI will acknowledge your name and confirm it will remember.

   **Message 2:** "What is 2 + 2?"

   *Expected response:* "4" — a simple test that the model is responding.

   **Message 3:** "What's my name?"

   *Expected response:* "Alex" — confirming conversational context within the session.

   **Message 4:** Type `/status`

   *Expected response:* A status display showing context usage, current model, and active toggles.

4. **Check the logs.** In your log terminal, observe:
   - Messages being received
   - Context being assembled
   - Model API calls being made
   - Responses being sent back

5. **Check system status from the terminal:**
   ```bash
   openclaw status
   ```

#### How to Test It
- ✅ The Dashboard loads at `http://127.0.0.1:18789`
- ✅ The AI responds to messages
- ✅ `/status` shows model information
- ✅ Multi-turn context works (it remembers your name within the conversation)
- ✅ `openclaw status` shows "running"

#### Common Pitfalls
- **Gateway not running:** If the Dashboard won't load, run `openclaw gateway status` to check.
- **API key issues:** If the AI doesn't respond or gives errors, check your API key in `~/.openclaw/openclaw.json`.
- **Wrong port:** If you changed the port during onboarding, the Dashboard URL will be different.

#### Stretch Goals
1. Try switching models mid-session with `/new claude-sonnet-4-20250514` (or another model name) and notice the difference in responses.
2. Ask the AI to do something that requires a tool: "What files are in my home directory?" Observe how it uses the shell tool.
3. Ask a complex multi-step question: "Search the web for the weather in Tokyo, then write a haiku about it."

---

### Project 2: "System Explorer" — Understanding Your Gateway From Inside Out

#### Theory Recap
This project applies: Section 1.4 (Gateway architecture) deeply. You'll explore the Gateway process, configuration file, and runtime behavior hands-on.

#### What You'll Build
A comprehensive "system report" about your OpenClaw installation, generated by asking your AI assistant to examine its own infrastructure.

#### What You'll Learn
- Reading and understanding `openclaw.json`
- Using CLI commands for system introspection
- Understanding the relationship between config, state, and runtime

#### Prerequisites
- Completed Project 1 (working installation with Dashboard access)

#### Step-by-Step Build Guide

1. **Examine the config file.** In your terminal:
   ```bash
   cat ~/.openclaw/openclaw.json
   ```

   Read through it. Identify:
   - Your model provider and model name
   - The Gateway port
   - Any channel configurations
   - Security settings

2. **Ask your assistant about itself** via the Dashboard:

   > "Can you read and summarize the contents of ~/.openclaw/openclaw.json? What model are you using?"

   The assistant will use the file system tool to read its own config file and explain it to you.

3. **Explore the state directory:**
   ```bash
   ls -la ~/.openclaw/
   ```

   You'll see directories like `agents/`, `skills/`, and files like `openclaw.json`.

4. **Check channel readiness:**
   ```bash
   openclaw channels status --probe
   ```

5. **Generate a system report.** Ask the Dashboard:

   > "Please run `openclaw status`, `openclaw gateway status`, and `node --version`, then give me a formatted report about my OpenClaw system health."

#### How to Test It
- ✅ You can read and partially understand `openclaw.json`
- ✅ Your assistant can read its own config
- ✅ You can identify the model provider, port, and state directory
- ✅ `openclaw channels status --probe` runs without errors

#### Common Pitfalls
- **Config file has trailing commas:** JSON doesn't allow trailing commas. If you edit the file manually, validate it.
- **Confusing `~/.openclaw/` with `~/.agents/`:** These are different directories. OpenClaw's system files are in `~/.openclaw/`. Personal agent skills are in `~/.agents/skills/`.

#### Stretch Goals
1. Find the session transcript file in `~/.openclaw/agents/` and examine the raw JSONL format.
2. Start the Gateway with `--verbose` and identify each startup phase in the logs.
3. Experiment with `openclaw logs --follow` while sending messages and map each log line to a step in the agent loop from Section 1.4.

---

### Project 3: "Config Tinkerer" — Your First Configuration Changes

#### Theory Recap
This project applies: Section 1.2 (config file), 1.3 (onboarding), 1.4 (Gateway), and 1.5 (Dashboard/Control UI).

#### What You'll Build
A customized OpenClaw configuration with personalized system prompt instructions and adjusted session settings.

#### What You'll Learn
- Editing `openclaw.json` safely
- Understanding the config → restart → verify cycle
- Adding custom instructions to your assistant's personality

#### Prerequisites
- Completed Projects 1 and 2

#### Step-by-Step Build Guide

1. **Back up your config:**
   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup
   ```

2. **Open the config in an editor:**
   ```bash
   nano ~/.openclaw/openclaw.json
   # or: code ~/.openclaw/openclaw.json (if you use VS Code)
   ```

3. **Add a custom system instruction.** Find or add a `system` or `agents` section and add a custom instruction message. For example, add to your config:
   ```json
   {
     "agents": {
       "defaults": {
         "instructions": "You are a helpful assistant named Claw. You always respond in a friendly, concise manner. When you're not sure about something, you say so honestly."
       }
     }
   }
   ```

4. **Save and restart the Gateway:**
   ```bash
   openclaw gateway restart
   ```

5. **Test your changes** in the Dashboard:

   > "What's your name?"

   The response should reflect your custom instructions.

6. **Try adjusting another setting.** Change the session idle reset:
   ```json
   {
     "session": {
       "reset": {
         "idleMinutes": 60
       }
     }
   }
   ```

7. **Restart and verify:**
   ```bash
   openclaw gateway restart
   openclaw status
   ```

#### How to Test It
- ✅ Your assistant responds according to your custom instructions
- ✅ `openclaw gateway restart` succeeds without errors
- ✅ `/status` in chat shows the updated configuration

#### Common Pitfalls
- **Invalid JSON:** The #1 cause of startup failures after config edits. Use a JSON validator or `python3 -m json.tool ~/.openclaw/openclaw.json` to check syntax.
- **Forgetting to restart:** Config changes don't take effect until the Gateway restarts.
- **Overwriting the whole file:** Always make targeted edits, don't replace the entire file.

#### Stretch Goals
1. Set the session to never auto-reset (remove the daily 4 AM reset) and observe how the conversation persists indefinitely.
2. Change the model from Claude to GPT (or vice versa) and notice the difference in response style.
3. Explore the hot-reload modes: set `gateway.reload.mode` to `"hybrid"` and see if config changes take effect without a full restart.

---

## Phase 1 — End of Phase Review

### Conceptual Recap Questions

1. Explain the relationship between OpenClaw, the Gateway, the Dashboard, and the AI model. How do they all connect?
2. What is the agent loop? Walk through what happens when you send a message from the Dashboard to the time you see a response.
3. Why does OpenClaw use a daemon? What problem does it solve?
4. Where is the config file stored, and what are three important things it controls?
5. What is the difference between sessions and memory? (Answer based on what you've observed so far — we'll cover memory deeply in Phase 3.)

### Practical Consolidation Challenge

**"The Self-Documenting Agent"**: Ask your OpenClaw assistant (through the Dashboard) to write you a one-page summary of how it works. Tell it:

> "You are an OpenClaw assistant. Please explain — to a beginner who has never used you before — what you are, where you're running, what model you're using, and what you can do. Be specific: tell me your port, your config file location, and demonstrate one tool by checking the current date."

Read its response critically. Does it match what you learned in this phase? If anything is wrong or incomplete, you now know enough to spot the gaps. That's the mark of understanding.

---

**Congratulations!** You've completed Phase 1. You now have a mental model of what OpenClaw is, a running installation, and hands-on experience talking to your assistant. In Phase 2, you'll connect it to the outside world through chat channels.
