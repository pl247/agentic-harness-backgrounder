# Hermes Agent: An Agentic Framework for Local LLMs

Hermes Agent is an open-source agentic framework that turns a Large Language Model (LLM) into a capable, self-improving AI agent. It provides the scaffolding (harness) and the logical core (framework) needed for an LLM to observe, reason, act, and learn from feedback—all while running locally on your own hardware.

This guide explains Hermes Agent in generic industry terms, focusing on deployment with a locally hosted LLM (e.g., Nemotron 3 120B) served via **vLLM** on Ubuntu/Linux. Cloud models, paid services, and release-specific details are omitted for clarity.

---

## Core Concept: Agent Harness vs. Framework

- **Agent Harness**: The testing and evaluation environment that gives an LLM the tools and interfaces to act as an agent. It is the sandbox where agents are benchmarked for reliability, safety, and task completion.
- **Agent Framework**: The actual architecture, logic, and "brain" structure for the agent. It manages the **Agentic Loop**:
  ```
  Observation → Reasoning → Action → Feedback
  ```

Hermes Agent combines both: it is a harness that provides tools (web search, terminal, file editing, messaging, etc.) and a framework that implements the agentic loop with memory, skills, personality, scheduling, and self-improvement.

---

## Mental Model: How Hermes Works

At its heart, Hermes Agent operates around a single **conversation loop** (the agentic loop). Any entry point—CLI, messaging gateway, or API—ultimately invokes this loop:

1. **Prompt Assembly**  
   The system prompt is built from:
   - **SOUL.md**: The agent’s core identity (personality, tone, behavioral defaults).
   - **Tool-aware guidance**: Instructions on what tools are available and how to use them.
   - **Memory context**: Persistent user preferences (`USER.md`) and agent notes (`MEMORY.md`).
   - **Skills guidance**: Descriptions of loaded skills (reusable procedures).
   - **Context files**: Project-specific cues like `AGENTS.md`.
   - **Timestamp** and platform formatting hints.

2. **Provider Resolution**  
   Hermes determines which LLM to call and how:
   - For a local vLLM server, this means pointing to the OpenAI-compatible endpoint (`http://localhost:8000/v1`) with no API key required.
   - The resolver selects the appropriate API mode (chat_completions for vLLM/OpenAI-compatible servers).

3. **LLM Inference**  
   The assembled prompt is sent to the LLM (via vLLM). The model generates a response that may include:
   - Plain text (the agent’s reply).
   - Tool calls (requests to execute actions like `web_search`, `terminal`, `read_file`, etc.).

4. **Tool Dispatch & Execution**  
   If the LLM returns tool calls:
   - Hermes validates and routes each call to the appropriate tool implementation.
   - Tools run in sandboxes (local, Docker, SSH, etc.) with strict security hardening.
   - Results are fed back into the conversation loop.

5. **Feedback & Persistence**  
   After the LLM provides a final response:
   - The session is saved to an SQLite database (with FTS5 search) for future recall.
   - Persistent memory (`MEMORY.md`, `USER.md`) may be updated.
   - Successful workflows can be saved as new **skills** for reuse.

This loop repeats until the LLM produces a final response without tool calls. Every feature—memory, skills, cron, self-improvement—attaches to one of these stages.

---

## Five Pillars of Hermes Agent

Hermes Agent’s self-improving capability rests on five interconnected systems:

### 1. Memory (Persistent Long-Term Storage)
- **FILES**: `~/.hermes/memories/MEMORY.md` (agent’s notes) and `~/.hermes/memories/USER.md` (user profile).
- **Purpose**: Stores durable facts that survive sessions—user preferences, environment details, learned conventions, and corrections.
- **LIMITS**: `MEMORY.md` ~2,200 chars; `USER.md` ~1,375 chars (enough for ~800 and ~500 tokens).
- **USAGE**: Updated via the `memory` tool (add, replace, remove). Injected into every session’s system prompt as a frozen snapshot (updated at session start).

### 2. Skills (Reusable Procedures)
- **LOCATION**: `~/.hermes/skills/` (directory of `SKILL.md` files).
- **Purpose**: Encapsulate repeatable workflows (e.g., “review a GitHub PR”, “summarize a research paper”).
- **PROGRESSIVE DISCLOSURE**:  
  - Level 0: `skills_list()` — names and descriptions (low token cost).  
  - Level 1: `skill_view(name)` — full skill content when needed.  
  - Level 2: `skill_view(name, path)` — specific reference file inside a skill.
- **AUTO-GENERATION**: After completing a complex task (5+ tool calls), hitting errors and finding a fix, or receiving user corrections, Hermes can save the approach as a new skill via `skill_manage`.
- **CONDITIONAL ACTIVATION**: Skills can show/hide based on available toolsets (e.g., a DuckDuckGo fallback appears only when the `web` toolset is disabled).

### 3. Soul (Consistent Personality)
- **FILE**: `~/.hermes/SOUL.md`
- **Purpose**: Defines the agent’s identity, tone, communication style, and default behavior. It occupies slot #1 in the system prompt and does not drift between sessions.
- **CUSTOMIZATION**: Edit `SOUL.md` directly to set your preferred personality (concise, technical, creative, etc.). Overlay with `/personality <name>` for temporary shifts.

### 4. Crons (Built-in Scheduling)
- **TOOL**: `hermes cron`
- **Purpose**: Run agent tasks on a schedule (e.g., “check this repo for updates every night at 2 am”).
- **HOW IT WORKS**: Each cron job spawns a fresh agent instance with:
  - A prompt describing the task.
  - Optional attached skills.
  - A delivery target (Telegram, Discord, local log, etc.).
- **PERSISTENCE**: Jobs survive restarts and are managed via `hermes cron list`, `create`, `pause`, `resume`, `remove`.

### 5. Self-Improvement (Reflective Optimization)
- **MECHANISM**: After a session, Hermes can:
  - Analyze its own performance (tool usage, success/failure, token cost).
  - Extract successful patterns into new skills.
  - Adjust memory notes to avoid repeating mistakes.
- **RESULT**: The agent becomes more efficient and effective the more it is used—true “self-improving” behavior.

---

## Local Deployment with vLLM (Nemotron 3 120B)

To run Hermes Agent with a fully private, on-premise LLM:

### 1. Serve the Model with vLLM
```bash
# Install vLLM (requires CUDA-capable GPU)
pip install vllm

# Start the server (adjust model name and context length as needed)
vllm serve nvidia/Nemotron-3-120B-base \
  --port 8000 \
  --host 0.0.0.0 \
  --max-model-len 32768 \
  --enable-auto-tool-choice \
  --tool-call-parser hermes
```
- `--enable-auto-tool-choice` and `--tool-call-parser hermes` are required for the model to understand and execute Hermes’ tool calls.
- Set `--max-model-len` to at least 16k–32k for agent workloads.

### 2. Configure Hermes to Use the Local Endpoint
Edit (or create) `~/.hermes/config.yaml`:
```yaml
model:
  default: nemotron-3-120b   # arbitrary name; must match what you expect
  provider: custom
  base_url: http://localhost:8000/v1
  # api_key omitted – local vLLM typically requires no key
  context_length: 32768      # match vLLM server setting
```

Edit (or create) `~/.hermes/.env` (secrets file):
```bash
# No API key needed for local vLLM; leave empty or omit
# OPENAI_API_KEY=   # not required
```

### 3. Verify and Start Hermes
```bash
hermes doctor   # should show no missing dependencies and a reachable custom endpoint
hermes          # drops you into the interactive chat
```

You can now chat with the agent, and it will use your local Nemotron 3 120B via vLLM for all reasoning. Tools like `web_search`, `terminal`, `read_file`, etc., will still work (they run locally or via sandboxed backends).

### 4. Optional: Adjust Auxiliary Models (for Vision, Web Summarization, etc.)
By default, Hermes uses an auxiliary LLM for side tasks (vision, web extraction, compression). If you only have the local Nemotron model, point these to the same endpoint to avoid silent degradation:
```yaml
auxiliary:
  vision:
    provider: "main"   # reuse the primary model/provider
  web_extract:
    provider: "main"
  compression:
    summary_provider: "main"
```
(Place under an `auxiliary:` block in `config.yaml`.)

---

## Why This Matters: Secure, On-Premise Agentic AI

- **Data Sovereignty**: 100% of data (prompts, tool outputs, files) stays within your firewall—no third-party logging or egress.
- **Zero Marginal Cost**: No per-token fees; only hardware and electricity costs.
- **Performance**: vLLM provides high-throughput serving that rivals cloud APIs, making local agents responsive for real-time use.
- **Security Hardening**:  
  - Run Hermes harness in Docker/Singularity containers to isolate from the host OS.  
  - Use approval modes (`/yolo` off) so sensitive commands (e.g., `rm -rf`, network changes) require manual confirmation.  
  - Secrets (if any) are stored only in `.env` and never logged.

---

## Getting Started Checklist (Ubuntu/Linux)

1. **Install dependencies**: `git`, `python3-pip`, `uv` (optional), `ffmpeg`, `ripgrep`, `nodejs` (for WhatsApp bridge).
2. **Deploy vLLM server** serving Nemotron 3 120B (or another open-weight LLM).
3. **Configure Hermes** (`config.yaml` and `.env`) to point to `http://localhost:8000/v1`.
4. **Run `hermes doctor`** to validate setup.
5. **Launch Hermes** with `hermes` and begin interacting.
6. **Explore**: Try `/skills`, `/tools`, `/goal <task>`, and watch the agent create new skills over time.

---

## Further Reading (Internal)

Once Hermes is running, consult these built-in references:
- `hermes model` – switch providers/models mid-session.
- `hermes tools` – see which toolsets are enabled.
- `hermes skills` – browse, install, or create skills.
- `hermes cron` – schedule automated tasks.
- `hermes memory` – view or edit persistent notes.
- `/goal` – lock the agent onto a target across turns (Ralph-loop pattern).

---

*This documentation distills the essence of Hermes Agent as an agentic framework/harness for local LLMs, stripped of cloud-specific and promotional content. It focuses on the mental model, core components, and practical steps to run Hermes with a privately hosted LLM like Nemotron 3 120B via vLLM on Ubuntu/Linux.*