# Phase 6: Mastery & Contribution — Integrations, Ecosystem & Open Source

## Phase Overview

**Big-picture goal:** By the end of this phase, you'll be an OpenClaw master — capable of building complex real-world integrations, writing production plugins, contributing to the open-source project, and leading others in OpenClaw adoption.

**What you'll understand:**
- The integration pattern for connecting external services (Gmail, Calendar, GitHub, Spotify, smart home, health APIs)
- Plugin development at a professional level
- How the skills ecosystem works as a community
- The open-source contribution workflow

**What you'll be able to do:**
- Build end-to-end integrations with real APIs
- Write production-quality plugins with HTTP routes, custom tools, and event handlers
- Publish and maintain skills on ClawHub
- Contribute to OpenClaw's source code

---

## What You Need Before This Phase

- [ ] Completed Phases 1–5 (multi-agent, remote deployment, security)
- [ ] JavaScript/TypeScript proficiency (for plugin development)
- [ ] API credentials for services you want to integrate (Gmail, GitHub, Spotify, etc.)
- [ ] A GitHub account (for open-source contribution)

---

## 6.1 Building Real Integrations

### Part A — Theory & Conceptual Understanding

#### The Integration Pattern

Every OpenClaw integration follows the same fundamental pattern, regardless of what external service you're connecting:

```
External Service API
       ↕ (HTTP/REST/GraphQL)
OpenClaw Plugin or Skill
       ↕ (tool calls)
AI Agent
       ↕ (natural language)
You (via any channel)
```

The integration can be implemented at two levels:

1. **Skill-level integration** — You write SKILL.md instructions that teach the AI to use `curl`/`fetch` commands to call APIs directly. Simple but limited.

2. **Plugin-level integration** — You write a Node.js plugin that wraps the API into clean custom tools, handles authentication, error retry, and rate limiting. More work but much more robust.

#### Authentication Patterns

Different services use different auth methods:

| Pattern | Services | How to Handle |
|---------|----------|---------------|
| **API Key** | OpenAI, Anthropic, many REST APIs | Store in `openclaw.json` or environment variables |
| **OAuth 2.0** | Gmail, Google Calendar, GitHub, Spotify | Plugin handles the OAuth flow, stores refresh tokens |
| **Webhook** | GitHub, Stripe, linear, many SaaS | Plugin registers HTTP endpoints in the Gateway |
| **Local API** | Home Assistant, Obsidian, local services | Direct HTTP calls, usually no auth needed |

#### The Skill-Level Integration Approach

For simple APIs with key-based auth, a skill is often enough:

```markdown
---
name: weather-check
description: Check weather using wttr.in API
---

When asked about weather, use the bash tool to run:
curl -s "wttr.in/{city}?format=3"

Replace {city} with the requested city name (URL-encoded).
If no city is specified, ask the user for one.
```

This is quick to build but has limitations:
- No proper error handling
- API keys would need to be in the prompt (security risk)
- No caching, retry, or rate limiting
- Raw curl output may confuse the model

#### The Plugin-Level Integration Approach

For production integrations, a plugin wraps everything cleanly:

```javascript
// gmail-integration/index.js
export function definePluginEntry({ register }) {
  register(api => {
    api.registerTool('gmail_search', async ({ query, maxResults = 10 }) => {
      const auth = await getOAuthToken(); // handles refresh
      const results = await gmail.users.messages.list({
        auth,
        userId: 'me',
        q: query,
        maxResults
      });
      return formatResults(results);
    });

    api.registerTool('gmail_send', async ({ to, subject, body }) => {
      const auth = await getOAuthToken();
      await gmail.users.messages.send({ auth, userId: 'me', /* ... */ });
      return { success: true, to, subject };
    });
  });
}
```

Now the AI can simply call `gmail_search` and `gmail_send` as tools — clean, secure, and robust.

#### Theory Check

1. What are the two levels at which you can build integrations? When would you choose each?
2. Why is putting API keys in a SKILL.md a security risk?
3. Describe the integration pattern from external service → you. Name each layer.

---

### Part B — Hands-On: Building a Skill-Level Integration

**Step 1:** Create a simple weather skill:

```bash
mkdir -p ~/.openclaw/skills/weather
cat > ~/.openclaw/skills/weather/SKILL.md << 'EOF'
---
name: weather
description: Check current weather for any city worldwide
---

When the user asks about weather:

1. Use the bash tool to run: curl -s "wttr.in/CITY?format=%C+%t+%h+%w"
   Replace CITY with the city name (use + for spaces, e.g., New+York)
2. Parse the output and present it naturally
3. Include: conditions, temperature, humidity, and wind
4. If the city name is ambiguous, ask for clarification

Example:
User: "What's the weather in Tokyo?"
Tool: curl -s "wttr.in/Tokyo?format=%C+%t+%h+%w"
Output: "Sunny +25°C 45% →7km/h"
Response: "It's sunny in Tokyo at 25°C with 45% humidity and light easterly winds."
EOF
```

**Step 2:** Restart and test:
```bash
openclaw gateway restart
```

> "What's the weather in London right now?"

---

## 6.2 Specific Integration Guides

### Gmail Integration

**Theory:** Gmail integration uses OAuth 2.0 and Google's PubSub for real-time email notifications. The plugin handles the OAuth dance, and Google pushes new-email events to a webhook registered in the Gateway.

**Skill-level approach (read-only):**
```markdown
---
name: gmail-check
description: Check Gmail via the Google Gmail API
metadata: { "openclaw": { "requires": { "env": ["GMAIL_ACCESS_TOKEN"] } } }
---
When asked to check email, use the web tool to query:
https://gmail.googleapis.com/gmail/v1/users/me/messages?maxResults=5
with Authorization: Bearer $GMAIL_ACCESS_TOKEN header.
Parse and present the subjects and senders.
```

### GitHub Integration

**Theory:** GitHub has a rich REST and GraphQL API. OpenClaw can monitor repositories, manage issues, create PRs, and respond to webhooks.

**Skill example:**
```markdown
---
name: github-helper
description: Interact with GitHub repositories
metadata: { "openclaw": { "requires": { "bins": ["gh"] } } }
---
Use the GitHub CLI (gh) for all GitHub operations:
- `gh issue list` — list issues
- `gh pr list` — list pull requests
- `gh repo view` — view repository info
- `gh issue create --title "..." --body "..."` — create issue

Always confirm before creating or modifying anything.
```

### Spotify Integration

**Theory:** Spotify uses OAuth 2.0. A plugin can register tools for playback control, playlist management, and music search.

### Smart Home (Home Assistant)

**Theory:** Home Assistant exposes a REST API locally. Since OpenClaw runs on your network, it can call the Home Assistant API directly.

```markdown
---
name: home-assistant
description: Control smart home devices via Home Assistant
metadata: { "openclaw": { "requires": { "env": ["HA_URL", "HA_TOKEN"] } } }
---
To control devices, use the bash tool to call the Home Assistant API:
curl -X POST "$HA_URL/api/services/light/turn_on" \
  -H "Authorization: Bearer $HA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"entity_id": "light.living_room"}'

Common services: light/turn_on, light/turn_off, switch/toggle, climate/set_temperature
Always confirm device changes with the user first.
```

### Health APIs (WHOOP, etc.)

**Theory:** Health APIs like WHOOP provide recovery, sleep, and strain data via OAuth-authenticated REST APIs. A plugin handles auth and formats health data into daily summaries or trend reports.

#### Theory Check

1. Why does Gmail integration work best as a plugin rather than a skill?
2. For Home Assistant (a local service), would you use a skill or a plugin? Why?
3. How do webhooks enable real-time email notifications without polling?

---

### Part B — Hands-On: GitHub CLI Integration

**Step 1:** Install GitHub CLI:
```bash
# macOS
brew install gh
# Linux
sudo apt install gh
# or: sudo dnf install gh
```

**Step 2:** Authenticate:
```bash
gh auth login
```

**Step 3:** Create the skill:
```bash
mkdir -p ~/.openclaw/skills/github
cat > ~/.openclaw/skills/github/SKILL.md << 'EOF'
---
name: github
description: Manage GitHub repositories, issues, and PRs using the gh CLI
metadata: { "openclaw": { "requires": { "bins": ["gh"] } } }
---

Use the GitHub CLI (gh) for all GitHub operations:

## Reading
- `gh repo list` — list your repositories
- `gh issue list -R owner/repo` — list issues
- `gh pr list -R owner/repo` — list pull requests
- `gh pr view <number> -R owner/repo` — view PR details

## Writing (always confirm with user first)
- `gh issue create -R owner/repo --title "..." --body "..."`
- `gh pr create --title "..." --body "..." --base main`

## Workflows
- `gh run list -R owner/repo` — list recent CI runs
- `gh run view <id> -R owner/repo` — view run details

Always ask the user which repository they mean if not specified.
Never push, merge, or delete without explicit confirmation.
EOF
```

**Step 4:** Restart and test:
```bash
openclaw gateway restart
```

> "Show me my recent GitHub repositories."
> "List the open issues in my last repo."

---

## 6.3 Contributing to the OpenClaw Ecosystem

### Part A — Theory & Conceptual Understanding

#### The Skills Ecosystem as a Community

ClawHub is not just a download site — it's a community ecosystem:
- **Discovery:** Find skills others have built
- **Sharing:** Publish your own skills for others
- **Collaboration:** Fork, improve, and contribute back
- **Standards:** The SKILL.md format is a community standard

#### Contributing to OpenClaw Core

OpenClaw is MIT-licensed and open source. Contributing back to the core project involves:

1. **Understanding the codebase** — TypeScript/Node.js, monorepo structure
2. **Finding issues** — GitHub issues labeled `good first issue`
3. **Forking and branching** — standard Git workflow
4. **Writing tests** — the project has a test suite
5. **Submitting PRs** — with clear descriptions and test coverage

#### Theory Check

1. How does ClawHub benefit the broader OpenClaw community?
2. What license is OpenClaw released under? What does that allow?
3. What would be the first step if you wanted to contribute a bug fix to OpenClaw?

---

### Part B — Hands-On: Contributing

**Step 1:** Fork the OpenClaw repository:
```bash
gh repo fork openclaw/openclaw --clone
```

**Step 2:** Explore the codebase:
```bash
cd openclaw
ls -la
cat package.json | head -20
```

**Step 3:** Find beginner-friendly issues:
```bash
gh issue list --label "good first issue"
```

**Step 4:** Create a branch, make your change, and submit a PR.

---

## Phase 6 Projects

---

### Project 1: "Life Dashboard" — Multi-Service Integration Hub

#### Theory Recap
Applies Section 6.1 (integration pattern) and 6.2 (specific integrations).

#### What You'll Build
An integrated dashboard that combines weather, GitHub activity, and system health into a single conversational interface.

#### What You'll Learn
- Building multiple integrations that work together
- Cross-service data aggregation via natural language
- Error handling when services are unavailable

#### Prerequisites
- GitHub CLI installed and authenticated
- Working OpenClaw with Telegram

#### Step-by-Step Build Guide

1. **Install all prerequisite skills** (weather, GitHub from earlier sections).

2. **Create a unified dashboard skill:**
   ```markdown
   ---
   name: life-dashboard
   description: Unified view of weather, code, and system health
   ---

   When the user asks for their "dashboard" or "status update":

   1. Check the weather for their city (from memory)
   2. List recent GitHub notifications: `gh api notifications --jq '.[0:5] | .[] | .subject.title'`
   3. Check system disk space and memory
   4. Check OpenClaw Gateway status

   Format as a clean markdown dashboard with sections:
   - 🌤️ Weather
   - 🐙 GitHub Activity
   - 💻 System Health
   - 🦞 OpenClaw Status
   ```

3. **Restart and test:**
   ```bash
   openclaw gateway restart
   ```

   > "Show me my dashboard."

4. **Set up daily automation** to send this dashboard every morning.

#### How to Test It
- ✅ Dashboard includes data from all four sources
- ✅ Works from both Dashboard and Telegram
- ✅ Handles gracefully when a service is down (e.g., no internet for weather)

#### Stretch Goals
1. Add Spotify "now playing" data.
2. Add calendar events from Google Calendar.
3. Make the dashboard adaptive — show different data on weekdays vs. weekends.

---

### Project 2: "The Full Plugin" — Building a Production Plugin

#### Theory Recap
Applies Section 4.2 (plugins) and 6.1 (integration pattern) at full depth.

#### What You'll Build
A complete OpenClaw plugin that adds custom tools for a specific use case (e.g., a note-taking system with tagging, search, and archival).

#### What You'll Learn
- The `definePluginEntry` API
- Registering custom HTTP routes
- Registering custom tools
- Plugin lifecycle management

#### Prerequisites
- JavaScript/Node.js proficiency
- Understanding of the plugin architecture from Phase 4

#### Step-by-Step Build Guide

1. **Create the plugin directory:**
   ```bash
   mkdir -p ~/.openclaw/plugins/smart-notes
   cd ~/.openclaw/plugins/smart-notes
   npm init -y
   ```

2. **Create `openclaw.plugin.json`:**
   ```json
   {
     "name": "smart-notes",
     "version": "1.0.0",
     "description": "Tagged note-taking system with search and archival",
     "entry": "index.js"
   }
   ```

3. **Write the plugin (`index.js`):**
   ```javascript
   import { readFileSync, writeFileSync, existsSync, mkdirSync, readdirSync } from 'fs';
   import { join } from 'path';

   const NOTES_DIR = join(process.env.HOME, '.openclaw', 'notes');

   export function definePluginEntry({ register }) {
     register(api => {
       // Ensure notes directory exists
       if (!existsSync(NOTES_DIR)) mkdirSync(NOTES_DIR, { recursive: true });

       api.registerTool('create_note', async ({ title, content, tags }) => {
         const filename = `${Date.now()}_${title.replace(/\s+/g, '_')}.md`;
         const frontmatter = `---\ntitle: ${title}\ntags: [${tags.join(', ')}]\ndate: ${new Date().toISOString()}\n---\n\n`;
         writeFileSync(join(NOTES_DIR, filename), frontmatter + content);
         return { success: true, filename, tags };
       });

       api.registerTool('search_notes', async ({ query }) => {
         const files = readdirSync(NOTES_DIR).filter(f => f.endsWith('.md'));
         const results = files
           .map(f => ({ name: f, content: readFileSync(join(NOTES_DIR, f), 'utf8') }))
           .filter(f => f.content.toLowerCase().includes(query.toLowerCase()))
           .map(f => ({ file: f.name, preview: f.content.substring(0, 200) }));
         return { count: results.length, results };
       });

       api.registerTool('list_notes', async ({ tag }) => {
         const files = readdirSync(NOTES_DIR).filter(f => f.endsWith('.md'));
         const notes = files.map(f => {
           const content = readFileSync(join(NOTES_DIR, f), 'utf8');
           const titleMatch = content.match(/title: (.+)/);
           const tagsMatch = content.match(/tags: \[(.+)\]/);
           return {
             file: f,
             title: titleMatch ? titleMatch[1] : f,
             tags: tagsMatch ? tagsMatch[1] : ''
           };
         });
         if (tag) return notes.filter(n => n.tags.includes(tag));
         return notes;
       });
     });
   }
   ```

4. **Restart the Gateway:**
   ```bash
   openclaw gateway restart
   ```

5. **Test the plugin tools:**
   > "Create a note titled 'OpenClaw Study Notes' tagged with 'learning' and 'openclaw'. Content: I completed all 6 phases of the OpenClaw curriculum today!"

   > "Search my notes for 'curriculum'"

   > "List all notes tagged with 'learning'"

#### How to Test It
- ✅ `create_note` tool creates a proper markdown file
- ✅ `search_notes` finds notes by content
- ✅ `list_notes` filters by tag
- ✅ Notes persist across Gateway restarts

#### Stretch Goals
1. Add a `delete_note` and `archive_note` tool.
2. Register an HTTP endpoint (`/api/notes`) for external access.
3. Add full-text search indexing for large note collections.

---

### Project 3: "Community Champion" — Open Source Contribution

#### Theory Recap
Applies Section 6.3 (contributing to the ecosystem).

#### What You'll Build
A meaningful contribution to the OpenClaw ecosystem: either a published ClawHub skill pack, a documentation improvement, or a code contribution.

#### What You'll Learn
- Open-source contribution workflows
- Writing documentation and examples for others
- Community engagement best practices

#### Prerequisites
- GitHub account
- A polished skill or plugin from earlier projects

#### Step-by-Step Build Guide

1. **Choose your contribution type:**
   - **Option A:** Publish a skill pack (3+ related skills) to ClawHub
   - **Option B:** Improve OpenClaw documentation (fix errors, add examples)
   - **Option C:** Fix a bug or add a feature to OpenClaw core

2. **For Option A — Skill Pack:**
   ```bash
   # Organize your skills
   mkdir -p ~/openclaw-skills-pack/{code-review,meeting-notes,git-workflow}
   # Copy your polished SKILL.md files from Phase 4
   # Add README.md with installation and usage instructions
   # Publish via ClawHub
   clawhub sync --all
   ```

3. **For Option B — Documentation:**
   ```bash
   gh repo fork openclaw/docs --clone
   cd docs
   # Find an area to improve
   # Make your changes
   # Submit a PR
   gh pr create --title "Improve getting started guide with troubleshooting section"
   ```

4. **For Option C — Code Contribution:**
   ```bash
   gh repo fork openclaw/openclaw --clone
   cd openclaw
   gh issue list --label "good first issue"
   # Pick an issue, create a branch, implement, test, PR
   ```

#### How to Test It
- ✅ Your contribution is submitted (PR or ClawHub publish)
- ✅ Documentation/code passes review
- ✅ Skill pack works when installed by others

#### Stretch Goals
1. Get 10+ installs on your ClawHub skill pack.
2. Have a PR merged into OpenClaw core.
3. Write a blog post about your OpenClaw journey and share it with the community.

---

## Phase 6 — End of Phase Review

### Conceptual Recap Questions

1. Explain the integration pattern. How does an external API call flow from you saying "check my email" to the AI reading your inbox?
2. When should you build a skill-level vs. plugin-level integration?
3. What are the key security considerations when building integrations that handle OAuth tokens?
4. How does contributing skills to ClawHub benefit both you and the community?
5. Describe the architecture of a production OpenClaw plugin: what can it register and why?

### Practical Consolidation Challenge — The Final Boss

**"The Autonomous Life Manager"**: Build an OpenClaw setup that:
1. Has at least 2 agents (general + specialist)
2. Is deployed remotely with proper security
3. Has 5+ custom skills
4. Has at least 1 custom plugin
5. Integrates with at least 2 external services (GitHub + one more)
6. Has daily automated briefings
7. Passes a full security audit

Ask your assistant: "Give me a complete status report — who are you, where are you running, what can you do, who can talk to you, and what automated tasks do you manage?"

The response should demonstrate mastery of every concept in this curriculum.

---

## Curriculum Completion

**Congratulations!** 🦞

If you've completed all six phases with their projects, you now:

- **Understand** OpenClaw's architecture from the Gateway to the agent loop
- **Operate** a multi-channel, multi-agent setup with production security
- **Extend** OpenClaw with custom skills, plugins, and integrations
- **Automate** workflows that run without your involvement
- **Deploy** to remote servers with proper security
- **Contribute** to the community that makes OpenClaw better

You're not just a user — you're a practitioner. Keep building, keep sharing, and keep teaching others what you've learned.

---

## Appendix: Quick Reference

### Essential Commands

```bash
# Installation & Setup
openclaw onboard --install-daemon
openclaw gateway status
openclaw status
openclaw dashboard

# Gateway Management
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway --verbose
openclaw logs --follow

# Channel Management
openclaw channels status --probe

# Session Management
openclaw sessions --json
openclaw sessions cleanup --dry-run

# Skills Management
openclaw skills install <slug>
openclaw skills update --all

# Security
openclaw security audit

# In-Chat Commands
/new              # New session
/new <model>      # New session with model switch
/reset            # Reset session
/status           # Show context usage and model
/context list     # Show system prompt contents
/agent <name>     # Switch agent
```

### Key File Locations

| Path | Purpose |
|------|---------|
| `~/.openclaw/openclaw.json` | Main configuration file |
| `~/.openclaw/skills/` | Managed/shared skills |
| `~/.openclaw/agents/` | Agent state, sessions, memory |
| `~/.openclaw/plugins/` | Installed plugins |
| `~/.agents/skills/` | Personal agent skills |
| `<workspace>/skills/` | Workspace-specific skills |
| `<workspace>/.agents/skills/` | Project agent skills |

### Config Structure Cheatsheet

```json
{
  "gateway": { "auth": { "token": "..." }, "reload": { "mode": "hybrid" } },
  "agents": { "defaults": { "model": "...", "instructions": "..." } },
  "channels": { "telegram": { "enabled": true, "botToken": "..." } },
  "session": { "dmScope": "per-channel-peer", "reset": { "idleMinutes": 60 } },
  "skills": { "load": { "extraDirs": [] } },
  "providers": { "anthropic": { "apiKey": "..." } },
  "messages": { "groupChat": { "mentionPatterns": ["@claw"] } }
}
```
