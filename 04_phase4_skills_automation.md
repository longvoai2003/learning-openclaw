# Phase 4: Skills, Plugins & Automation

## Phase Overview

**Big-picture goal:** By the end of this phase, you'll understand how to extend OpenClaw beyond its built-in capabilities. You'll write custom skills, install community skills, understand plugins, and set up automation that runs without your involvement.

**What you'll understand:**
- The skills architecture: what a skill is, how SKILL.md works, loading precedence, gating
- How plugins differ from skills and extend the system with code
- The automation model: cron jobs, heartbeats, webhooks, polling
- Model provider management: costs, capabilities, switching

**What you'll be able to do:**
- Write custom skills from scratch
- Install and manage community skills via ClawHub
- Configure plugins for extended functionality
- Set up scheduled automation and event-driven workflows
- Switch between model providers strategically

---

## What You Need Before This Phase

- [ ] Completed Phase 3 (memory, sessions, and tools working)
- [ ] Familiarity with markdown (skills are markdown files)
- [ ] Optional: basic JavaScript/Node.js knowledge for plugins

---

## 4.1 The Skills System

### Part A — Theory & Conceptual Understanding

#### What a Skill Is, Architecturally

A **skill** is a markdown file (`SKILL.md`) that contains **instructions for the AI model**. When loaded, the skill's content is injected into the system prompt — the invisible instructions the AI reads before every conversation turn.

**Key insight:** Skills don't add new code or tools to the system. They add **knowledge and behavioral instructions** to the AI's prompt. A skill teaching the AI to "generate Python unit tests" doesn't install a testing framework — it gives the AI detailed instructions on *how* to write tests when asked.

**Analogy:** A skill is like a training manual for a new employee. The employee (AI) already knows how to work. The manual (skill) teaches them specific procedures, formats, and domain knowledge for their particular role.

#### SKILL.md Format and Frontmatter

Every skill is a `SKILL.md` file with YAML frontmatter at the top:

```markdown
---
name: daily-reporter
description: Generate daily summary reports from various data sources
metadata: { "openclaw": { "emoji": "📊" } }
---

When asked to create a daily report, follow this procedure:

1. Check today's date and time
2. Gather data from the requested sources
3. Format the report with these sections:
   - Executive Summary (2-3 sentences)
   - Key Metrics (bullet points)
   - Action Items (numbered list)
4. Save to ~/reports/daily_YYYY-MM-DD.md

Always use markdown formatting. Include timestamps on all reports.
```

**Required frontmatter:**
- `name` — unique identifier for the skill
- `description` — what the skill does (shown in UI and used by the AI for context)

**Optional frontmatter keys:**
- `metadata` — JSON object for OpenClaw-specific gating, install specs, emoji
- `homepage` — URL shown in the macOS Skills UI
- `user-invocable` — `true`/`false` (default: `true`). When `true`, exposed as a slash command
- `disable-model-invocation` — `true`/`false`. When `true`, excluded from the model's prompt
- `command-dispatch` — set to `tool` to bypass the model and dispatch directly to a tool
- `command-tool` — which tool to invoke with command-dispatch
- `command-arg-mode` — `raw` (default) forwards raw args to the tool

#### Locations and Precedence

Skills can live in multiple places. **Higher in this list = higher precedence:**

1. **Extra skill folders** — `skills.load.extraDirs` in config (lowest precedence)
2. **Bundled skills** — shipped with the OpenClaw install
3. **Managed/local skills** — `~/.openclaw/skills`
4. **Personal agent skills** — `~/.agents/skills`
5. **Project agent skills** — `<workspace>/.agents/skills`
6. **Workspace skills** — `<workspace>/skills` (highest precedence)

**Why precedence matters:** If two skills have the same name, the one with higher precedence wins. This lets you override bundled skills with your own versions.

**Per-agent vs. shared:**
- **Workspace skills** (`<workspace>/skills`) — only for that specific agent/workspace
- **Personal agent skills** (`~/.agents/skills`) — apply across all workspaces on your machine
- **Shared skills** (`~/.openclaw/skills`) — visible to all agents

#### Gating: Load-Time Filters

Gating controls **when a skill is eligible to load.** The `metadata.openclaw` section can specify requirements:

```yaml
metadata: {
  "openclaw": {
    "requires": {
      "bins": ["ffmpeg"],
      "env": ["SPOTIFY_API_KEY"],
      "config": ["browser.enabled"]
    },
    "os": ["darwin", "linux"],
    "primaryEnv": "SPOTIFY_API_KEY"
  }
}
```

- `requires.bins` — binaries that must exist on PATH
- `requires.env` — environment variables that must be set
- `requires.config` — `openclaw.json` values that must be truthy
- `os` — restrict to certain platforms (darwin/linux/win32)

If a gate check fails, the skill is **silently skipped** — no error, it just doesn't load. This prevents skills that need tools you don't have from cluttering the prompt.

#### Token Impact

Every loaded skill adds tokens to the system prompt. The skill's full markdown content is included. This means:
- A 500-word skill ≈ 650 tokens per conversation turn
- 10 skills loaded = potentially 6,500+ tokens of system prompt
- This leaves fewer tokens for conversation history and tool results

**Best practice:** Keep skills focused and concise. Disable skills you don't use. Use gating to prevent irrelevant skills from loading.

#### ClawHub: Community Skills Marketplace

[ClawHub](https://clawhub.com) is the community marketplace for OpenClaw skills. Key commands:

```bash
# Install a skill from ClawHub
openclaw skills install <skill-slug>

# Update all installed skills
openclaw skills update --all

# Sync and publish your own skills
clawhub sync --all
```

Installed skills go into `~/.openclaw/skills` (managed skills location).

#### Theory Check

1. What is a skill architecturally? Does it add new code/tools, or something else?
2. If you have two skills with the same name — one in `~/.openclaw/skills` and one in `<workspace>/skills` — which one wins and why?
3. Why does gating exist? What problem does it solve for a skill that requires `ffmpeg`?

---

### Part B — Hands-On: Writing Your First Skill

**Step 1:** Create a skills directory:
```bash
mkdir -p ~/.openclaw/skills/daily-greeting
```

**Step 2:** Write your first SKILL.md:
```bash
cat > ~/.openclaw/skills/daily-greeting/SKILL.md << 'EOF'
---
name: daily-greeting
description: Greet the user warmly and provide a motivational quote at the start of each conversation
---

When starting a new conversation with the user, or when the user says "good morning":

1. Greet them by name (check memory for their name)
2. Mention the current day and date
3. Share a unique motivational quote (never repeat the same one)
4. Ask what they'd like to focus on today

Keep the greeting concise — no more than 4 lines.
EOF
```

**Step 3:** Restart the Gateway to pick up the new skill:
```bash
openclaw gateway restart
```

**Step 4:** Test it. Start a new session (`/new`) and say:
> "Good morning!"

The AI should follow your skill's instructions.

---

## 4.2 Plugins — Code-Level Extensions

### Part A — Theory & Conceptual Understanding

#### How Plugins Differ From Skills

| | Skills | Plugins |
|---|---|---|
| **Format** | Markdown (SKILL.md) | JavaScript/TypeScript (Node.js modules) |
| **What they do** | Add instructions to the AI's prompt | Add new tools, API endpoints, and runtime behavior |
| **Runtime** | No code execution | Code runs inside the Gateway process |
| **Complexity** | Simple — just write markdown | Complex — requires programming |
| **Use case** | Teaching behaviors | Adding capabilities |

**Analogy:** Skills are like giving someone a cookbook (instructions). Plugins are like installing a new appliance in the kitchen (new capabilities).

#### Plugin Architecture

A plugin is a Node.js module that exports a `definePluginEntry` function:

```javascript
// openclaw.plugin.json marks this as a plugin
// The plugin registers with the Gateway API:

export function definePluginEntry({ register }) {
  register(api => {
    // Register new HTTP routes
    api.registerHttpRoute('/my-webhook', handler);

    // Register new tools
    api.registerTool('my-custom-tool', toolHandler);

    // Subscribe to events
    api.on('message', messageHandler);
  });
}
```

Plugins can:
- Add new HTTP endpoints to the Gateway
- Register custom tools the AI can use
- Subscribe to Gateway events (messages, session changes, etc.)
- Integrate with external services

#### Theory Check

1. What is the fundamental difference between a skill and a plugin?
2. Can a plugin add a new tool that the AI can call? Can a skill?
3. Why would you choose to write a plugin instead of a skill?

---

### Part B — Hands-On: Understanding Plugins

For now, we'll explore plugins conceptually and examine existing ones. Writing a full plugin is an advanced topic covered in Phase 6.

**Step 1:** Check if you have any plugins installed:
```bash
ls ~/.openclaw/plugins/ 2>/dev/null || echo "No plugins directory yet"
```

**Step 2:** Explore bundled plugin documentation:
> "What plugins are available for OpenClaw? What do they do?"

---

## 4.3 Model Providers

### Part A — Theory & Conceptual Understanding

#### How OpenClaw Abstracts Over Providers

OpenClaw doesn't care which AI model you use — it abstracts over all of them through a unified provider interface. You can use:

| Provider | Models | Strengths | Cost Tier |
|----------|--------|-----------|-----------|
| **Anthropic** | Claude Sonnet, Claude Haiku, Claude Opus | Best at agentic tasks, tool use, coding | Medium-High |
| **OpenAI** | GPT-4o, GPT-4o-mini, o1/o3 | Wide ecosystem, strong general performance | Medium-High |
| **Google** | Gemini Pro, Gemini Flash | Good multimodal, competitive pricing | Medium |
| **Local models** | Ollama, llama.cpp | Privacy, no ongoing cost | Free (but needs GPU) |

#### When to Switch Models

- **Complex tasks** (multi-step research, coding): Use Claude Sonnet or GPT-4o
- **Simple queries** (quick questions, translations): Use Claude Haiku or GPT-4o-mini (cheaper)
- **Privacy-critical work:** Use a local model
- **Cost management:** Monitor your API usage and switch to cheaper models for routine tasks

#### Switching Models

In chat: `/new claude-sonnet-4-20250514` or `/new gpt-4o`

In config:
```json
{
  "providers": {
    "anthropic": {
      "apiKey": "sk-ant-..."
    }
  },
  "agents": {
    "defaults": {
      "model": "claude-sonnet-4-20250514"
    }
  }
}
```

#### Theory Check

1. Can you use OpenClaw with a model running entirely on your machine? What are the tradeoffs?
2. Why would you use Claude Haiku instead of Claude Sonnet for some tasks?
3. How do you switch models mid-conversation?

---

### Part B — Hands-On: Switching Providers

**Step 1:** Check your current model:
Type `/status` in chat and note the model.

**Step 2:** Switch models within a session:
> `/new gpt-4o` 

(If you have an OpenAI key configured.) Ask the same question to both models and compare.

**Step 3:** Compare outputs. Ask both models:
> "Write a Python function that finds the longest palindrome in a string. Include comments."

Note differences in coding style, explanation, and approach.

---

## 4.4 Automation: Cron, Heartbeats, Webhooks & Events

### Part A — Theory & Conceptual Understanding

#### The Event-Driven Architecture

OpenClaw's automation system is built on an **event-driven architecture**. Instead of you always initiating conversations, the system can trigger actions based on:

1. **Cron jobs** — tasks that run on a schedule (e.g., "every morning at 8 AM")
2. **Heartbeats** — periodic checks ("every 5 minutes, check if...")
3. **Webhooks** — external services calling into OpenClaw ("when GitHub has a new issue, notify me")
4. **Polling** — periodically checking external services for changes
5. **Gmail PubSub** — Google's push notification system for email events

**Analogy:** Think of automation like setting alarms vs. having a doorbell:
- **Cron jobs** are alarms — they go off at set times regardless of what's happening
- **Webhooks** are doorbells — they ring when someone shows up (an external event)
- **Heartbeats** are a security guard doing rounds — checking things periodically
- **Polling** is looking out the window every few minutes to see if the mail has arrived

#### How Automation Creates Sessions

When an automated task triggers, it creates a **background task** with its own session. The results can be:
- Sent to you via a connected channel (Telegram, Discord, etc.)
- Stored as a file on disk
- Used to trigger further actions

#### Theory Check

1. What is the difference between a cron job and a webhook?
2. Why would you use heartbeats instead of cron for monitoring a website?
3. How does an automated task communicate its results to you?

---

### Part B — Hands-On: Setting Up Your First Automation

**Step 1:** Ask your assistant to set up a simple scheduled task:

> "Set up a daily reminder at 9 AM that checks Hacker News for the top story and sends me the headline and link."

**Step 2:** Verify the automation is configured by checking the config or asking:
> "What automated tasks do I have set up?"

**Step 3:** Test by triggering it manually:
> "Run my Hacker News check right now as a test."

---

## Phase 4 Projects

---

### Project 1: "Skill Forge" — Write Three Custom Skills

#### Theory Recap
Applies Section 4.1 (skills system) comprehensively.

#### What You'll Build
Three custom skills of increasing complexity: a simple greeting skill, a code review skill, and a project management skill.

#### What You'll Learn
- SKILL.md format mastery
- Frontmatter configuration
- Testing and iterating on skills
- Understanding token impact

#### Prerequisites
- Working OpenClaw with Dashboard and at least one channel

#### Step-by-Step Build Guide

1. **Skill 1: Code Reviewer** — Create `~/.openclaw/skills/code-review/SKILL.md`:
   ```markdown
   ---
   name: code-review
   description: Perform thorough code reviews with security, performance, and style checks
   ---

   When asked to review code, follow this structure:

   ## Code Review Framework
   1. **Security**: Check for injection vulnerabilities, exposed secrets, unsafe operations
   2. **Performance**: Identify O(n²) patterns, unnecessary allocations, missing caching
   3. **Readability**: Variable naming, function length, comment quality
   4. **Testing**: Are edge cases handled? Suggest test cases.
   5. **Architecture**: Does this follow the project's patterns?

   Format your review as:
   - 🔴 Critical (must fix)
   - 🟡 Warning (should fix)
   - 🟢 Suggestion (nice to have)

   Always end with a summary score: /10 with justification.
   ```

2. **Skill 2: Meeting Notes** — Create `~/.openclaw/skills/meeting-notes/SKILL.md`:
   ```markdown
   ---
   name: meeting-notes
   description: Transform rough meeting notes into structured action items and summaries
   ---

   When the user provides meeting notes or asks to summarize a meeting:

   1. Extract and list all **attendees** mentioned
   2. Write a 2-3 sentence **executive summary**
   3. List all **decisions made** as bullet points
   4. List all **action items** in this format:
      - [ ] [Action] — Owner: [Name] — Due: [Date if mentioned]
   5. Note any **open questions** that weren't resolved
   6. Suggest **follow-up** topics for the next meeting

   Save the output to ~/meetings/YYYY-MM-DD-topic.md when asked.
   ```

3. **Skill 3: Git Workflow** — Create `~/.openclaw/skills/git-workflow/SKILL.md`:
   ```markdown
   ---
   name: git-workflow
   description: Assist with Git operations using conventional commits and branch naming
   metadata: { "openclaw": { "requires": { "bins": ["git"] } } }
   ---

   When helping with Git operations:

   **Branch naming:** Use the format `type/short-description`
   - Types: feature/, bugfix/, hotfix/, chore/, docs/

   **Commit messages:** Follow Conventional Commits:
   - feat: new feature
   - fix: bug fix
   - docs: documentation
   - style: formatting
   - refactor: code restructuring
   - test: adding tests
   - chore: maintenance

   **Before committing:** Always run `git status` and `git diff --staged` first.
   **After committing:** Show the log with `git log --oneline -5`.
   ```

4. **Restart the Gateway:**
   ```bash
   openclaw gateway restart
   ```

5. **Test each skill:**
   - For code-review: paste a code snippet and say "Review this code"
   - For meeting-notes: paste some rough notes and say "Summarize this meeting"
   - For git-workflow: say "Help me commit my changes to git"

#### How to Test It
- ✅ Each skill is loaded (check `/context list` for their presence)
- ✅ Code review follows the structured format with emoji severity levels
- ✅ Meeting notes produce action items and summaries
- ✅ Git workflow uses conventional commit format

#### Common Pitfalls
- **Forgetting to restart the Gateway** after adding skills
- **Skill file not named SKILL.md** — must be exactly `SKILL.md`
- **Too verbose skills** consuming too many tokens

#### Stretch Goals
1. Add gating to a skill: make the git-workflow skill only load when `git` is on PATH.
2. Install a community skill from ClawHub and compare it to yours.
3. Create a skill with `user-invocable: true` and trigger it with a slash command.

---

### Project 2: "The Morning Dashboard" — Automated Daily Briefing

#### Theory Recap
Applies Section 4.4 (automation), 3.1 (memory), 3.3 (browser, shell).

#### What You'll Build
An automated daily briefing that runs every morning at 8 AM, gathers information from multiple sources, and sends you a summary via Telegram.

#### What You'll Learn
- Setting up cron-based automation
- Combining multiple tools in an automated workflow
- Delivering automated results through channels

#### Prerequisites
- Working OpenClaw with Telegram connected
- Memory configured with your preferences

#### Step-by-Step Build Guide

1. **Design the briefing content.** Ask the Dashboard:
   > "I want to set up a daily briefing that runs at 8 AM. It should: (1) check the weather for San Francisco, (2) get the top 3 Hacker News stories, (3) check my disk space usage, and (4) greet me by name from memory. Send the result to Telegram."

2. **Follow the assistant's setup instructions** for configuring the automation.

3. **Test the briefing manually:**
   > "Run my morning briefing right now."

4. **Wait for the next 8 AM trigger** and verify it arrives on Telegram.

5. **Iterate on the content:**
   > "Add today's top story from TechCrunch to the briefing."

#### How to Test It
- ✅ Manual trigger produces a formatted briefing
- ✅ Briefing arrives on Telegram at 8 AM
- ✅ Content includes all requested sections
- ✅ Personal greeting uses your name from memory

#### Common Pitfalls
- **Timezone mismatch:** The cron runs on Gateway host time. Make sure you know your server's timezone.
- **API rate limits:** Browsing multiple sites in an automated run can hit rate limits.
- **Token cost:** Automated runs cost money. Each briefing = one full agent loop with tool calls.

#### Stretch Goals
1. Add a weekly summary that runs on Sundays with a week-in-review format.
2. Include stock prices or cryptocurrency data in the briefing.
3. Make the briefing adaptive: if it's a weekday, include work-related checks; on weekends, skip them.

---

### Project 3: "Skill Publisher" — Share a Skill on ClawHub

#### Theory Recap
Applies Section 4.1 (skills, ClawHub) and community contribution concepts.

#### What You'll Build
A polished, well-documented skill published to ClawHub for the community to use.

#### What You'll Learn
- ClawHub publishing workflow
- Writing professional skill documentation
- Community contribution process

#### Prerequisites
- At least one working custom skill (from Project 1)

#### Step-by-Step Build Guide

1. **Choose your best skill** from Project 1 and polish it.

2. **Add comprehensive frontmatter:**
   ```markdown
   ---
   name: code-review-pro
   description: Thorough multi-dimension code review with security, performance, and style analysis
   metadata: {
     "openclaw": {
       "emoji": "🔍",
       "homepage": "https://github.com/yourname/code-review-skill"
     }
   }
   ---
   ```

3. **Add a README.md** alongside the SKILL.md explaining how to use it.

4. **Sync to ClawHub:**
   ```bash
   clawhub sync --all
   ```

5. **Verify** your skill appears on ClawHub.

#### How to Test It
- ✅ Skill is published on ClawHub
- ✅ Others can install it with `openclaw skills install <slug>`
- ✅ Documentation is clear and complete

#### Stretch Goals
1. Add installation requirements (gating) to your published skill.
2. Get 5 community members to install and review your skill.
3. Create a skill collection (3+ related skills) and publish them together.

---

## Phase 4 — End of Phase Review

### Conceptual Recap Questions

1. What is the architectural difference between a skill and a plugin?
2. Explain skill loading precedence. Where would you put a skill that should override a bundled skill?
3. How does gating prevent unnecessary token usage?
4. Describe two automation patterns and when you'd choose one over the other.
5. When should you use Claude Sonnet vs. Claude Haiku?

### Practical Consolidation Challenge

**"The Automated Skill Factory"**: Create a skill that, when invoked, helps you write OTHER skills. It should walk you through: naming, description, writing the instructions, adding frontmatter, saving the file, and restarting the Gateway. Meta!

---

**Phase 4 complete!** You can now extend OpenClaw with custom capabilities and automate workflows. In Phase 5, you become a power user: multi-agent setups, remote deployment, and advanced security.
