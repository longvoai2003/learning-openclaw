# Phase 2: Connecting the World — Chat Channels, Pairing & Groups

## Phase Overview

**Big-picture goal:** By the end of this phase, your OpenClaw assistant will be reachable from your everyday chat apps. You'll understand the channel abstraction model, set up at least two channels, configure DM safety, and manage group chat behavior.

**What you'll understand:**
- How OpenClaw abstracts different chat platforms into a unified "channel" model
- The message routing system — how a Telegram message reaches the same Gateway as a Dashboard message
- The security model for DMs: pairing, allowlists, and approval flows
- How group chats work with mentions and filtering

**What you'll be able to do:**
- Connect Telegram, Discord, WhatsApp, and/or other channels
- Configure DM safety and sender allowlists
- Set up group chat mention patterns
- Route messages across channels to the same assistant

---

## What You Need Before This Phase

- [ ] Completed Phase 1 (working OpenClaw installation with Gateway running)
- [ ] A Telegram account and phone (for the quickest channel setup)
- [ ] Optional: WhatsApp, Discord, or Slack accounts for additional channels

---

## 2.1 The Channel Abstraction Model

### Part A — Theory & Conceptual Understanding

#### What a "Channel" Is in OpenClaw

A **channel** is OpenClaw's abstraction over a chat platform. Telegram, WhatsApp, Discord, Slack, Signal, iMessage — each is a different chat app with different APIs, different authentication methods, and different message formats. OpenClaw's channel system normalizes all of these into a single internal format.

From the Gateway's perspective, a message is a message regardless of whether it came from Telegram or WhatsApp. The channel layer handles all the platform-specific translation.

**Analogy:** Think of channels like phone carriers. You have AT&T, Verizon, T-Mobile — each with different networks, different SIM cards, different apps. But when someone calls you, your phone rings the same way regardless of what carrier the caller uses. The phone (Gateway) doesn't care about the carrier (channel); it just receives calls (messages).

#### How Message Routing Works

```
Telegram ──┐
WhatsApp ──┤
Discord  ──┼──→ Gateway ──→ Session Router ──→ Agent Loop ──→ Response
Slack    ──┤                                                     │
Dashboard──┘                                                     │
    ↑                                                            │
    └────────────────────────────────────────────────────────────┘
    Response routed back to the ORIGINAL channel
```

Key routing principles:
1. **Responses go back the way they came.** If you message from Telegram, the response goes to Telegram.
2. **Sessions can be shared or isolated.** Depending on your `dmScope` config, the same person messaging from Telegram and WhatsApp might share one session or have separate ones.
3. **Multiple channels run simultaneously.** You don't choose one — you can have all of them active at once.

#### Supported Channels

| Channel | Setup Complexity | Notes |
|---------|-----------------|-------|
| **Telegram** | Easy (bot token) | Fastest to set up; recommended first channel |
| **Discord** | Medium (bot application) | Great for team/community use |
| **WhatsApp** | Medium (QR pairing) | Most common personal messaging; stores state on disk |
| **Slack** | Medium (app configuration) | Best for workplace integration |
| **Signal** | Medium | Privacy-focused |
| **iMessage** | Complex (macOS only, via BlueBubbles) | macOS companion app required |

#### Channel Configuration in openclaw.json

All channel configs live under the `channels` key:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "your-bot-token"
    },
    "whatsapp": {
      "enabled": true,
      "allowFrom": ["+15555550123"]
    },
    "discord": {
      "enabled": true,
      "botToken": "your-discord-bot-token"
    }
  }
}
```

#### Theory Check

1. What does it mean that OpenClaw "abstracts" chat platforms? Why is this useful?
2. If you send a message from Telegram, which app does the response arrive in? Could it arrive somewhere else?
3. Why can multiple channels run simultaneously from one Gateway?

---

### Part B — Hands-On: Understanding Your Channel Setup

**Step 1:** Check your current channel status:

```bash
openclaw channels status --probe
```

**Step 2:** Examine the channel configuration:

```bash
cat ~/.openclaw/openclaw.json | python3 -m json.tool
```

Look for the `channels` section. Note which channels are enabled (if any beyond the Dashboard).

---

## 2.2 Setting Up Telegram (Your First External Channel)

### Part A — Theory & Conceptual Understanding

#### Why Telegram First?

Telegram is recommended as your first external channel because:
1. **Simplest setup** — just a bot token, no QR codes or OAuth
2. **Fast bot creation** — BotFather creates bots in seconds
3. **Rich features** — supports images, files, inline keyboards, long messages
4. **Free** — no per-message costs

#### How Telegram Bots Work (Under the Hood)

When you create a Telegram bot, you get a **bot token** — a long string like `123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11`. This token lets OpenClaw:

1. **Register with Telegram's API** to receive messages sent to your bot
2. **Send responses** back through Telegram's API
3. **Receive updates** via long-polling or webhook (OpenClaw uses long-polling by default)

**Long-polling** means the Gateway periodically asks Telegram: "Any new messages for me?" This is simpler than webhooks (which require a public URL) and works perfectly from behind a NAT/firewall.

#### Theory Check

1. Why is Telegram the recommended first channel for learners?
2. What is a Telegram bot token, and what two things does it allow OpenClaw to do?
3. What is long-polling, and why does OpenClaw use it instead of webhooks for Telegram by default?

---

### Part B — Hands-On: Connecting Telegram

**Step 1:** Create a Telegram bot.

1. Open Telegram and search for **@BotFather**
2. Send `/newbot`
3. Choose a name (e.g., "My OpenClaw Assistant")
4. Choose a username (must end in `bot`, e.g., `my_openclaw_bot`)
5. BotFather gives you a token — **copy it immediately**

**Step 2:** Add the token to OpenClaw.

You can do this through the Control UI or by editing the config:

```bash
# Edit the config file
nano ~/.openclaw/openclaw.json
```

Add or update the Telegram section:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_BOT_TOKEN_HERE"
    }
  }
}
```

**Step 3:** Restart the Gateway.

```bash
openclaw gateway restart
```

**Step 4:** Verify the channel is connected.

```bash
openclaw channels status --probe
```

You should see Telegram listed as connected.

**Step 5:** Send your first message!

Open Telegram, find your bot (search for the username you chose), and send:

> "Hello from Telegram! Can you confirm which channel I'm reaching you from?"

#### Common Mistakes

- **Exposing the bot token:** Never commit it to git or share it publicly. Anyone with the token can control your bot.
- **Bot not responding:** Check `openclaw logs --follow` for errors. Common cause: incorrect token.
- **Double-running the bot:** If another process is already using this bot token, Telegram will return errors. Make sure no other service is using the same bot.

---

## 2.3 Pairing, Allowlists & DM Safety

### Part A — Theory & Conceptual Understanding

#### The Security Problem

Your OpenClaw assistant can run shell commands, read files, and browse the web on your computer. If anyone could message it and have it execute commands, that would be a catastrophic security hole.

OpenClaw solves this with a **layered DM safety model**:

1. **Allowlists** (`allowFrom`) — only specified phone numbers, usernames, or IDs can interact
2. **Pairing** — new senders must be explicitly approved before they can use the assistant
3. **Elevated mode** — dangerous operations require explicit confirmation

#### How Allowlists Work

In `openclaw.json`, each channel can have an `allowFrom` list:

```json
{
  "channels": {
    "whatsapp": {
      "allowFrom": ["+15555550123", "+15555550456"]
    },
    "telegram": {
      "allowFrom": ["123456789"]
    }
  }
}
```

Messages from senders not on the list are **silently ignored** — the person sending them won't even know they were dropped.

#### How Pairing Works

For channels without explicit allowlists, OpenClaw uses a pairing flow:
1. A new sender messages your bot
2. OpenClaw notifies you (the owner) about the new contact
3. You approve or reject the pairing
4. Once paired, that sender can use the assistant

This is like the "accept friend request" pattern — no one gets access without your approval.

#### DM Scope (Session Isolation for DMs)

Remember from Phase 1 that `dmScope` controls session isolation:
- `main` — all DMs share one session (everyone sees everyone's context — usually not what you want)
- `per-peer` — each sender gets their own session across all channels
- `per-channel-peer` — each sender on each channel gets their own session (recommended)

**Recommended config:**
```json
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

#### Theory Check

1. Why does OpenClaw need a safety model for DMs? What could go wrong without one?
2. What's the difference between an allowlist and pairing? When would you use each?
3. Explain `per-channel-peer` DM scope. Why is it more secure than `main`?

---

### Part B — Hands-On: Configuring DM Safety

**Step 1:** Set up an allowlist for Telegram.

Find your Telegram user ID (send a message to `@userinfobot` on Telegram to get it), then add it:

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_TOKEN",
      "allowFrom": ["YOUR_USER_ID"]
    }
  }
}
```

**Step 2:** Set the DM scope:

```json
{
  "session": {
    "dmScope": "per-channel-peer"
  }
}
```

**Step 3:** Restart and test:

```bash
openclaw gateway restart
```

**Step 4:** Test from an allowed account — it should work. Ask a friend to message the bot — their message should be silently dropped.

---

## 2.4 Group Chat Behavior

### Part A — Theory & Conceptual Understanding

#### The Group Chat Problem

In a group chat with 10 people, you don't want the AI responding to every single message. OpenClaw solves this with **mention rules** — the bot only responds when explicitly mentioned.

#### How Mention Patterns Work

```json
{
  "channels": {
    "whatsapp": {
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    }
  },
  "messages": {
    "groupChat": {
      "mentionPatterns": ["@openclaw", "@claw"]
    }
  }
}
```

- `"*"` means "all groups" — the wildcard pattern
- `requireMention: true` means the bot only responds when someone uses `@openclaw` or `@claw`
- You can configure per-group rules using the group's ID instead of `"*"`

#### Theory Check

1. Why would you enable `requireMention` in group chats?
2. What does the `"*"` wildcard mean in the groups configuration?
3. How would you configure the bot to respond to both `@ai` and `@assistant` mention patterns?

---

### Part B — Hands-On: Group Chat Setup

This is demonstrated as conceptual config — you'll practice it fully in the Projects below.

---

## 2.5 Adding More Channels: WhatsApp & Discord

### Part A — Theory & Conceptual Understanding

#### WhatsApp: QR Pairing

WhatsApp doesn't use a simple token like Telegram. Instead:
1. OpenClaw starts a WhatsApp Web session
2. You scan a QR code with your phone (like linking WhatsApp Web to a browser)
3. OpenClaw maintains that session, storing encryption keys on disk
4. Messages are end-to-end encrypted

**Important tradeoff:** WhatsApp stores more state on disk than Telegram, and the connection can be fragile (if you log into WhatsApp Web elsewhere, OpenClaw's session may disconnect).

#### Discord: Bot Application

Discord uses OAuth2 and a bot application model:
1. You create a "Discord Application" in the Discord Developer Portal
2. You add a "Bot" to the application and get a bot token
3. You invite the bot to your Discord server using an OAuth2 URL
4. OpenClaw connects using the bot token

Discord bots can work in both DMs and server channels, making it great for team use.

#### Theory Check

1. How does WhatsApp authentication differ from Telegram authentication?
2. What state does WhatsApp store on disk, and what can cause the connection to break?
3. For a team of 5 people who want to share one OpenClaw assistant, which channel would you recommend and why?

---

### Part B — Hands-On: Setting Up WhatsApp

**Step 1:** Enable WhatsApp in config:

```json
{
  "channels": {
    "whatsapp": {
      "enabled": true,
      "allowFrom": ["+1YOUR_NUMBER"]
    }
  }
}
```

**Step 2:** Restart the Gateway and check for the QR code:

```bash
openclaw gateway restart
openclaw logs --follow  # Look for QR code instructions
```

**Step 3:** Scan the QR code with your phone (WhatsApp → Settings → Linked Devices → Link a Device).

**Step 4:** Send a test message from WhatsApp to your own number or the paired bot.

---

## Phase 2 Projects

---

### Project 1: "Multi-Channel Hub" — Same Assistant, Three Channels

#### Theory Recap
This project applies: Sections 2.1 (channel abstraction), 2.2 (Telegram), 2.3 (DM safety), and 2.5 (WhatsApp/Discord).

#### What You'll Build
An OpenClaw setup with Telegram, Dashboard, and at least one additional channel all connected simultaneously to the same Gateway.

#### What You'll Learn
- Running multiple channels simultaneously
- Observing the same assistant responding on different platforms
- Verifying DM isolation works correctly

#### Prerequisites
- Completed Phase 1 (working Gateway)
- Telegram bot created (Section 2.2)
- At least one additional account (WhatsApp or Discord)

#### Step-by-Step Build Guide

1. **Ensure Telegram is already connected** (from Section 2.2).

2. **Add a second channel** (Discord or WhatsApp) to `openclaw.json`:
   ```json
   {
     "channels": {
       "telegram": {
         "enabled": true,
         "botToken": "YOUR_TELEGRAM_TOKEN",
         "allowFrom": ["YOUR_TELEGRAM_ID"]
       },
       "discord": {
         "enabled": true,
         "botToken": "YOUR_DISCORD_TOKEN"
       }
     }
   }
   ```

3. **Set DM scope to per-channel-peer:**
   ```json
   {
     "session": {
       "dmScope": "per-channel-peer"
     }
   }
   ```

4. **Restart the Gateway:**
   ```bash
   openclaw gateway restart
   openclaw channels status --probe
   ```
   Verify both channels show as connected.

5. **Test isolation.** Send a message from Telegram:
   > "Remember: my favorite color is blue."

   Then send from the Dashboard:
   > "What's my favorite color?"

   With `per-channel-peer`, the Dashboard session should NOT know about the Telegram session's context.

6. **Test simultaneous responses.** Send messages from both channels within seconds of each other and observe both get responses.

#### How to Test It
- ✅ `openclaw channels status --probe` shows multiple channels connected
- ✅ Messages from each channel get individual responses
- ✅ Session isolation is working (context doesn't leak between channels)

#### Common Pitfalls
- **Forgetting to restart:** After config changes, always `openclaw gateway restart`.
- **WhatsApp disconnecting:** If WhatsApp drops, check `openclaw logs --follow` for reconnection attempts.
- **Discord permissions:** The bot needs appropriate permissions in the server (Send Messages, Read Messages).

#### Stretch Goals
1. Add a third channel and test all three simultaneously.
2. Switch `dmScope` to `per-peer` and notice how your context is now shared across Telegram and Dashboard for the same user.
3. Set up mention patterns for a Discord server channel.

---

### Project 2: "The Gatekeeper" — Security-First Channel Configuration

#### Theory Recap
This project applies: Section 2.3 (pairing, allowlists, DM safety) deeply.

#### What You'll Build
A hardened OpenClaw configuration with strict allowlists, per-channel-peer isolation, and verified rejection of unauthorized senders.

#### What You'll Learn
- Implementing real security restrictions
- Testing that unauthorized messages are actually dropped
- Understanding the audit capabilities (`openclaw security audit`)

#### Prerequisites
- At least one channel connected (from Project 1)
- A friend or second account to test unauthorized access

#### Step-by-Step Build Guide

1. **Configure strict allowlists.** Edit `openclaw.json`:
   ```json
   {
     "channels": {
       "telegram": {
         "enabled": true,
         "botToken": "YOUR_TOKEN",
         "allowFrom": ["YOUR_USER_ID_ONLY"]
       }
     },
     "session": {
       "dmScope": "per-channel-peer"
     }
   }
   ```

2. **Restart and test from your allowed account:**
   ```bash
   openclaw gateway restart
   ```
   Send a message from your Telegram — it should work.

3. **Test unauthorized access.** Ask a friend to message your bot on Telegram. Watch the logs:
   ```bash
   openclaw logs --follow
   ```
   You should see the message being dropped. Your friend should receive no response.

4. **Run a security audit:**
   ```bash
   openclaw security audit
   ```

5. **Document your security configuration.** Ask the Dashboard:
   > "List all the security settings in my openclaw.json — who is allowed to talk to us, which channels are active, and what DM isolation mode is set."

#### How to Test It
- ✅ Your messages get responses
- ✅ Unauthorized messages are silently dropped
- ✅ `openclaw security audit` passes
- ✅ You can articulate your security configuration

#### Common Pitfalls
- **Wrong user ID format:** Telegram uses numeric user IDs (e.g., `"123456789"`), WhatsApp uses phone numbers with country code (e.g., `"+15555550123"`).
- **Testing from the same account:** You can't test "unauthorized" by messaging yourself. Use a different account.

#### Stretch Goals
1. Configure group-specific mention rules for a Telegram group.
2. Set up the pairing flow (remove the allowlist) and test approving/rejecting a new contact.
3. Add `session.identityLinks` to link your Telegram and WhatsApp identities so they share context.

---

### Project 3: "Group Chat Commander" — Setting Up Smart Group Chat Behavior

#### Theory Recap
This project applies: Section 2.4 (group chat behavior) and 2.3 (safety).

#### What You'll Build
An OpenClaw bot that works intelligently in a Telegram or Discord group — responding only when mentioned, with proper group-specific configuration.

#### What You'll Learn
- Configuring mention-based group interaction
- Managing group chat sessions vs. DM sessions
- Setting per-group rules

#### Prerequisites
- Telegram or Discord bot connected (from Project 1)
- A group chat you have admin access to

#### Step-by-Step Build Guide

1. **Add your bot to a Telegram group.** Add the bot as a member.

2. **Configure group rules:**
   ```json
   {
     "channels": {
       "telegram": {
         "enabled": true,
         "botToken": "YOUR_TOKEN",
         "groups": {
           "*": {
             "requireMention": true
           }
         }
       }
     },
     "messages": {
       "groupChat": {
         "mentionPatterns": ["@your_bot_username"]
       }
     }
   }
   ```

3. **Restart and test:**
   ```bash
   openclaw gateway restart
   ```

4. **Test WITHOUT mention:** Send a regular message in the group. The bot should NOT respond.

5. **Test WITH mention:** Send `@your_bot_username What's the weather like?` The bot SHOULD respond.

6. **Test DM alongside group:** Send a DM to the bot. It should respond without needing a mention (DMs don't require mentions).

#### How to Test It
- ✅ Bot ignores non-mentioned messages in groups
- ✅ Bot responds to mentioned messages in groups
- ✅ Bot responds to all DMs (no mention needed)
- ✅ Group sessions are isolated from DM sessions

#### Common Pitfalls
- **Bot privacy mode in Telegram:** BotFather privacy settings may prevent the bot from seeing group messages. Send `/setprivacy` to BotFather and set it to "Disable" (allowing the bot to see all group messages).
- **Wrong mention pattern:** The mention pattern must match exactly how users would mention the bot.

#### Stretch Goals
1. Set up a group-specific instruction: "In this group, always respond in Spanish."
2. Configure two groups with different `requireMention` settings — one requiring mention, one responding to all messages.
3. Set up a "read-only" group where the bot monitors messages but only responds when directly asked.

---

## Phase 2 — End of Phase Review

### Conceptual Recap Questions

1. Explain the channel abstraction model. How does it let one Gateway serve multiple chat platforms?
2. Walk through the journey of a Telegram message from the moment you tap "Send" to the moment you see a response.
3. Why does OpenClaw need DM safety? Describe the three layers of protection.
4. What is `per-channel-peer` DM scope and why is it recommended?
5. How do group chat mention patterns prevent the bot from being too chatty?

### Practical Consolidation Challenge

**"The Channel Portfolio"**: Set up your OpenClaw with at least two channels (Telegram + one more). Configure distinct DM safety for each. Join a group chat with your bot. Then send this message from each channel:

> "Which channel am I talking to you from? And which other channels can you be reached on?"

Verify the assistant correctly identifies the channel in each case. This proves you understand multi-channel routing.

---

**You've completed Phase 2!** Your assistant is now reachable from your everyday chat apps, secured with proper access controls. In Phase 3, you'll unlock persistent memory, understand sessions deeply, and learn to use the powerful built-in tools.
