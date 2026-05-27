# Agentic Harness Backgrounder

This repository contains documentation for **Hermes Agent**, an open-source agentic framework that turns a Large Language Model (LLM) into a capable, self-improving AI agent — all running locally on your own hardware.

## 🌐 Why Agentic AI Matters — and What It Means for the Network

Agentic AI marks a shift from "AI as a tool you call" to "AI as a system that acts." Instead of single prompt-response exchanges, agents plan, decompose problems, call tools, observe results, and iterate toward a goal. The implications compound quickly.

### What Makes It Significant

The model is no longer the product. The product is the model plus tools, memory, and a planner. Four properties drive the shift:

1. **Autonomy over orchestration**. Traditional automation requires humans to define every branch. Agents reason about which branch to take, handling ambiguity and long-horizon tasks.
2. **Tool use as a first-class capability**. Once an LLM reliably calls APIs, queries databases, and chains results, it stops being a chatbot and becomes a worker.
3. **Composability**. Agents spawn sub-agents and specialize. A research agent hands off to a code agent, which hands off to a verifier — mirroring how human teams scale.
4. **Persistent state**. Memory across sessions lets agents own long-running workflows end-to-end.

### How It Evolves the Space

Copilots become coworkers. SaaS apps become tool surfaces — MCP servers and function APIs — rather than UIs, and whoever owns the agent layer owns the user relationship. New failure modes follow: hallucinated tool calls, runaway loops, prompt injection, and goal misalignment become operational risks, not research curiosities. Evaluation also gets harder; judging a 40-step trajectory with branching decisions is nothing like judging a single response, which is why trace-based observability is suddenly load-bearing.

Network Impact: Swarms of Agents On-Prem

Traffic patterns change fundamentally. A single user query can generate dozens to hundreds of internal calls — agent to model, agent to tool, agent to vector DB, agent to agent. East-west fan-out replaces simple request-response. Each hop sits on the critical path of a reasoning loop, so tail latency on any link stalls the whole trajectory. Low-latency fabric, RDMA between GPU nodes, and NIC-to-GPU affinity matter end-to-end, not just inside the training cluster. Agent memory and KV-cache locality also break stateless load balancing; session-aware routing becomes the norm.

Security posture has to evolve in parallel. Each agent and sub-agent needs its own workload identity, scoped credentials, and audit trail — SPIFFE/SPIRE-style patterns fit well. Per-agent egress policy matters more than per-namespace, since agents will try to reach tools they shouldn't. Tool outputs fetched over the network must be treated as untrusted input, because prompt injection is now a network-adjacent threat. A read-only research agent and a write-to-prod agent should not share reachability, even when they share a model backend.

On-prem deployment is what makes this tractable at scale. Sensitive data stays inside the perimeter. Agentic workloads burn ten to a hundred times more tokens than chat, so sustained throughput on owned GPUs beats per-token API pricing quickly. Co-locating model, tools, and data tightens reasoning loops, and version control over models matters when agents have side effects.


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

For a detailed analysis of the actual network traffic flows in a distributed LLM inference setup with Hermes Agent driving the workload, see [NETWORKING.md](NETWORKING.md).

## 😄 Joke of the Day

Why don't networks ever get lost?

Because they always follow the protocol!

## 🚀 Quick Start

1. Serve your local LLM with vLLM (see **HERMES.md** for details).
2. Configure Hermes to point to `http://localhost:8000/v1`.
3. Run `hermes doctor` to verify setup.
4. Launch Hermes with `hermes` and begin interacting.

## 📚 Documentation

- **[HERMES.md](HERMES.md)** – Deep dive covering the mental model, five pillars (Memory, Skills, Soul, Crons, Self-Improvement), local vLLM setup, and advanced usage.
- **[NETWORKING.md](NETWORKING.md)** – Practical analysis of network traffic flow in distributed LLM inference with Hermes Agent driving workloads on vLLM-served models.

## 💡 Key Ideas Distilled

- **Self-Improving**: Hermes writes its own skills from successful workflows, getting smarter with use.
- **Fully Local**: Zero data egress, zero per-token costs when paired with a locally served model.
- **Secure by Design**: Container isolation, approval modes, and secret hardening.
- **Universal Tools**: Out-of-the-box support for web search, terminal execution, file editing, memory, delegation, and 20+ messaging platforms.

---

*Documentation maintained in this repo. For the full reference, see [HERMES.md](HERMES.md).*