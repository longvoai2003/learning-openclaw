# Workspace Architecture Analysis (`~/.openclaw/workspace/`)

After analyzing the markdown files within the workspace folder, it is clear that OpenClaw utilizes a structured, multi-file approach to define the agent's persona and context. 

Here is the breakdown of each file's role:

1. **[AGENTS.md](file:///home/vo/.openclaw/workspace/AGENTS.md) (The System Prompt / Bootloader)**
   - Acts as the primary set of rules for the AI.
   - It does **not** contain the persona itself. Instead, it explicitly instructs the AI to read other files ([SOUL.md](file:///home/vo/.openclaw/workspace/SOUL.md), [USER.md](file:///home/vo/.openclaw/workspace/USER.md), memory) before acting.
   - It defines operational rules: memory management, when to speak in group chats, how to use tools, and heartbeat protocols.

2. **[BOOTSTRAP.md](file:///home/vo/.openclaw/workspace/BOOTSTRAP.md) (The Initialization Script)**
   - Used on the very first run.
   - Instructs the AI to initiate a conversation with the user to discover its identity, the user's details, and its soul/behavior.
   - Tells the AI to write this information into [IDENTITY.md](file:///home/vo/.openclaw/workspace/IDENTITY.md), [USER.md](file:///home/vo/.openclaw/workspace/USER.md), and [SOUL.md](file:///home/vo/.openclaw/workspace/SOUL.md), and then delete itself.

3. **[IDENTITY.md](file:///home/vo/.openclaw/workspace/IDENTITY.md) (Structural Metadata)**
   - Stores simple, structural data about the persona: Name, Creature, Vibe, Emoji, and Avatar path.

4. **[SOUL.md](file:///home/vo/.openclaw/workspace/SOUL.md) (The Behavioral Core)**
   - Contains the actual narrative prompt describing the AI's personality, core truths, boundaries, and tone (e.g., "Be genuinely helpful, not performatively helpful").

5. **[USER.md](file:///home/vo/.openclaw/workspace/USER.md) (Human Context)**
   - Stores information about the user (name, pronouns, timezone, preferences). This helps the AI personalize interactions.

6. **[TOOLS.md](file:///home/vo/.openclaw/workspace/TOOLS.md) (Local Environment Notes)**
   - A cheat sheet for environment-specific variables like camera names, SSH IP addresses, or TTS voice preferences, preventing hardcoded secrets in shared skills.

7. **[HEARTBEAT.md](file:///home/vo/.openclaw/workspace/HEARTBEAT.md) (Periodic Tasks)**
   - A template for defining background tasks the agent should check periodically (e.g., emails, calendar) when triggered by a cron/heartbeat tick.

---

### Conclusion for Phase 1 Curriculum
My previous assumption that [AGENTS.md](file:///home/vo/.openclaw/workspace/AGENTS.md) should be the "Single Source of Truth" for identity was incorrect for this specific workspace architecture. [AGENTS.md](file:///home/vo/.openclaw/workspace/AGENTS.md) is the *system ruleset*, but the persona is explicitly configured across [IDENTITY.md](file:///home/vo/.openclaw/workspace/IDENTITY.md) and [SOUL.md](file:///home/vo/.openclaw/workspace/SOUL.md). 
Therefore, Project 3 in the curriculum *should* teach the user to configure [IDENTITY.md](file:///home/vo/.openclaw/workspace/IDENTITY.md) and [SOUL.md](file:///home/vo/.openclaw/workspace/SOUL.md). The confusion stemmed from the "What You'll Learn" section incorrectly citing [AGENTS.md](file:///home/vo/.openclaw/workspace/AGENTS.md) as the file for customizing personality.
