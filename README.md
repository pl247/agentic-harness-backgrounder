# Agentic Harness Backgrounder

This repository contains documentation for **Hermes Agent**, an open-source agentic framework that turns a Large Language Model (LLM) into a capable, self-improving AI agent — all running locally on your own hardware.

## 📖 Core Concept: Agent Harness vs. Framework

- **Agent Harness**: The testing and evaluation environment that gives an LLM the tools and interfaces to act as an agent. It is the sandbox where agents are benchmarked for reliability, safety, and task completion.
- **Agent Framework**: The actual architecture, logic, and "brain" structure for the agent. It manages the **Agentic Loop**:
  ```
  Observation → Reasoning → Action → Feedback
  ```

Hermes Agent combines both: it is a harness that provides tools (web search, terminal, file editing, messaging, etc.) and a framework that implements the agentic loop with memory, skills, personality, scheduling, and self-improvement.

## 🔐 Why This Matters: Secure, On-Premise Agentic AI

- **Data Sovereignty**: 100% of data (prompts, tool outputs, files) stays within your firewall—no third-party logging or egress.
- **Zero Marginal Cost**: No per-token fees; only hardware and electricity costs when using a locally served LLM (e.g., via vLLM).
- **Performance**: vLLM provides high-throughput serving that rivals cloud APIs, making local agents responsive for real-time use.
- **Security Hardening**:  
  - Run Hermes harness in Docker/Singularity containers to isolate from the host OS.  
  - Use approval modes (`/yolo` off) so sensitive commands (e.g., `rm -rf`, network changes) require manual confirmation.  
  - Secrets (if any) are stored only in `.env` and never logged.

## 🌐 How Agents Change the Networking Dynamic

Traditional LLM usage is request/response: a user asks a question, gets an answer, and the interaction ends. Agentic systems like Hermes fundamentally shift this dynamic:

- **Continuous Traffic**: Agents operate in loops—observing, reasoning, acting, and getting feedback—generating sustained request/response patterns rather than single turns.
- **Tool-Induced Requests**: Each agent turn can trigger dozens of tool calls (web searches, terminal commands, file reads, API requests), multiplying network traffic by 10–100x compared to naive chat.
- **Persistent Connections**: Long-running agent sessions (via CLI, cron jobs, or messaging gateways) keep connections open to local model servers (vLLM), external APIs, and messaging platforms.
- **East-West Traffic Growth**: In on-premise deployments, agents drive significant internal traffic between the agent harness, local LLM servers (vLLM), tool backends (Docker/SSH), and internal services being queried or modified.
- **Predictable Bursts**: Scheduled cron jobs create regular traffic spikes (e.g., nightly repository checks), enabling capacity planning but also requiring bandwidth provisioning for peak loads.
- **New Monitoring Needs**: Traditional LLM monitoring (token count per request) is insufficient; you must now track tool call frequency, loop iterations, and concurrent agent instances to understand real network load.

## 🚀 Quick Start

1. Serve your local LLM with vLLM (see **HERMES.md** for details).
2. Configure Hermes to point to `http://localhost:8000/v1`.
3. Run `hermes doctor` to verify setup.
4. Launch Hermes with `hermes` and begin interacting.

## 📚 Documentation

- **[HERMES.md](HERMES.md)** – Deep dive covering the mental model, five pillars (Memory, Skills, Soul, Crons, Self-Improvement), local vLLM setup, and advanced usage.

## 💡 Key Ideas Distilled

- **Self-Improving**: Hermes writes its own skills from successful workflows, getting smarter with use.
- **Fully Local**: Zero data egress, zero per-token costs when paired with a locally served model.
- **Secure by Design**: Container isolation, approval modes, and secret hardening.
- **Universal Tools**: Out-of-the-box support for web search, terminal execution, file editing, memory, delegation, and 20+ messaging platforms.

---

*Documentation maintained in this repo. For the full reference, see [HERMES.md](HERMES.md).*