# Agentic Harness Backgrounder

This repository contains documentation for **Hermes Agent**, an open-source agentic framework that turns a Large Language Model (LLM) into a capable, self-improving AI agent — all running locally on your own hardware.

## 📖 Documentation
- **[HERMES.md](HERMES.md)** – The essence of Hermes Agent explained in generic industry terms, focused on deployment with a locally hosted LLM (e.g., Nemotron 3 120B) served via **vLLM** on Ubuntu/Linux.

## 🚀 Quick Start
1. Serve your local LLM with vLLM (see HERMES.md for details).
2. Configure Hermes to point to `http://localhost:8000/v1`.
3. Run `hermes doctor` to verify setup.
4. Launch Hermes with `hermes` and begin interacting.

## 🔧 Why Hermes?
- **Agentic Framework**: Provides the harness (tools) and framework (reasoning loop) for LLMs to act as agents.
- **Self-Improving**: Writes its own skills from successful workflows.
- **Fully Local**: Zero data egress, zero per-token costs when paired with a locally served model.
- **Secure by Design**: Container isolation, approval modes, and secret hardening.

## 💡 Core Ideas
- **Agent Harness vs. Framework**: Hermes is both — it gives the LLM tools to act and manages the Observation → Reasoning → Action → Feedback loop.
- **Five Pillars**: Memory, Skills, Soul (personality), Crons (scheduling), and Self-Improvement.
- **Mental Model**: Built around a single conversation loop where prompt assembly, provider resolution, LLM inference, tool dispatch, and persistence repeat until a final response is produced.

---

*Documentation maintained in this repo. For the full reference, see [HERMES.md](HERMES.md).*