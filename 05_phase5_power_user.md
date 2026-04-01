# Phase 5: Power User — Multi-Agent, Remote Deployment & Advanced Security

## Phase Overview

**Big-picture goal:** By the end of this phase, you'll operate OpenClaw at an advanced level. You'll understand multi-agent architectures, deploy OpenClaw to remote servers, configure the macOS companion app, and implement a complete security model.

**What you'll understand:**
- Multi-agent routing, sub-agents, and workspaces as a system design concept
- The gateway-as-server model for remote/cloud deployment
- The macOS companion app architecture
- The formal security model including gateway lock, OAuth, tool policies, and elevated mode

**What you'll be able to do:**
- Set up multi-agent configurations with specialized agents
- Deploy OpenClaw to a remote server (Railway, Docker, VPS)
- Configure and use the macOS companion app features
- Implement a complete, production-grade security configuration

---

## What You Need Before This Phase

- [ ] Completed Phases 1–4 (full local setup with skills and automation)
- [ ] A remote server or cloud account for deployment (Railway, Render, Fly.io, or any VPS)
- [ ] Optional: a Mac for the companion app section

---

## 5.1 Multi-Agent Setups

### Part A — Theory & Conceptual Understanding

#### The Multi-Agent Concept

Until now, you've been using a single agent — one AI personality with one set of skills and one configuration. OpenClaw supports **multiple agents** running inside the same Gateway. Each agent can have:

- Its own model (e.g., one agent uses Claude Sonnet, another uses GPT-4o)
- Its own skills (a coding agent vs. a creative writing agent)
- Its own sessions and memory
- Its own tools and permissions

**Analogy:** Think of a company with different departments. The front desk (Gateway) routes your call to the right department (agent). The Engineering department has different tools and knowledge than the Marketing department, but they share the same building.

#### Agent Routing

When a message arrives, the Gateway decides which agent handles it based on:
- **Channel routing rules** — "Telegram DMs go to agent-main, Discord goes to agent-gaming"
- **Explicit switching** — You type `/agent coding` to switch to the coding agent
- **Sub-agent delegation** — A main agent can delegate tasks to specialized sub-agents

```
┌──────────────────────────────────────────┐
│                 Gateway                   │
│                                           │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐ │
│  │ Agent:   │  │ Agent:   │  │ Agent:   │ │
│  │ main     │  │ coding   │  │ research │ │
│  │ Claude   │  │ Claude   │  │ GPT-4o   │ │
│  │ Sonnet   │  │ Sonnet   │  │          │ │
│  │          │  │ +coding  │  │ +browser │ │
│  │ general  │  │  skills  │  │  skills  │ │
│  └─────────┘  └──────────┘  └──────────┘ │
└──────────────────────────────────────────┘
```

#### Workspaces and Isolation

Each agent operates within a **workspace** — a directory context that defines:
- Which project files are accessible
- Which workspace-level skills load
- How sessions are stored
- Tool permissions and sandboxing rules

Multi-agent setups let you have:
- A general assistant for daily life questions
- A coding agent with access to your source code
- A research agent focused on web browsing and data gathering
- A home automation agent controlling smart devices

All sharing the same Gateway but fully isolated in terms of knowledge and permissions.

#### Theory Check

1. Why would you want multiple agents instead of one agent that does everything?
2. How does agent routing work? What determines which agent handles a message?
3. What is a workspace, and how does it relate to agent isolation?

---

### Part B — Hands-On: Setting Up a Multi-Agent Configuration

**Step 1:** Configure a second agent in `openclaw.json`:

```json
{
  "agents": {
    "main": {
      "model": "claude-sonnet-4-20250514",
      "instructions": "You are a helpful general assistant."
    },
    "coding": {
      "model": "claude-sonnet-4-20250514",
      "instructions": "You are a senior software engineer. Always provide production-ready code with tests and documentation.",
      "workspace": "~/projects"
    }
  }
}
```

**Step 2:** Restart and test switching:
```bash
openclaw gateway restart
```

**Step 3:** In the Dashboard, switch agents:
> `/agent coding`
> "Write me a Python REST API with FastAPI"

> `/agent main`
> "What's a good recipe for dinner tonight?"

Notice how each agent has different behavior based on its instructions.

---

## 5.2 Gateway Configuration Deep Dive

### Part A — Theory & Conceptual Understanding

#### The openclaw.json Structure

The config file is the single source of truth for all Gateway behavior. Major sections:

```json
{
  "gateway": { /* port, auth, reload, bind */ },
  "agents": { /* agent definitions, defaults */ },
  "channels": { /* telegram, whatsapp, discord, etc. */ },
  "session": { /* dmScope, reset, maintenance */ },
  "skills": { /* load dirs, entries, allowBundled */ },
  "providers": { /* anthropic, openai, google keys */ },
  "messages": { /* groupChat patterns */ },
  "browser": { /* enabled, headless mode */ },
  "automation": { /* cron, heartbeat configs */ }
}
```

#### Environment Variables

All configuration can also be set via environment variables (useful for Docker/cloud deployments):
- `OPENCLAW_GATEWAY_TOKEN` — Gateway auth token
- `OPENCLAW_GATEWAY_PASSWORD` — Gateway auth password
- `ANTHROPIC_API_KEY` — Anthropic API key
- `OPENAI_API_KEY` — OpenAI API key
- `OPENCLAW_CONFIG_PATH` — override config file location

#### Hot Reload Modes

The Gateway can reload configuration without a full restart:
- `gateway.reload.mode = "hybrid"` — reloads most config changes without restart
- Some changes (port, auth) still require a full restart

#### Theory Check

1. Name five top-level sections of openclaw.json and what each controls.
2. When would you use environment variables instead of the config file?
3. What is hot reload, and which config changes can it handle?

---

### Part B — Hands-On: Advanced Configuration

**Step 1:** Create a comprehensive config. Back up first:
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup
```

**Step 2:** Enable hot reload:
```json
{
  "gateway": {
    "reload": {
      "mode": "hybrid"
    }
  }
}
```

**Step 3:** Test hot reload by changing a non-critical setting (like instructions) and observing the change without restarting.

---

## 5.3 Remote Deployment

### Part A — Theory & Conceptual Understanding

#### The Gateway-as-Server Model

OpenClaw was designed from the start to work as a headless server. This means you can run it on:
- A cloud VPS (Hetzner, DigitalOcean, AWS, etc.)
- A Platform-as-a-Service (Railway, Render, Fly.io)
- A Raspberry Pi on your home network
- A Docker container in any environment

When deployed remotely, the Gateway becomes a **24/7 service** that doesn't depend on your laptop being open.

#### Why Deploy Remotely?

| Scenario | Local | Remote |
|----------|-------|--------|
| Laptop closed | ❌ Gateway stops | ✅ Gateway keeps running |
| Internet outage at home | ❌ Channels disconnect | ✅ Server stays online |
| Multiple devices | ✅ Works via channels | ✅ Works via channels |
| Latency | Low (same machine) | Medium (network hop) |
| Cost | Free (your hardware) | $$$ (server costs) |

#### Security for Remote Deployments

When the Gateway is remote, security is CRITICAL:
- **Always use Gateway auth tokens** — without auth, anyone who finds your port can control your AI
- **Use HTTPS** — either via reverse proxy (Nginx, Caddy) or the platform's built-in TLS
- **Bind to loopback** if behind a reverse proxy — never expose the raw port to the internet
- **Use Tailscale** for private network access — no public exposure needed

#### Docker Deployment

```dockerfile
FROM node:24-slim
RUN npm install -g openclaw@latest
COPY openclaw.json /root/.openclaw/openclaw.json
EXPOSE 18789
CMD ["openclaw", "gateway"]
```

#### Theory Check

1. What are two advantages of remote deployment over running locally?
2. Why is authentication even more critical for remote deployments?
3. What is Tailscale, and how does it solve the remote access security problem?

---

### Part B — Hands-On: Docker Deployment

**Step 1:** Create a Dockerfile:
```bash
mkdir ~/openclaw-deploy
cd ~/openclaw-deploy

cat > Dockerfile << 'EOF'
FROM node:24-slim
RUN npm install -g openclaw@latest
COPY openclaw.json /root/.openclaw/openclaw.json
EXPOSE 18789
CMD ["openclaw", "gateway", "--port", "18789"]
EOF
```

**Step 2:** Create a deployment config (remove sensitive keys from version control):
```bash
cp ~/.openclaw/openclaw.json ./openclaw.json
# Edit to add Gateway auth token:
# "gateway": { "auth": { "token": "your-secure-token" } }
```

**Step 3:** Build and run:
```bash
docker build -t openclaw-server .
docker run -d -p 18789:18789 --name openclaw openclaw-server
```

**Step 4:** Verify:
```bash
curl http://localhost:18789/v1/models -H "Authorization: Bearer your-secure-token"
```

---

## 5.4 Advanced Security

### Part A — Theory & Conceptual Understanding

#### The Complete Security Model

OpenClaw's security model has multiple layers, from outermost to innermost:

```
Layer 1: Network binding (loopback by default)
  │
  └─ Layer 2: Gateway authentication (token/password)
       │
       └─ Layer 3: Channel allowlists and pairing
            │
            └─ Layer 4: DM scope isolation
                 │
                 └─ Layer 5: Tool policies and sandboxing
                      │
                      └─ Layer 6: Elevated mode for dangerous operations
```

#### Gateway Lock

The Gateway can be "locked" — preventing configuration changes via the API:
- Useful when the Gateway is deployed remotely and you don't want anyone (including the AI) changing the config
- Config changes must be made by editing the file directly and restarting

#### Tool Policies

You can restrict which tools the AI can use:
- Disable shell access entirely for untrusted users
- Restrict file system access to specific directories
- Disable browser for certain agents
- Require confirmation for destructive operations

#### Docker Sandboxing

For maximum security, OpenClaw can run tool calls inside a Docker container:
- The AI's shell commands execute in an isolated container
- No access to the host file system (unless explicitly mounted)
- Network access can be restricted
- Even if the AI is tricked into running malicious commands, they're contained

#### Security Audit

```bash
openclaw security audit
```

This command reviews your configuration and flags potential security issues.

#### Theory Check

1. Name all six layers of the security model from outermost to innermost.
2. When would you use Docker sandboxing? What does it protect against?
3. What does `openclaw security audit` check?

---

### Part B — Hands-On: Security Hardening

**Step 1:** Run a security audit:
```bash
openclaw security audit
```

**Step 2:** Address any findings.

**Step 3:** Test your security by trying to access the Gateway without auth:
```bash
curl http://localhost:18789/v1/models
# Should fail without auth token
```

---

## 5.5 macOS Companion App

### Part A — Theory & Conceptual Understanding

#### How the Companion App Works

The macOS companion app is a native macOS application that wraps the Gateway. It provides:
- **Menu bar icon** — quick access to Gateway status, start/stop, and settings
- **Voice wake** — "Hey Claw" voice activation
- **Voice overlay** — dictate to your assistant by voice
- **Peekaboo (visual context)** — share your screen with the assistant for visual understanding
- **Skills UI** — graphical skill management

The companion app is not a separate system — it runs the same Gateway as the CLI install, just with a native macOS UI wrapper.

#### Theory Check

1. Does the macOS companion app run a different Gateway than the CLI install?
2. What is Peekaboo, and what kind of tasks would it be useful for?
3. What advantage does the menu bar icon provide over running the Gateway from the terminal?

---

### Part B — Hands-On: Companion App (macOS only)

If you're on macOS, download from: https://github.com/openclaw/openclaw/releases/latest

Follow the setup wizard — it will detect your existing configuration if you've already installed via CLI.

---

## Phase 5 Projects

---

### Project 1: "The Agent Team" — Multi-Agent Specialization

#### Theory Recap
Applies Section 5.1 (multi-agent setups).

#### What You'll Build
A three-agent setup with specialized agents for coding, research, and personal tasks.

#### What You'll Learn
- Multi-agent configuration and routing
- Agent-specific skills and instructions
- Switching between agents contextually

#### Prerequisites
- Completed Phase 4

#### Step-by-Step Build Guide

1. **Design three agents:**
   - `main` — general assistant, friendly tone
   - `coder` — software engineer, produces production code
   - `researcher` — web research specialist, thorough and cited

2. **Configure in openclaw.json:**
   ```json
   {
     "agents": {
       "main": {
         "model": "claude-sonnet-4-20250514",
         "instructions": "You are a friendly personal assistant named Claw."
       },
       "coder": {
         "model": "claude-sonnet-4-20250514",
         "instructions": "You are a senior full-stack engineer. Always write production-ready code with error handling, types, and tests. Use modern best practices."
       },
       "researcher": {
         "model": "claude-sonnet-4-20250514",
         "instructions": "You are a thorough research analyst. Always cite sources. Provide balanced perspectives. Structure findings with executive summary, key findings, and methodology."
       }
     }
   }
   ```

3. **Add agent-specific skills** by putting skills in agent-specific workspace directories.

4. **Test each agent** with appropriate tasks and verify they produce different output styles.

5. **Test delegation:** Ask the main agent to hand off a coding question to the coder agent.

#### How to Test It
- ✅ Switching between agents with `/agent <name>` works
- ✅ Each agent responds with its specialized style
- ✅ Sessions are isolated between agents

#### Stretch Goals
1. Set up channel-based routing: Telegram → main, Discord → coder.
2. Create a "router" skill for the main agent that detects coding vs. research questions and suggests the right agent.
3. Add a fourth agent for creative writing with a completely different model.

---

### Project 2: "Cloud Claw" — Remote Docker Deployment

#### Theory Recap
Applies Section 5.3 (remote deployment) and 5.4 (security).

#### What You'll Build
A fully deployed, secured OpenClaw Gateway running in Docker on a remote server.

#### What You'll Learn
- Docker containerization for OpenClaw
- Remote deployment security best practices
- Persistent data management in containers

#### Prerequisites
- Docker installed locally for testing
- A remote server or cloud platform account

#### Step-by-Step Build Guide

1. **Create the deployment directory:**
   ```bash
   mkdir ~/openclaw-cloud && cd ~/openclaw-cloud
   ```

2. **Write the Dockerfile:**
   ```dockerfile
   FROM node:24-slim
   RUN npm install -g openclaw@latest
   RUN mkdir -p /root/.openclaw
   EXPOSE 18789
   CMD ["openclaw", "gateway", "--port", "18789"]
   ```

3. **Write docker-compose.yml:**
   ```yaml
   version: '3.8'
   services:
     openclaw:
       build: .
       ports:
         - "18789:18789"
       volumes:
         - openclaw-data:/root/.openclaw
       environment:
         - OPENCLAW_GATEWAY_TOKEN=${GATEWAY_TOKEN}
         - ANTHROPIC_API_KEY=${ANTHROPIC_KEY}
       restart: unless-stopped
   volumes:
     openclaw-data:
   ```

4. **Create a `.env` file (never commit this):**
   ```
   GATEWAY_TOKEN=your-secure-random-token
   ANTHROPIC_KEY=sk-ant-your-key
   ```

5. **Test locally:**
   ```bash
   docker-compose up -d
   curl http://localhost:18789/v1/models -H "Authorization: Bearer your-secure-random-token"
   ```

6. **Deploy to your remote server** and verify from your phone via Telegram.

#### How to Test It
- ✅ Gateway responds to authenticated requests
- ✅ Unauthenticated requests are rejected
- ✅ Telegram messages work through the remote Gateway
- ✅ Gateway persists data across container restarts

#### Stretch Goals
1. Add Tailscale to the container for secure private access.
2. Set up a reverse proxy with automatic HTTPS (Caddy or Nginx + Let's Encrypt).
3. Deploy to Railway or Fly.io using their container deployment workflow.

---

### Project 3: "Fort Knox" — Production Security Configuration

#### Theory Recap
Applies Section 5.4 (advanced security) comprehensively.

#### What You'll Build
A fully hardened OpenClaw deployment that passes all security audit checks.

#### What You'll Learn
- Implementing all six security layers
- Security audit remediation
- Principle of least privilege for AI agents

#### Prerequisites
- Working Gateway (local or remote)

#### Step-by-Step Build Guide

1. **Run initial audit:**
   ```bash
   openclaw security audit
   ```

2. **Fix every finding.** Document each fix.

3. **Implement all layers:**
   - ✅ Loopback binding (or Tailscale for remote)
   - ✅ Gateway auth token
   - ✅ DM allowlists for all channels
   - ✅ Per-channel-peer DM scope
   - ✅ Tool restrictions for untrusted inputs
   - ✅ Elevated mode enabled for destructive operations

4. **Re-run audit and achieve clean result.**

5. **Penetration test:** Try to bypass each layer and document whether it holds.

#### How to Test It
- ✅ `openclaw security audit` passes with no critical findings
- ✅ Unauthorized access attempts are blocked
- ✅ Destructive commands require confirmation
- ✅ Documentation of all security measures is complete

#### Stretch Goals
1. Set up Docker sandboxing and verify tool calls run in isolation.
2. Create a "security policy" document and ask your assistant to enforce it.
3. Configure separate tool permission levels for different agents.

---

## Phase 5 — End of Phase Review

### Conceptual Recap Questions

1. When would a multi-agent setup be better than one agent with many skills?
2. What are the security implications of deploying the Gateway to a public cloud server?
3. Walk through all six layers of the security model and explain what each prevents.
4. How does Docker sandboxing protect your system from a compromised AI session?
5. What role does Tailscale play in remote access security?

### Practical Consolidation Challenge

**"The Remote Multi-Agent Hub"**: Deploy a Gateway with at least two agents to a Docker container. Configure strict security. Verify you can reach both agents from Telegram, with proper DM isolation, from your phone while the laptop is closed.

---

**Phase 5 complete!** You're now a power user. In Phase 6, you'll achieve mastery: building real integrations, contributing to the ecosystem, and understanding OpenClaw at the source level.
