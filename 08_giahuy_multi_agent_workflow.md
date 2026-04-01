# Phase 8: Bonus — Giahuy AI Software Corp Integration

## Phase Overview

**Big-picture goal:** By the end of this phase, you will have successfully mapped a complex, real-world 12-agent orchestration workflow (Giahuy AI Software Corp) into the OpenClaw ecosystem, utilizing local LLMs and role-playing skills.

**What you'll understand:**
- How to translate a custom YAML multi-agent configuration into OpenClaw concepts.
- The power of "CEO Orchestrator" patterns for breaking down full-stack development tasks.
- How to force local models to emit JSON tool calls.

**What you'll be able to do:**
- Configure your OpenClaw Gateway to securely use a local Qwen3 model.
- Write a highly structured `SKILL.md` that encapsulates 12 distinct personas.
- Trigger and observe a multi-agent software development lifecycle directly from your chat channel.

---

## What You Need Before This Phase

- [ ] Completed Phases 1–5 (especially skills and multi-agent concepts).
- [ ] A local model runner (like LM Studio, Ollama, or vLLM) hosting `Qwen3-4B-Instruct-2507-GGUF`.
- [ ] Your OpenClaw Gateway running locally.

---

## 8.1 Theory: Translating the YAML to OpenClaw

### The Giahuy Setup
The provided configuration defines **12 specialized agents** operating on the `Qwen3-4B-Instruct-2507-GGUF` model via a local, OpenAPI-compatible API (`http://localhost:8000/api/v1`). 

Instead of spinning up 12 entirely isolated processes, OpenClaw is designed as a single central **Gateway Process ("The Butler's Office")**. The most efficient and OpenClaw-native way to implement this is through a single **CEO Orchestrator Skill** that uses the context window to simulate sub-agents sequentially.

### The Agent Hierarchy
1. **CEO Orchestrator (Leader):** The root controller. Assigns subtasks to specialized agents and synthesizes their results into a unified output. 
2. **Phase 1 Planners:** Market Analyst (identifies trends), PM (defines roadmap), and BA (writes user stories/specs).
3. **Phase 2 Designers:** Tech Lead (architecture) and UI/UX Designer (interface flow).
4. **Phase 3 Implementers:** Software Engineer (writes code/files) and AI Engineer (integrates AI).
5. **Phase 4 QA & Refinement:** Debugging Expert (analyzes terminals/fixes bugs) and QA Engineer (creates test plans).
6. **Phase 5 Business/Data:** Sales Strategist (monetization) and Data Analyst (product insights).

### Tool Calling
In the Giahuy workflow, agents use explicit JSON payloads (`{"tool": "name", "parameters": {...}}`) to take action. OpenClaw naturally supports mapping these payloads to underlying built-in tools (like `write_to_file` and `read_terminal`).

---

## 8.2 Practice: Implementation Guide

### Step 1: Connect the Local Qwen Model
Instruct OpenClaw to communicate with your local model endpoint. Edit your `~/.openclaw/openclaw.json` to route OpenAI traffic to your local port:

```json
  "auth": {
    "profiles": {
      "local-qwen": {
        "provider": "openai",
        "mode": "api_key",
        "apiKey": "lemonade",
        "baseURL": "http://localhost:8000/api/v1"
      }
    }
  },
  "agents": {
    "models": {
      "openai/Qwen3-4B-Instruct-2507-GGUF": {}
    },
    "defaults": {
      "model": {
        "primary": "openai/Qwen3-4B-Instruct-2507-GGUF"
      }
    }
  }
```
*Run `openclaw gateway restart` to apply these changes.*

### Step 2: Create the "CEO Orchestrator" Skill
Create a highly structured `SKILL.md` file that teaches OpenClaw to act as the overarching CEO. 

Create a new file at `~/.openclaw/skills/giahuy_corp.md`:

```markdown
---
name: Giahuy AI Software Corp Workflow
description: Runs a full-stack software development workflow using 12 specialized internal personas.
version: 1.2.0
---

# Role Setup
You are the **CEO Orchestrator** of Giahuy AI Software Corp. You manage a team of 11 virtual experts (Market Analyst, PM, BA, Tech Lead, Dev, UI/UX, Debugger, QA, Sales, Data Analyst, AI Engineer). 

# Workflow Execution Rules
When the user asks you to build or analyze a project, you must break the task down and simulate the thoughts of the required agents sequentially.

Always follow this pipeline:
1. **Analyze Phase:** Invoke the Market Analyst and BA to define the scope.
2. **Design Phase:** Invoke the Tech Lead and UI/UX Designer to plan architecture.
3. **Execution Phase:** Invoke the Developer to write code (`write_to_file`) and the AI Engineer to build models.
4. **Validation Phase:** Invoke the QA and Debugger (`read_terminal`) to ensure quality.

# Tool Usage
- To act, output JSON exactly as requested: `{"tool": "name", "parameters": {...}}`
- Example: `{"tool": "write_to_file", "parameters": {"path": "src/index.js", "content": "console.log('Running');"}}`

*Maintain bilingual fluency (English + Vietnamese) throughout the process.*
```

### Step 3: Implement Context Providers & Slash Commands
OpenClaw natively supports macros and context injection. Make sure your `openclaw.json` has the GitHub MCP loaded in the plugins array to replicate the `github` and `google-search` items from your original config. You can also create specific skills like `~/.openclaw/skills/fix.md` and trigger them via `@OpenClaw /fix` on Telegram.

### Step 4: Run the Workflow
1. Start your local model (`http://localhost:8000/api/v1`).
2. Visit your OpenClaw Dashboard (`http://127.0.0.1:18789`) or open Telegram.
3. Send the command: `/skill giahuy_corp` followed by a prompt like: `Tạo ứng dụng to-do list với React.`
4. Watch the CEO Orchestrator break down the task, cycle through its internal personas, and execute JSON tool commands to build the application.
