# Phase 1b: OpenRouter — The Universal Intelligence Router

## Phase Overview

**Big-picture goal:** By the end of this phase, you will understand how to use OpenRouter with OpenClaw to seamlessly route requests to thousands of models, optimizing for both cost and capability without being locked into a single provider.

**What you'll understand:**
- The concept of LLM routing and why it is useful.
- How OpenAI-compatible endpoints work in the context of custom providers.
- The difference between native capabilities and proxy layers.

**What you'll be able to do:**
- Configure OpenClaw to authenticate with OpenRouter.
- Define a custom provider in `openclaw.json` for complex model mappings.
- Use the `openrouter/openrouter/auto` capability for cost-effective dynamic model selection.

---

## What You Need Before This Phase

- [ ] Completed Phase 1 (Foundation)
- [ ] OpenClaw installed and the Gateway running
- [ ] An account and an API key from [openrouter.ai](https://openrouter.ai)

---

## 1.6 The Universal Model Router

### Part A — Theory & Conceptual Understanding

#### Why Abstract the Provider?

When you first set up OpenClaw, you connected it directly to Anthropic, OpenAI, or Google. You gave it a key, and it talked directly to their servers. 

However, the AI landscape changes daily. Claude 3.5 Sonnet might be the best for coding today, while Gemini 1.5 Pro might be better for huge context, and Llama 3.1 might be the most cost-effective for simple automation. Managing separate API keys, billing, and configurations for every single provider becomes exhausting.

OpenRouter solves this by acting as a universal translation layer. You talk to OpenRouter, and OpenRouter talks to the models. Because OpenRouter mimics the OpenAI API format, OpenClaw can easily treat it like a single giant model provider. 

#### The Analogy: The Universal Power Adapter

**Analogy:** Imagine traveling across the world. Instead of buying a specific wall plug adapter for every single country (a different API key for every AI company), you buy one Universal Travel Adapter (OpenRouter). You plug OpenClaw into the Universal Adapter, and it automatically connects you to the local grid, no matter where you are. 

#### The "Auto" Model Magic

One of the most powerful features of OpenRouter is the `openrouter/auto` model. Instead of hardcoding which model you want, OpenRouter will automatically dynamically route your prompt to the cheapest model that meets your needs. If it's a simple heartbeat message, it might route to Llama-3 8B. If it's a complex coding request, it routes to Claude 3.5 Sonnet. OpenClaw supports this model natively.

#### Theory Check

1. Why might you choose to use OpenRouter instead of connecting directly to Anthropic and OpenAI?
2. Explain the analogy of the Universal Travel Adapter in this context.
3. What is the biggest advantage of using the `openrouter/auto` model for your Gateway's automation tasks?

---

### Part B — Hands-On: Configuring OpenRouter

**Step 1:** Stop your Gateway service to safely edit configuration.

```bash
openclaw gateway stop
```
Stops the background Gateway daemon.

**Step 2:** Open your OpenClaw configuration file.

```bash
nano ~/.openclaw/openclaw.json
```
Opens the configuration file in a terminal text editor.

**Step 3:** Add your OpenRouter API key and configure the model defaults. Modify your configuration to include the `env` and `agents` sections:

```json
{
  "env": {
    "OPENROUTER_API_KEY": "sk-or-v1-your-key-here"
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openrouter/anthropic/claude-3.5-sonnet",
        "fallbacks": ["openrouter/openrouter/auto"]
      },
      "models": {
        "openrouter/anthropic/claude-3.5-sonnet": {
          "alias": "Claude 3.5 (OpenRouter)"
        },
        "openrouter/openrouter/auto": {
          "alias": "Auto Router"
        }
      }
    }
  }
}
```
Configures OpenClaw to use OpenRouter credentials and defaults the primary model to Claude 3.5 Sonnet via OpenRouter, with a fallback to the dynamic Auto model.

**Step 4:** Save the file and restart the Gateway.

```bash
openclaw gateway start
```
Restarts the background Gateway daemon using the new configuration.

**Step 5:** Verify the model switch is active.

```bash
openclaw status
```
Displays a high-level summary of your Gateway version, ports, and connected channels. Look for "Claude 3.5 (OpenRouter)" under your active models.

#### Common Mistakes to Watch For

- **Malformed JSON:** Forgetting a comma between blocks or leaving an extra comma at the end of the JSON array. **Fix:** Use `python3 -m json.tool ~/.openclaw/openclaw.json` to validate your syntax.
- **Wrong Model Identifier format:** OpenRouter uses `<provider>/<slug>` but OpenClaw also has its own prefixing. For standard OpenClaw routing to OpenRouter you typically map it directly via `openrouter/<provider>/<slug>`. **Fix:** Check OpenRouter's model page for the exact slug if you encounter "Model Not Found" API errors.

---

## Phase 1b Projects

---

### Project 1b: "The Model Shapeshifter"

#### Theory Recap
This project applies Section 1.6 (The Universal Model Router) by leveraging a single credential to access drastically different AI architectures without restarting the gateway.

#### What You'll Build
You will configure OpenClaw to switch between Anthropic, Google, and Meta models on the fly through the Dashboard using OpenRouter.

#### What You'll Learn
- Defining multiple OpenRouter aliases in `openclaw.json`
- Using the `/model` shortcut in the Dashboard

#### Prerequisites
- OpenRouter API key configured (completed in Section 1.6)

#### Step-by-Step Build Guide

1. **Update the models catalog:** Add Google and Meta to your `agents.defaults.models` object in `~/.openclaw/openclaw.json`:
   ```json
   "models": {
     "openrouter/anthropic/claude-3.5-sonnet": { "alias": "Claude 3.5" },
     "openrouter/google/gemini-1.5-pro": { "alias": "Gemini 1.5 Pro" },
     "openrouter/meta-llama/llama-3.1-70b-instruct": { "alias": "Llama 3.1" }
   }
   ```
   *Note: remember to separate entries with commas and ensure valid JSON.*

2. **Apply the configuration:**
   ```bash
   openclaw gateway restart
   ```
   Restarts the gateway to pick up the new model definitions.

3. **Open your Dashboard:**
   ```bash
   openclaw dashboard
   ```
   Opens the local Dashboard UI.

4. **Swap models on the fly:**
   Type `/model` in the chat bar. You will see an autocomplete menu showing "Claude 3.5", "Gemini 1.5 Pro", and "Llama 3.1". Select Llama, press enter, and send a message. Then switch to Gemini and send another.

#### How to Test It
- ✅ Using the `/model` command displays all three models.
- ✅ You receive successful responses from models created by three different corporations, all using a single API key.

#### Common Pitfalls
- **Unverified OpenRouter Limits:** OpenRouter has rate limits for new accounts and credit balances. **Fix:** Ensure your account is funded at openrouter.ai.

#### Stretch Goals
1. Configure `openrouter/openrouter/auto` as your ONLY model, ask it a coding question, and check your OpenRouter activity log online to see which model it intelligently selected to answer you!

---

## Phase 1b — End of Phase Review

### Conceptual Recap Questions

1. Why is OpenClaw treating OpenRouter as a single provider, despite it wrapping dozens of different AI companies' models?
2. What role does `agents.defaults.models` play in `openclaw.json`? Why do we map aliases?

### Practical Consolidation Challenge

**"The Cheap Automation Switch"**: Define a new channel (e.g., Telegram) and use the `groupChat.mentionPatterns` override. Instead of using the primary Claude model, configure the group chat to *exclusively* use `openrouter/meta-llama/llama-3.1-8b-instruct` to save money on noisy group channel interactions, while letting your private Dashboard continue to use Claude 3.5 Sonnet. Try to solve this using `~/.openclaw/openclaw.json` overrides!
