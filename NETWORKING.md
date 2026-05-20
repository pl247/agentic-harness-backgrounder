# Networking Traffic Flow: Hermes Agent with vLLM and Nemotron 3 120B

What Actually Flows Across Your GPU Fabric

A practical look at network traffic in distributed LLM inference


You've built a solid platform. Two hosts, each with a pair of L40S GPUs, dual 100G to the frontend network, and dual 100G on the backend wired up for RoCEv2. You're running Nemotron-3 120B in FP8 on vLLM, orchestrated by a Ray cluster, with a Hermes agent on top driving the workload.

The natural question: what is this thing actually going to push across each network?

Let's walk through it.


## The Cast of Characters

Before we talk traffic numbers, it's worth understanding that the two networks in this design carry completely different kinds of traffic, with very different characteristics.

### The Frontend Network (dual 100G, north-side)
This network handles:

    Hermes agent ↔ vLLM API calls (HTTP/JSON)
    Ray dashboard, control RPCs, scheduling heartbeats
    Tool calls reaching out to retrieval, search, code execution
    Operational traffic: SSH, Prometheus scrapes, log shipping


It speaks in kilobytes and the occasional megabyte, runs over TCP, and is generally well-behaved.

### The Backend Network (dual 100G, east-west, RoCEv2)
This network carries one thing: NCCL collectives between GPUs that don't share a PCIe root complex. It moves gigabytes per second, bypasses the kernel via RDMA, and is extremely sensitive to loss and latency.

Both are 100G, but they will see very different volumes. Here's why.


## What Actually Happens When a Token Is Generated

When Hermes sends a prompt to vLLM, this is the chain of events:

    Hermes POSTs ~10 KB of JSON to the vLLM OpenAI-compatible endpoint on host 1. (frontend, microseconds)
    The vLLM API server hands the request to a Ray actor — the scheduler. (frontend, microseconds)
    The scheduler dispatches work to four GPU worker actors, two on each host. (frontend, ~MB)
    The GPUs begin the actual inference work.


For every transformer layer — and Nemotron-3 120B has roughly 88 of them — the math goes like this:

    Each GPU computes its slice of attention → all-reduce to combine results
    Each GPU computes its slice of the MLP → all-reduce again


That's two all-reduces per layer × 88 layers = 176 all-reduces per token, per token, per request.

This is the work that lives or dies on your backend RoCE fabric.


## Doing the Math

A tensor-parallel all-reduce moves an activation tensor sized roughly:



hidden_size × dtype_bytes  ≈  12,288 × 2  =  24 KB per token per all-reduce


A quick note on precision: even though your weights are FP8, the all-reduce itself is typically performed in BF16 for numerical stability. FP8 saves you VRAM, but it doesn't shrink the activation traffic on the wire.

A ring all-reduce isn't free — each participant sends approximately 2× the payload around the ring. So per token:



176 all-reduces × 24 KB × 2 (ring overhead) ≈ 8.4 MB of NCCL wire traffic per token


With TP=4 across two hosts, roughly half of those collectives traverse the inter-host link:



~4 MB per generated token crosses the backend network


Now let's plug in some realistic workloads.

### Scenario A: A single user, light agent loop

Hermes is generating at ~30 tokens/sec (reasonable for FP8 on L40S with decent batching):



30 tok/s × 4 MB = 120 MB/s ≈ 1 Gbps sustained


Your dual 100G backend is barely engaged — about 0.5% of one link. A 10 GbE link would handle this steady-state load. The catch is that prefill behaves very differently.

### Scenario B: Prefill burst on a 16K-token prompt

Prefill is where things get interesting. It processes the entire prompt in parallel, in one large compute phase.



16,000 tokens × 4 MB/token ≈ 64 GB of NCCL traffic


Modern GPUs, properly fed, will work through that prefill in 1–3 seconds. Which means the backend sees:



~20–60 Gbps burst, sustained for a couple of seconds


This is the real reason you have 100G RoCE. Not for the average — for the prefill spike. A 25G link would be saturated here, prefill latency would stretch, GPUs would stall waiting for activations, and end-to-end tokens-per-second would drop noticeably.

### Scenario C: Concurrent users (the production case)

vLLM's continuous batching is excellent, and it scales NCCL traffic close to linearly with the number of active sequences. With, say, 16 concurrent agent sessions each generating at 20 tok/s:



16 × 20 × 4 MB = 1.28 GB/s ≈ 10 Gbps sustained on the backend


Add prefill bursts as new sessions arrive, and those can stack to 40–80 Gbps.

This is where dual 100G earns its keep. The goal isn't to size for the average — it's to make sure the GPUs never wait for the network, because idle GPU time at this price point is the most expensive resource in the rack.


## Meanwhile, on the Frontend Network

Looking at the same 16 concurrent agent sessions, each issuing ~10 LLM calls per task with growing context:

    Average request payload: ~30 KB
    Average streamed response: ~5 KB
    Ray internal control chatter: a few MB/s aggregate
    Tool call fan-out: typically <100 Mbps


Total frontend traffic, even under heavy agent load:



~50–200 Mbps, possibly 500 Mbps if tools are particularly chatty


That's a fraction of a percent of your dual 100G frontend. And that's fine — the frontend wasn't sized for bandwidth. It was sized for redundancy, low latency, and headroom for the inevitable future use cases (large RAG embeddings, batch ingest, model updates) that will eventually push real volume through it.


## Topology Matters More Than Bandwidth

Here's an important nuance: how you split the model across the four GPUs changes the traffic profile dramatically.

### Option 1: TP=4 across both hosts (what we modeled above)

    Every layer's all-reduce crosses the inter-host link
    Backend traffic: high and sustained
    Per-token latency: lowest
    Throughput per request: highest
    Network performance directly affects token latency


### Option 2: TP=2 within each host, PP=2 across hosts

    All-reduces stay on PCIe inside each host
    Inter-host traffic reduces to pipeline activations only — essentially one hidden state per micro-batch per stage boundary
    Backend traffic drops by roughly an order of magnitude
    Trade-off: pipeline bubbles, more complex scheduling, slightly higher latency unless batching is tuned well


For an L40S setup without NVLink between boxes, Option 2 is often the better choice if you can tolerate marginally higher latency. The 100G RoCE backend becomes significantly over-provisioned in that case, which gives you generous headroom for growth.

If you stay with TP=4, you'll genuinely use the 100G capacity. Both are valid designs — the key is choosing deliberately.


## The Cheat Sheet

Here's what to expect on your dashboards under a moderately busy production workload, assuming TP=4 across hosts:

Network 	Steady-state 	Peak (prefill bursts) 	Primary contributor
Frontend (dual 100G) 	50–500 Mbps 	1–2 Gbps 	HTTP, Ray RPC, tools
Backend RoCE (dual 100G) 	5–15 Gbps 	40–80 Gbps 	NCCL all-reduces

With TP=2 / PP=2 instead:

Network 	Steady-state 	Peak 	Primary contributor
Frontend 	50–500 Mbps 	1–2 Gbps 	Same as above
Backend RoCE 	200–800 Mbps 	2–5 Gbps 	Pipeline activations


## Summary

The frontend network provides comfort, redundancy, and future-proofing. It will rarely be stressed by the agent or API traffic itself.

The backend network is what determines whether your GPUs run efficiently. Every byte of NCCL traffic that doesn't arrive on time is GPU compute that goes unused — and at L40S prices, that adds up quickly.

The priority order for distributed inference networking:

    Inter-host NCCL collectives (gigabytes/sec under load) — the dominant factor
    KV cache memory pressure (not network, but the next big constraint)
    Ray and HTTP traffic (a small fraction of total)
    Operational and monitoring traffic (smaller still)


Design around #1, and the rest takes care of itself.