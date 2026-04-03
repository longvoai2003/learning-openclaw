# 🦞 OpenClaw — ResearchSwarm × Computer Vision
## Fully Autonomous Multi-Agent Pipeline via OpenRouter

---

> **Vision:** A fully autonomous pipeline where you type `/research "train YOLOv9 on custom dataset"` in Discord or Telegram and the AI Research Swarm handles **everything** — paper collection, analysis, GPU scouting, GPU rental, SSH environment setup, monitoring, and cleanup. The **only** step humans do manually is **writing the training script** with AI pair-programming (Antigravity / Claude Opus), because that is real engineering you want to own.

---

## 🧭 Pipeline Philosophy: Roles

| Role | Who | What They Do |
|------|-----|-------------|
| **Research AI Swarm** | 🤖 Autonomous Agents (via OpenRouter) | Paper gathering, analysis, critique, report writing, GPU scouting, GPU rental, SSH setup, monitoring, cleanup |
| **Human Engineer** | 👨‍💻 You + AI Pair-Programming | Writing the training setup script only (dataset config, model config, hyperparameters, launch command) |

> **Why this split?** The training script is the **creative engineering decision** — choosing the model architecture, tuning hyperparameters, structuring the dataset pipeline. Everything else (infrastructure, rental, SSH, monitoring) is deterministic automation that agents handle better than humans.

---

## 📐 System Architecture Overview

```
Discord / Telegram
        │
        ▼
  ┌─────────────────────────────────────────────────────┐
  │              OpenClaw Bot (Webhook Layer)            │
  │         /research [CV topic or train task]           │
  └────────────────────┬────────────────────────────────┘
                       │
                       ▼
  ┌─────────────────────────────────────────────────────┐
  │           ClaWHub Orchestrator (Brain)               │
  │  - Agent Routing      - Shared Memory/State          │
  │  - Task Queue         - Agent Handoff Contracts      │
  └──┬──────────┬──────────┬──────────┬─────────────────┘
     │          │          │          │
     ▼          ▼          ▼          ▼
 [Scout]   [Analyst]  [Critic]   [Writer]      ← 🤖 AI Swarm (OpenRouter)
     │
     ▼
 [VastAI GPU Scout]  ← 🤖 Scouts & reports GPU pricing
     │
     ▼
  ════════════════════════════════════════
  ║  Human picks Offer ID → /rent ID   ║
  ════════════════════════════════════════
     │
     ▼
 [VastAI Rental Agent]  ← 🤖 Rents the human's chosen GPU
     │
     ▼
  ════════════════════════════════════════
  ║  Human uploads training script      ║
  ║  (coded with AI pair-programming)   ║
  ════════════════════════════════════════
     │
     ▼
 [SSH & Network Agent]  ← 🤖 Autonomous env setup + launches script
 [Monitor Agent]        ← 🤖 Reports training progress to chat
 [Cleanup Agent]        ← 🤖 Downloads model, destroys instance
```

---

## 🧩 Agent Roster

| # | Agent | Role | Model (via OpenRouter) | Est. Cost/Run |
|---|-------|------|----------------------|---------------|
| 1 | **Scout** | Web search, CV paper gathering | `mistralai/mistral-7b-instruct` | ~$0.01 |
| 2 | **Analyst** | Summarize papers, cross-reference | `qwen/qwen-2.5-72b-instruct` | ~$0.03 |
| 3 | **Critic** | Fact-check, flag contradictions | `meta-llama/llama-3.1-8b-instruct` | ~$0.01 |
| 4 | **Writer** | Final polished report | `anthropic/claude-sonnet-4` | ~$0.05 |
| 5 | **GPU Scout** | Scout VastAI market, rank offers | `meta-llama/llama-3.1-8b-instruct` | ~$0.01 |
| 6 | **Rental Agent** | Rent human's chosen GPU via VastAI CLI | Python Skill (no LLM needed) | $0 |
| 7 | **SSH Agent** | Remote env setup with auto-recovery | `qwen/qwen-2.5-coder-7b-instruct` | ~$0.02 |
| 8 | **Monitor Agent** | Training progress → Discord/Telegram | `mistralai/mistral-7b-instruct` | ~$0.02 |
| 9 | **Cleanup Agent** | Download model, destroy instance | Python Skill (no LLM needed) | $0 |

> **All LLM inference** is via **OpenRouter API** — no local GPU required.
> Your GTX 1650 laptop is just the bot host. All the heavy thinking happens in the cloud.

---

## 📦 Tech Stack

| Layer | Technology |
|-------|-----------|
| Bot Interface | discord.py / python-telegram-bot |
| Agent Orchestration | ClaWHub + LangGraph or CrewAI |
| **LLM Inference** | **OpenRouter API (all agents)** |
| GPU Rental | VastAI CLI (`vastai` Python SDK) |
| Remote Execution | Paramiko (SSH) / Fabric |
| CV Training | PyTorch, Ultralytics YOLO, MMDetection, HuggingFace Transformers |
| Monitoring | TensorBoard + Custom FastAPI Dashboard |
| Notifications | Discord Webhooks / Telegram Bot API |
| Memory/State | Redis + SQLite (via ClaWHub) |
| Config Management | `.env` + Vault (optional) |

---

## 🔌 Shared OpenRouter Client

All agents share a single helper for calling OpenRouter:

```python
# openrouter_client.py
import httpx
import os

OPENROUTER_API_KEY = os.getenv("OPENROUTER_API_KEY")
OPENROUTER_URL = "https://openrouter.ai/api/v1/chat/completions"

async def call_openrouter(model: str, system_prompt: str, user_message: str) -> str:
    """Shared OpenRouter caller used by ALL agents"""
    async with httpx.AsyncClient(timeout=120) as client:
        response = await client.post(
            OPENROUTER_URL,
            headers={"Authorization": f"Bearer {OPENROUTER_API_KEY}"},
            json={
                "model": model,
                "messages": [
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": user_message}
                ]
            }
        )
    return response.json()['choices'][0]['message']['content']
```

---

---

# ═══════════════════════════════════════════
# 🤖 AUTONOMOUS AGENTS (OpenRouter-Powered)
# ═══════════════════════════════════════════

> Everything below runs autonomously. The only human interaction is:
> 1. Choosing a GPU Offer ID after scouting
> 2. Providing the training script (coded with AI pair-programming)

---

# 🔵 PHASE 1 — Research Swarm (CV Intelligence Gathering)

> Triggered by: `/research [Computer Vision topic]`
> Goal: Produce a comprehensive, cited CV research briefing

## 1.1 Scout Agent

**Responsibility:** Web search, ArXiv scraping, GitHub trending repos

```python
# scout_agent.py
from arxiv import Search, SortCriterion
from openrouter_client import call_openrouter

SCOUT_SYSTEM = """
You are a Computer Vision research scout. Your job is to:
1. Find the 5 most relevant recent papers on the given topic from ArXiv
2. Find top GitHub repos (stars > 500) related to the topic
3. Find benchmark leaderboard results (Papers With Code)
4. Return structured JSON with: papers[], repos[], benchmarks[]
Respond ONLY in JSON. No preamble.
"""

async def run_scout(topic: str) -> dict:
    results = list(Search(
        query=f"computer vision {topic}",
        max_results=5,
        sort_by=SortCriterion.SubmittedDate
    ).results())

    papers = [{"title": r.title, "abstract": r.summary[:500], "url": r.entry_id} for r in results]

    return await call_openrouter(
        model="mistralai/mistral-7b-instruct",
        system_prompt=SCOUT_SYSTEM,
        user_message=f"Topic: {topic}\nPapers found: {papers}"
    )
```

## 1.2 Analyst Agent

**Responsibility:** Summarize, cross-reference, identify trends

```python
# analyst_agent.py
from openrouter_client import call_openrouter

ANALYST_SYSTEM = """
You are a senior Computer Vision researcher and analyst.
Given a list of papers and repos, you must:
1. Summarize each paper in 3 bullet points
2. Identify common themes and emerging trends
3. Compare methodologies (e.g., YOLO vs. Transformer-based detectors)
4. Highlight the SOTA model and its benchmark scores
5. Suggest which approach best fits: real-time inference vs. accuracy-first
Output as structured markdown sections.
"""

async def run_analyst(scout_output: dict) -> str:
    return await call_openrouter(
        model="qwen/qwen-2.5-72b-instruct",
        system_prompt=ANALYST_SYSTEM,
        user_message=f"Analyze these findings:\n{scout_output}"
    )
```

## 1.3 Critic Agent

**Responsibility:** Fact-check claims, flag hallucinations or outdated info

```python
# critic_agent.py
from openrouter_client import call_openrouter

CRITIC_SYSTEM = """
You are a rigorous CV research critic and peer reviewer.
For each claim in the analysis:
1. Flag any claim that seems unverified or potentially hallucinated
2. Check if benchmark scores match known leaderboards
3. Note if any paper is cited incorrectly
4. Rate confidence: HIGH / MEDIUM / LOW for each finding
5. Return a JSON critique: { issues: [], confidence_map: {}, overall_rating: "X/10" }
"""

async def run_critic(analyst_output: str) -> dict:
    return await call_openrouter(
        model="meta-llama/llama-3.1-8b-instruct",
        system_prompt=CRITIC_SYSTEM,
        user_message=analyst_output
    )
```

## 1.4 Writer Agent

**Responsibility:** Compile everything into a Discord/Telegram-ready briefing

```python
# writer_agent.py
from openrouter_client import call_openrouter

WRITER_SYSTEM = """
You are a technical writer specializing in Computer Vision.
Compile the research findings, analysis, and critique into a clean,
professional briefing. Format for Discord (use ** for bold, ``` for code).
Include: Executive Summary, Key Papers, SOTA Models, Recommended Approach,
Open Source Repos, and Next Steps.
Keep it under 1800 characters for Discord or split into multiple messages.
"""

async def run_writer(analyst_output: str, critic_output: dict) -> str:
    return await call_openrouter(
        model="anthropic/claude-sonnet-4",
        system_prompt=WRITER_SYSTEM,
        user_message=f"Analysis:\n{analyst_output}\n\nCritique:\n{critic_output}"
    )
```

---

# 🟡 PHASE 2 — VastAI GPU Scout (Market Intelligence)

> Triggered automatically after research concludes with a training recommendation
> Goal: **Scout the GPU market and report best options with Offer IDs**

## 2.1 GPU Scout Agent

```python
# gpu_scout_agent.py
import subprocess
import json
from openrouter_client import call_openrouter

GPU_SCOUT_SYSTEM = """
You are a GPU market intelligence agent. Your job is to:
1. Search VastAI for the best GPU offers matching the training requirements
2. Rank by: price-performance ratio, reliability, VRAM, network speed
3. Present a structured comparison table with OFFER IDs clearly visible
4. Recommend the best option with justification
5. Estimate total training cost based on expected training duration

DO NOT rent any instances. Report only.
Return as structured JSON: { recommendations: [], estimated_cost: {}, best_pick: {} }
"""

async def scout_gpu_market(
    gpu_name: str = "RTX_3090",
    min_vram_gb: int = 24,
    max_price_per_hr: float = 0.50,
    min_reliability: float = 0.95
) -> dict:
    """Search VastAI offers and produce a market report — NO RENTING"""
    cmd = [
        "vastai", "search", "offers",
        f"gpu_name={gpu_name}",
        f"gpu_ram>={min_vram_gb}",
        f"dph_total<={max_price_per_hr}",
        f"reliability>={min_reliability}",
        "rentable=true",
        "--raw"
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    offers = json.loads(result.stdout)

    return await call_openrouter(
        model="meta-llama/llama-3.1-8b-instruct",
        system_prompt=GPU_SCOUT_SYSTEM,
        user_message=f"Available offers:\n{json.dumps(offers[:15], indent=2)}\n\nTask: YOLOv9 training, ~100 epochs, estimated 2-4 hours on single GPU."
    )
```

## 2.2 GPU Scout Report Output (Example)

The GPU Scout produces a report like this and sends it to Discord/Telegram:

```
📊 VastAI GPU Market Report
━━━━━━━━━━━━━━━━━━━━━━━━━━

| Rank | Offer ID  | GPU         | VRAM  | $/hr  | Reliability | Location |
|------|-----------|-------------|-------|-------|-------------|----------|
| 1    | `102948`  | RTX 3090    | 24GB  | $0.32 | 99.2%       | EU-West  |
| 2    | `893421`  | RTX 4090    | 24GB  | $0.45 | 98.7%       | US-East  |
| 3    | `441092`  | A100 40GB   | 40GB  | $0.89 | 99.5%       | US-West  |

🏆 Recommended: Offer `102948` (RTX 3090 @ $0.32/hr)
💰 Estimated cost: $0.96 - $1.28 (3-4 hours training)
⚠️  Offer valid as of scan time — prices change rapidly

👉 To rent, type: `/rent 102948`
```

---

# 🟠 PHASE 3 — VastAI GPU Rental Agent (Automated)

> **Input:** Human types `/rent 102948` in the Discord/Telegram chat
> **Output:** Running GPU instance with SSH credentials

### The Human-in-the-Loop Workflow
1. The human reviews the **GPU Scout Report** (Phase 2).
2. The human selects an **Offer ID** from the table (e.g., `102948`).
3. The human tells OpenClaw: `/rent 102948`
4. The **Rental Agent** (a Python Skill, no LLM needed) picks up the ID and executes the VastAI CLI.

```python
# rental_agent.py — OpenClaw Skill (pure Python, no LLM)
import subprocess
import json
import time

def rent_instance(offer_id: int, docker_image: str = "pytorch/pytorch:2.1.0-cuda11.8-cudnn8-runtime") -> dict:
    """Rent a VastAI GPU instance"""
    cmd = [
        "vastai", "create", "instance", str(offer_id),
        "--image", docker_image,
        "--disk", "50",
        "--ssh", "--direct", "--raw"
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return json.loads(result.stdout)

def wait_for_instance_ready(instance_id: int, timeout: int = 300) -> dict:
    """Poll until instance status is 'running'"""
    for _ in range(timeout // 10):
        cmd = ["vastai", "show", "instance", str(instance_id), "--raw"]
        result = subprocess.run(cmd, capture_output=True, text=True)
        info = json.loads(result.stdout)
        if info.get('actual_status') == 'running':
            return info
        time.sleep(10)
    raise TimeoutError(f"Instance {instance_id} did not start in time")

def destroy_instance(instance_id: int):
    """Destroy instance after training is complete"""
    subprocess.run(["vastai", "destroy", "instance", str(instance_id)])
```

---

# ════════════════════════════════════════════════════
# 👨‍💻 HUMAN STEP: Write Training Script (AI Pair-Programming)
# ════════════════════════════════════════════════════

> **This is the ONLY manual step.** You write the training setup script with Antigravity / Claude Opus.
> This is the creative engineering — choosing architecture, hyperparameters, dataset pipeline.

### What You Code (with AI help)

```python
# training_setup.sh — example script you write and upload
#!/bin/bash

# Dataset setup
mkdir -p /workspace/datasets
wget -O /workspace/datasets/coco128.zip https://ultralytics.com/assets/coco128.zip
cd /workspace/datasets && unzip coco128.zip

# Launch training
yolo detect train \
    model=yolov9c.pt \
    data=/workspace/datasets/coco128/data.yaml \
    epochs=100 \
    batch=16 \
    imgsz=640 \
    project=/workspace/runs \
    name=exp \
    device=0 \
    exist_ok=True \
    plots=True
```

> After writing this, you tell OpenClaw: `/train` or *"Start training with my script"*
> The SSH Agent takes it from there.

---

# 🟣 PHASE 4 — SSH & Network Agent (Automated via OpenRouter)

> **Input:** `{ssh_host, ssh_port, instance_id, training_script}`
> **Output:** Fully configured remote environment with training running

**Responsibility:** Remote env setup, dependency installation, executing the human's training script, with auto-error recovery powered by a coder LLM via OpenRouter.

```python
# ssh_agent.py
import paramiko
import time
import json
from pathlib import Path
from openrouter_client import call_openrouter

SSH_SYSTEM = """
You are an expert SSH, Linux, and deep learning infrastructure engineer.
You know:
- Setting up CUDA/cuDNN environments
- Installing PyTorch, Ultralytics, MMDetection, HuggingFace
- Configuring tmux sessions for persistent training
- Diagnosing and fixing CUDA OOM errors, dependency conflicts
Always return shell commands as a JSON array: {"commands": ["cmd1", "cmd2"]}
"""

class SSHAgent:
    def __init__(self, host: str, port: int, username: str = "root", key_path: str = "~/.ssh/id_rsa"):
        self.host = host
        self.port = port
        self.username = username
        self.key_path = Path(key_path).expanduser()
        self.client = paramiko.SSHClient()
        self.client.set_missing_host_key_policy(paramiko.AutoAddPolicy())

    def connect(self, retries: int = 5):
        for attempt in range(retries):
            try:
                self.client.connect(
                    hostname=self.host, port=self.port, username=self.username,
                    key_filename=str(self.key_path), timeout=30
                )
                return
            except Exception:
                time.sleep(10 * (attempt + 1))
        raise ConnectionError("Failed to connect via SSH")

    def run(self, command: str) -> tuple[str, str, int]:
        stdin, stdout, stderr = self.client.exec_command(command, timeout=120)
        return stdout.read().decode(), stderr.read().decode(), stdout.channel.recv_exit_status()

    async def ask_llm_for_commands(self, task_description: str, error_context: str = "") -> list[str]:
        """Ask OpenRouter coder LLM to generate or fix bash commands"""
        prompt = task_description
        if error_context:
            prompt += f"\n\nThe previous command failed with error:\n{error_context}\nPlease provide commands to fix this and continue."

        response = await call_openrouter(
            model="qwen/qwen-2.5-coder-7b-instruct",
            system_prompt=SSH_SYSTEM,
            user_message=prompt
        )
        data = json.loads(response)
        return data.get('commands', [])

    async def execute_with_auto_recovery(self, task_description: str, max_retries: int = 3):
        """Executes a task autonomously. If it fails, asks the LLM to fix it."""
        commands = await self.ask_llm_for_commands(task_description)

        for attempt in range(max_retries):
            success = True
            for cmd in commands:
                out, err, code = self.run(cmd)
                if code != 0:
                    commands = await self.ask_llm_for_commands(task_description, error_context=err)
                    success = False
                    break
            if success:
                return
        raise RuntimeError("Auto-recovery failed after max retries")
```

---

# 🟢 PHASE 5 — Monitor Agent (Automated via OpenRouter)

> **Input:** Running training session on remote GPU
> **Output:** Ongoing context-aware updates back to the human chat

### How It Works (Reverse Polling via OpenClaw Skill)
1. OpenClaw periodically SSHs into the GPU instance.
2. It fetches TensorBoard data and runs `nvidia-smi`.
3. It sends the raw metrics to the Monitor LLM via OpenRouter.
4. The LLM produces a **natural language training update** and chats with you in Discord/Telegram.
5. You can ask `/status` anytime for a live update.

```python
# monitor_agent.py
from openrouter_client import call_openrouter

MONITOR_SYSTEM = """
You are a CV training monitor. Given raw training metrics and GPU stats,
produce a concise, friendly Discord message. Include:
- Current epoch / total epochs
- Key metrics (mAP, losses) with trend arrows (↑ ↓ →)
- GPU health (utilization, memory, temperature)
- Actionable advice if something looks wrong
Keep it under 1000 characters. Use Discord markdown formatting.
"""

async def generate_training_report(metrics: dict, gpu_status: dict, epoch: int, total: int) -> str:
    return await call_openrouter(
        model="mistralai/mistral-7b-instruct",
        system_prompt=MONITOR_SYSTEM,
        user_message=f"Epoch: {epoch}/{total}\nMetrics: {metrics}\nGPU: {gpu_status}"
    )
```

### Example Monitor Output in Discord

```
📊 Training Update — Epoch 50/100
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📈 mAP@50: 0.652 ↑  |  mAP@50-95: 0.441 ↑
📉 Box Loss: 0.031 ↓  |  Cls Loss: 0.018 →

⚡ GPU: 94% util | 18.2/24 GB | 72°C
💡 Looking healthy! Loss is still dropping. Let it cook 🍳

Type /status for a fresh update or /stop to end early.
```

---

# 🔴 PHASE 6 — Model Download & Cleanup (Automated)

> Triggered automatically when training completes or `/stop` is called

```python
# cleanup_agent.py — Pure Python Skill
import subprocess

def download_model(ssh_agent, local_dir: str = "./trained_models"):
    """Download best.pt via SCP"""
    subprocess.run([
        "scp", "-P", str(ssh_agent.port),
        "-i", str(ssh_agent.key_path),
        f"{ssh_agent.username}@{ssh_agent.host}:/workspace/runs/exp/weights/best.pt",
        f"{local_dir}/best.pt"
    ])

def full_cleanup(ssh_agent, instance_id: int):
    """Kill sessions, destroy GPU instance"""
    ssh_agent.run("tmux kill-server")
    ssh_agent.client.close()
    from rental_agent import destroy_instance
    destroy_instance(instance_id)
```

---

---

# 💬 Discord/Telegram Bot Commands

| Command | Function | Automated? |
|---------|----------|-----------|
| `/research [topic]` | Trigger full research swarm | 🤖 Fully auto |
| `/gpu-scout [gpu] [budget]` | Scout VastAI market, report offers | 🤖 Fully auto |
| `/rent [offer_id]` | Rent the chosen GPU instance | 🤖 Auto (human picks ID) |
| `/train` | Upload & launch training script | 🤖 Auto (human wrote script) |
| `/status` | Get live training metrics + GPU health | 🤖 Fully auto |
| `/stop` | Emergency stop + download model + cleanup | 🤖 Fully auto |

---

---

# 💰 Cost Summary (All via OpenRouter)

| Component | Model | Est. Cost/Run |
|-----------|-------|--------------|
| Scout Agent | `mistral-7b-instruct` | ~$0.01 |
| Analyst Agent | `qwen-2.5-72b-instruct` | ~$0.03 |
| Critic Agent | `llama-3.1-8b-instruct` | ~$0.01 |
| Writer Agent | `claude-sonnet-4` | ~$0.05 |
| GPU Scout | `llama-3.1-8b-instruct` | ~$0.01 |
| SSH Agent | `qwen-2.5-coder-7b-instruct` | ~$0.02 |
| Monitor Agent (×5 reports) | `mistral-7b-instruct` | ~$0.02 |
| Rental + Cleanup Agents | Python Skills (no LLM) | $0 |
| **OpenRouter Total** | | **~$0.15/run** |
| **VastAI GPU** (RTX 3090 × 3hrs) | | **~$1-3/run** |
| VPS hosting (bot always on) | Hetzner CX21 | $6/month |
| **Grand Total per run** | | **~$1.50 - $3.50** |

> Compare to: AWS p3.2xlarge = $3.06/hr + setup overhead ≈ **$50-200/run**

---

# 📁 Project Structure

```
openclaw-cv-pipeline/
├── agents/                  ← 🤖 Autonomous (OpenRouter-powered)
│   ├── openrouter_client.py ← Shared API client
│   ├── scout_agent.py
│   ├── analyst_agent.py
│   ├── critic_agent.py
│   ├── writer_agent.py
│   ├── gpu_scout_agent.py
│   ├── ssh_agent.py
│   └── monitor_agent.py
├── skills/                  ← 🤖 Python Skills (no LLM needed)
│   ├── rental_agent.py
│   └── cleanup_agent.py
├── human/                   ← 👨‍💻 Scripts you write with AI pair-programming
│   └── training_setup.sh   ← Your training script
├── bots/
│   ├── discord_bot.py
│   └── telegram_bot.py
├── core/
│   ├── orchestrator.py
│   ├── notifications.py
│   └── state_manager.py
├── configs/
│   ├── models.yaml
│   └── vastai_filters.yaml
├── .env                     ← OPENROUTER_API_KEY, VASTAI_API_KEY, etc.
├── requirements.txt
└── README.md
```

---

# 🎯 Implementation Order

### Phase 1 — Research Swarm (all OpenRouter)
1. `openrouter_client.py` — shared API caller
2. Scout Agent → ArXiv + GitHub search
3. Analyst Agent → Paper analysis
4. Critic Agent → Fact-checking
5. Writer Agent → Report generation

### Phase 2 — GPU Pipeline (automated)
6. GPU Scout Agent → Market intelligence with Offer IDs
7. Rental Agent → Python Skill for VastAI CLI
8. SSH Agent → Autonomous environment setup & auto-recovery
9. Monitor Agent → Reverse-polling training reports

### Phase 3 — Human Training Script (pair-programming)
10. Write `training_setup.sh` with Antigravity / Claude Opus

### Phase 4 — Orchestrator & Bot Commands
11. Cleanup Agent → Download model, destroy instance
12. Full orchestrator wiring
13. Discord/Telegram bot commands (`/research`, `/rent`, `/train`, `/status`, `/stop`)
14. FastAPI dashboard (stretch goal)

---

*Built with 🦞 OpenClaw × ClaWHub × OpenRouter — Autonomous CV Research & Training Pipeline*
