# Learning OpenClaw

Welcome to the definitive curriculum for mastering the OpenClaw AI Gateway. OpenClaw allows you to build, deploy, and scale autonomous AI agents across platforms like Telegram, Discord, and WhatsApp.

This repository tracks a 6-phase journey from basic installation to building advanced, multi-agent systems. By following this curriculum, you will transform from an OpenClaw beginner to an advanced power user capable of deploying autonomous research teams and contributing to the open-source software.

## Curriculum Roadmap

### [Phase 1: Foundation & Onboarding](01_phase1_foundation.md)
**Goal:** Prove you can get OpenClaw running and interacting.
- Understand the core OpenClaw architecture (Gateway, Daemon, Channels, Skills).
- Clone, install, and configure OpenClaw locally.
- Successfully send and receive your first message through a Telegram bot.
- Learn basic troubleshooting for environment and connection issues.

### [Phase 2: Channel Mastery](02_phase2_channels.md)
**Goal:** Understand how OpenClaw interacts with the outside world.
- Master the abstraction layer that allows the same AI to operate on Telegram, WhatsApp, and Discord.
- Configure safe direct messaging (DM) behavior and identity verification.
- Understand how OpenClaw behaves in group chats (mentions, context windows).
- Differentiate between synchronous and asynchronous messaging.

### [Phase 3: Memory & Built-in Tools](03_phase3_memory_tools.md)
**Goal:** Give your AI persistence and agency.
- Understand how OpenClaw creates and manages persistent memory across sessions.
- Grant your agent the ability to execute terminal shell commands safely.
- Enable file system reading and writing.
- Configure web browsing capabilities so your agent can search and scrape the internet.

### [Phase 4: Skills, Plugins & Automation](04_phase4_skills_automation.md)
**Goal:** Transform a generic chatbot into a specialized, automated worker.
- Write custom `SKILL.md` files to define highly specialized agent behaviors.
- Leverage multiple LLM providers (OpenAI, Anthropic, local models via Ollama) for different tasks.
- Understand the Plugin architecture (Action providers, Evaluators).
- Set up autonomous, scheduled tasks using CRON jobs and webhook triggers (e.g., automated Kaggle training).

### [Phase 5: The Power User](05_phase5_power_user.md)
**Goal:** Scale your OpenClaw deployment for production workloads.
- Orchestrate multi-agent setups where different agents communicate and divide labor.
- Deploy OpenClaw to remote cloud servers (AWS, DigitalOcean) securely.
- Connect local execution environments to cloud agents via tools like Ngrok.
- Implement advanced security hardening, secrets management, and permission boundaries.

### [Phase 6: Mastery & Contribution](06_phase6_mastery.md)
**Goal:** Become an active contributor to the OpenClaw ecosystem.
- Build and publish your own custom Node.js plugins for OpenClaw.
- Integrate complex real-world APIs into the agent's action space.
- Fork the OpenClaw repository, navigate its core source code (`core/`, `plugin-*/`), and submit pull requests.
- Author advanced, complex skill frameworks.

## Getting Started

To begin, proceed to **[Phase 1: Foundation & Onboarding](01_phase1_foundation.md)** and follow the installation instructions.

---

*This curriculum is designed to be project-based. Do not merely read the documents; build the systems described in each phase.*
