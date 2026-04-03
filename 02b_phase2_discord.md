# Phase 2b: Discord Integration — Connecting Your Workspace

## Phase Overview

**Big-picture goal:** By the end of this phase, you will have successfully connected your OpenClaw assistant to a Discord server (guild) and securely configured its response routing, demonstrating both DM and group channel isolation.

**What you'll understand:**
- How the Discord Bot API integrates with OpenClaw's channel abstraction
- The necessity of Discord Privileged Intents for conversational AI
- How OpenClaw handles session isolation between Discord DMs and server channels
- The mechanics of role-based and user-based routing within Discord guilds

**What you'll be able to do:**
- Create and provision a Discord Bot in the Developer Portal
- Securely inject your `DISCORD_BOT_TOKEN` into the OpenClaw configuration using environment variables
- Configure strict `allowlist` access controls for your Discord server
- Ensure the bot only responds when explicitly mentioned

---

## What You Need Before This Phase

- [ ] Completed Phase 1 (working OpenClaw installation with Gateway running)
- [ ] A Discord account with permission to create applications
- [ ] Administrator access to a Discord server (for adding the bot)

---

## 2b.1 The Discord Bot API & Gateway Routing

### Part A — Theory & Conceptual Understanding

#### How Discord Connects to OpenClaw

Unlike Telegram's simple token-based system, Discord uses a **Bot Application** model configured through the Discord Developer Portal. Discord requires bots to explicitly request permission to read messages. These permissions are called **Intents**.

For an AI assistant to function, it *must* have the **Message Content Intent** enabled. Without it, the bot can see that a message was sent, but cannot read the text inside it! 

Additionally, because Discord supports massive community servers, OpenClaw isolates sessions based on the context:
- **Direct Messages:** Handled via `session.dmScope`, maintaining context specific to the user.
- **Guild Channels:** Isolated into unique session keys (e.g., `agent:<agentId>:discord:channel:<channelId>`). This prevents a conversation in the `#general` channel from bleeding into the `#coding` channel.

#### The Analogy: The Secure Building

**Analogy:** Think of Discord as a secure corporate building. The **Bot Token** is the ID badge that lets your assistant into the lobby. The **Intents** are the security clearances required to actually hear what people are saying. And the **Guild Allowlist** in `openclaw.json` is the list of specific meeting rooms your assistant is allowed to enter and speak in.

#### Theory Check

1. Why does your Discord bot need the "Message Content Intent" enabled, and what happens if it is disabled?
2. If two different users talk to the bot in the same Discord server channel, do they share the same memory context? Why or why not?
3. What is the difference between authenticating the bot (Token) and authorizing the bot's behavior (Intents/Allowlist)?

---

### Part B — Hands-On: Quick Setup & Authentication

**Step 1:** Create the Bot Application.

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications).
2. Click **New Application** and give it a name.
3. Navigate to the **Bot** tab and click **Add Bot**.
4. Scroll down to **Privileged Gateway Intents** and enable:
   - **Message Content Intent** (Required)
   - **Server Members Intent** (Recommended for role routing)
5. Click **Reset Token** and copy your `DISCORD_BOT_TOKEN`. (Keep this secret!)

**Step 2:** Invite the Bot to your server.

1. In the Developer Portal, go to **OAuth2** -> **URL Generator**.
2. Select the `bot` and `applications.commands` scopes.
3. Under Bot Permissions, select: *View Channels*, *Send Messages*, *Read Message History*, and *Attach Files*.
4. Copy the generated URL, paste it into your browser, and invite the bot to your server.

**Step 3:** Securely inject the token into OpenClaw.

Instead of pasting the token plainly in your `openclaw.json` (which is a security risk), we will use config references to inject it via an environment variable.

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN_HERE"
openclaw config set channels.discord.token --ref-provider default --ref-source env --ref-id DISCORD_BOT_TOKEN
openclaw config set channels.discord.enabled true --strict-json
```

*This securely tells OpenClaw to fetch the token from the environment at runtime instead of saving it on disk.*

**Step 4:** Restart the Gateway.

```bash
openclaw gateway restart
```

**Step 5:** Check the setup.

```bash
openclaw channels status --probe
```

You should see Discord listed as `connected`.

#### Common Mistakes to Watch For

- **Message Content Intent Disabled:** The bot appears online but ignores all messages. **Fix:** Go back to the Discord Developer Portal, enable the Message Content Intent, and restart the gateway.
- **Hardcoding the token:** Putting the token directly into `openclaw.json` when the repo is shared. **Fix:** Use the `--ref-provider` environment injection method described above.

---

## 2b.2 Advanced Routing & Mentions

### Part A — Theory & Conceptual Understanding

#### Group Chat Rules

If you drop an AI bot into a busy Discord server with no rules, it will attempt to respond to *every single message*, rapidly depleting your API credits and annoying users.

To prevent this, OpenClaw requires you to specify a `groupPolicy` for Discord, which defaults to safely ignoring everything. You must explicitly configure an `allowlist` for your guild (server), and define exactly *who* can trigger the bot, and *how* (e.g., requiring `@mentions`).

#### Theory Check

1. What is the default `groupPolicy` behavior in OpenClaw for Discord?
2. Why is `requireMention: true` critical for busy server environments?
3. How does OpenClaw map a Discord Server ID to its routing engine?

---

### Part B — Hands-On: Guild Workspace Setup

**Step 1:** Enable Developer Mode in Discord to get your IDs.

1. In Discord, go to **User Settings** (gear icon) -> **Advanced** -> Toggle on **Developer Mode**.
2. Right-click your Server icon in the sidebar and click **Copy Server ID**.
3. Right-click your own Avatar and click **Copy User ID**.

**Step 2:** Configure the server allowlist.

Edit your `~/.openclaw/openclaw.json` to secure the Discord channel:

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "groupPolicy": "allowlist",
      "guilds": {
        "YOUR_SERVER_ID": {
          "requireMention": true,
          "users": ["YOUR_USER_ID"]
        }
      }
    }
  }
}
```

*This config tells OpenClaw to only respond in `YOUR_SERVER_ID`, only if it is explicitly `@mentioned`, and only if the message comes from `YOUR_USER_ID`.*

**Step 3:** Restart the Gateway.

```bash
openclaw gateway restart
```

#### Common Mistakes to Watch For

- **Getting IDs Wrong:** Accidentally swapping the User ID and the Server ID. **Fix:** `guilds` takes the Server ID as the key, and `users` takes the User IDs in an array.
- **Expecting Implicit Replies:** Sending a message without mentioning the bot when `requireMention` is true. **Fix:** Always start your message with `@BotName` in group channels when this is enabled.

---

## Phase 2b Projects

---

### Project 1: "The Support Bot" — Role-Based Agent Routing

#### Theory Recap
While `channels.discord.guilds` controls *if* the bot can respond to a user (authorization), **Bindings** control *which* AI agent responds (routing). OpenClaw allows evaluating bindings by Discord Role ID, letting you route different guild members to different specialized agents based on their roles.

*Note: Role-based bindings accept role IDs only and are evaluated after peer or parent-peer bindings and before guild-only bindings. If a binding sets other match fields (like `guildId` + `roles`), all configured fields must match.*

#### What You'll Build
You will configure OpenClaw to route members with the "VIP Support" role to a smart, expensive agent (e.g., Opus), while everyone else in the server gets routed to a fast, basic agent (e.g., Sonnet).

#### What You'll Learn
- Defining `bindings` for Discord role-based routing.
- The order of precedence for binding evaluation (`guildId` + `roles` vs just `guildId`).

#### Prerequisites
- A connected Discord bot from Section 2b.1.
- A Discord server with a specific role created (e.g., "VIP Support").
- At least two standard agents defined in your OpenClaw setup (e.g., `opus` and `sonnet`).

#### Step-by-Step Build Guide

1. **Get the Required IDs:** 
   - Right-click your server icon -> **Copy Server ID**.
   - Create a role named "VIP Support", right-click it in Server Settings -> **Copy Role ID**.

2. **Update the config:** Edit `openclaw.json` to add the `bindings` array at the top level of your configuration.
   ```json
   {
     "bindings": [
       {
         "agentId": "opus",
         "match": {
           "channel": "discord",
           "guildId": "YOUR_SERVER_ID",
           "roles": ["YOUR_VIP_ROLE_ID"]
         }
       },
       {
         "agentId": "sonnet",
         "match": {
           "channel": "discord",
           "guildId": "YOUR_SERVER_ID"
         }
       }
     ],
     "channels": {
       "discord": {
         "enabled": true,
         "groupPolicy": "allowlist",
         "guilds": {
           "YOUR_SERVER_ID": {
             "requireMention": true
           }
         }
       }
     }
   }
   ```
   *Explanation: OpenClaw evaluates bindings top-down. Users with the VIP role match the first binding and get `opus`. Other users fall through to the second binding and get `sonnet`.*

3. **Restart the gateway:**
   ```bash
   openclaw gateway restart
   ```

#### How to Test It
- ✅ Give yourself the VIP Role and `@mention` the bot in the server -> Confirm the response style originates from the Opus agent.
- ✅ Remove the VIP Role and `@mention` the bot again -> Confirm the response style now originates from the Sonnet agent.

#### Common Pitfalls
- **Forgetting the base allowlist:** Bindings only route the request. If the server isn't also approved in `channels.discord.guilds`, the bot will ignore the message entirely. **Fix:** Ensure the guild is in the `channels.discord.guilds` allowlist.
- **Using names instead of IDs:** Bindings require the numeric Discord IDs (`123456789...`), not role names like "VIP Support".

#### Stretch Goals
1. Add a third binding that routes direct messages (DMs) from users (where `guildId` is not present) to a specialized `dm_assistant` agent.

---

### Project 2: "The File Dispatcher" — Sending Artifacts to Discord

#### Theory Recap
OpenClaw can interact with the host file system and generate artifact files (such as code, summaries, or structured data). In order to send these files directly to a Discord channel, the Discord bot application must have explicit authorization through the **Attach Files** permission. This prevents malicious or unbounded file dumping unless explicitly granted by the server administrator.

#### What You'll Build
You will re-authorize your OpenClaw Discord bot to have file attachment privileges. Then, you will ask your agent to generate a markdown file and send it directly to your Discord server channel.

#### What You'll Learn
- How to update an existing Discord bot's permissions without deleting it.
- How OpenClaw agents interact with and dispatch generated files to a Discord channel.

#### Prerequisites
- A connected Discord bot from Section 2b.1.
- Administrator permission in your Discord server.
- The `allowlist` configuration from Section 2b.2 correctly set up.

#### Step-by-Step Build Guide

1. **Re-authorize the Bot with File Permissions:** 
   In the [Discord Developer Portal](https://discord.com/developers/applications), navigate to your bot's **OAuth2** -> **URL Generator**.
   Select the `bot` and `applications.commands` scopes.
   Under Bot Permissions, select: *View Channels*, *Send Messages*, *Read Message History*, and **Attach Files**.
   Copy the URL and invite the bot to your server again to update permissions.

2. **Trigger the File Generation:**
   In your Discord server, `@mention` the bot and run the following prompt:
   ```text
   @YourBotName Please create a markdown file titled "OpenClaw_Summary.md" explaining your core capabilities, and send the file here.
   ```

#### How to Test It
- ✅ The bot acknowledges the request and successfully uploads the `OpenClaw_Summary.md` file as an attachment in the Discord channel.
- ✅ Downloading and opening the file reveals the correct markdown content.

#### Common Pitfalls
- **"Missing Permissions" Error:** The bot fails to send the message or logs a Discord API error. **Fix:** Ensure you actually re-authorized the bot using the URL with the **Attach Files** box checked. Server roles must also not override and deny this permission.

---

## Phase 2b — End of Phase Review

### Conceptual Recap Questions

1. Explain why relying solely on OpenClaw's internal allowlist is insufficient if your Discord bot token is exposed publicly.
2. Outline the exact data flow required to securely inject your `DISCORD_BOT_TOKEN` into the Gateway without saving it in plaintext.
3. Describe the difference in session handling between a Discord DM (`dmScope: per-channel-peer`) and a Discord Guild channel.

### Practical Consolidation Challenge

**"The Split Personality"**: Configure the Discord channel so that any user can DM the bot directly (via standard DM pairing), but within your main server, only you can trigger it, and only with a `@mention`. Prove the isolation works by teaching the bot a secret in a DM, and verifying it does not remember the secret when conversing in the server channel.

### Next Functionalities To Explore

Once you have mastered the basics of Discord integration, consider exploring these advanced features (detailed in Phase 5 and Appendix docs):
- **Slash Commands:** Registering native `/commands` via `channels.discord.commands.native` to trigger specific agent skills.
- **PluralKit Integration:** Enable `pluralkit: true` to seamlessly support system headmates if your community utilizes PluralKit.
- **Voice Channels & Messages:** Extending your gateway to process incoming voice notes via Discord.
- **Forum Channels:** Creating automatic categorized threads when messaging a Discord forum parent channel.
