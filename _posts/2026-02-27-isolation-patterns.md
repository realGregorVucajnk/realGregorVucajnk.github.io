---
title: "The Two Patterns of Agent Isolation (and Why You Need a Third)"
date: 2026-02-27
categories: [multi-agent-systems]
tags: [isolation, security, multiclaw, agents]
image:
  path: a1-header.png
  alt: "Abstract visualization of digital isolation and connectivity — three geometric structures evolving from a single cube to an interconnected constellation"
media_subpath: /assets/img/posts/multiclaw-series/
---

> **Series**: Multi-Agent Isolation (Article 1 of 5)

---

## Introduction

Browser Use recently published "How We Built Secure, Scalable Agent Sandbox Infrastructure," and it is one of the best pieces of agent infrastructure writing to date. They walked through their evolution from AWS Lambda sandboxes to Unikraft micro-VMs, arriving at an architecture where a single agent runs in a stripped-down VM with exactly three environment variables and zero secrets. The control plane holds everything. The agent is disposable.

Their architecture is genuinely impressive for single-agent isolation.

But what happens when you need seven agents working together? When an Engineering agent writes code, a QA agent validates it, a PM agent plans the sprint, and an Orchestrator agent governs the whole system — each needing different credentials, different egress rules, and different trust levels? Browser Use's model answers the question "how do I keep one agent from accessing my secrets?" It does not answer the question "how do I prevent seven agents from accidentally accessing each other's secrets, exhausting each other's resources, or making unintended changes to each other's state -- while still letting them collaborate?"

That is the problem I ran into building Multiclaw. The industry had two well-established isolation patterns. I needed a third.

![Three-panel comparison of isolation patterns — Pattern 1 (tool sandbox), Pattern 2 (agent VM), Pattern 3 (ensemble with named roles: Orchestrator, PM, Engineering, QA, Research, Growth, and central Communication Adapter)](a1-three-pattern-triptych.png)

---

## Pattern 1: Isolate the Tool

The simplest model. Your agent runs on your infrastructure — same process as your backend, access to your database credentials, sitting right next to your API keys. When the agent needs to do something dangerous (execute code, run a terminal command, automate a browser), it calls out to a separate sandbox over HTTP. The sandbox has nothing to leak. The code runs, returns results, and the sandbox is destroyed.

**How it works:**

The agent loop runs in your trusted environment. It has access to LLM API keys, database connections, and internal services. A dangerous tool call — say, executing user-provided Python code — gets serialized into an HTTP request and sent to an isolated sandbox. The sandbox is ephemeral: it spins up, runs the code in an environment with no credentials and no network access to internal services, returns the result, and dies.

**Exemplars:** Early Browser Use used Lambda functions as code execution sandboxes. E2B offers sandboxed environments for AI-generated code. Modal provides serverless function sandboxes. In each case, the tool is isolated but the agent is not.

**The problem is everything the sandbox does not cover.** The agent loop itself shares a process with your backend. If the agent makes a reasoning error and issues an unintended API call, it does so with your real credentials. If it gets prompt-injected, the damage is the same. If the agent process leaks memory, your API server slows down. If you need to redeploy your backend, every running agent dies. The trust boundary only covers tool execution, not the agent itself.

For single-agent scenarios where the agent is part of your own infrastructure and you trust it like you trust your own code, this works. Many teams start here and never need to move beyond it.

---

## Pattern 2: Isolate the Agent

Browser Use's current architecture. The entire agent — not just its tools — runs in a sandbox with zero secrets. The agent communicates with the outside world exclusively through a control plane that holds all the real credentials.

**How it works:**

The agent boots inside a Unikraft micro-VM with exactly three environment variables:

- `SESSION_TOKEN` — identifies this agent session
- `CONTROL_PLANE_URL` — where to send all requests
- `SESSION_ID` — ties the agent to its conversation state

That is it. The agent has no LLM API keys, no database credentials, no S3 access tokens. When it needs to make an LLM call, it sends the request to the control plane, which injects the real API key and forwards it to the LLM provider. When it needs to read or write files, the control plane generates presigned S3 URLs with scoped, time-limited access. The agent never touches a real credential.

The control plane itself is a stateless FastAPI service. It holds the credential mapping (session -> real API keys), proxies LLM calls, generates presigned URLs for file transfers, and stores conversation state in a database. The agent is disposable: kill it, and you lose nothing. Suspend it, and it costs near-zero resources. Spin up a new one, and it picks up where the last one left off from control plane state.

**Hardening details matter here.** Browser Use does not stop at isolation boundaries. They compile Python to bytecode and delete the source before the agent runs — so even if an attacker escapes the VM boundary, they find `.pyc` files instead of readable source code. They drop privileges after startup. They strip environment variables from `os.environ` after reading them into memory. These are defense-in-depth measures that assume the primary boundary will be breached.

**The result** is a clean, scalable, single-agent isolation model. Each agent is a disposable VM talking to a trusted control plane. The blast radius of a compromised agent is limited to whatever that single agent session can access through the control plane's scoped proxy.

For single-agent products — chatbots, coding assistants, browser automation agents — this is close to an optimal architecture.

---

## Pattern 3: Isolate the Ensemble (Multiclaw)

Patterns 1 and 2 assume a single agent, or at most multiple independent agents that never need to talk to each other. Multiclaw has seven agents that must cooperate: an Orchestrator governing infrastructure and policy, a PM planning product work, an Engineering agent writing code, a QA agent validating it, a Research/Design agent, a Growth/GTM agent, and a Communication Adapter mediating all traffic. They have different trust levels, different credential needs, and explicit governance relationships.

The question is not "how do I isolate an agent from the outside world?" It is "how do I isolate agents from each other while still letting them work together, and how do I govern the ensemble as a whole?"

**How I built it:**

![Multiclaw ensemble architecture — seven named agents in rootless Podman pods, each with Gateway Sidecar (convenience layer) and nftables (enforcement boundary), mediated by central Communication Adapter](a1-ensemble-architecture.png)

Every agent gets a dedicated Linux user — `orchestrator`, `pm`, `engineering`, `qa_validation`, `research_design`, `growth_gtm`, `comm_adapter`. Each user runs a rootless Podman pod containing two containers: the OpenClaw runtime (where the agent logic runs) and a Gateway sidecar (policy/auth convenience layer). Egress enforcement is anchored in nftables at the rootless-netns boundary, outside the pod namespace. That is 15 containers total across 7 pods.

**Credential isolation is absolute.** Each agent has its own 1Password service account scoped to a single vault containing only that agent's secrets. There is no shared service account. There is no entity in the system — not even the Orchestrator — that holds credentials for all agents. If the Engineering agent makes a mistake -- an unintended API call, an accidental credential exposure in a log, a reasoning error that leads it outside its intended scope -- the blast radius is limited to the Engineering vault. It cannot touch the PM's vault, the Orchestrator's vault, or anyone else's. The same containment holds if an agent is deliberately compromised through prompt injection, but the architecture is designed primarily for the more common case: autonomous agents will make mistakes, and the infrastructure must contain those mistakes.

**Knowledge isolation is per-agent.** Each agent has a dedicated Obsidian knowledge vault — a directory of Markdown files bind-mounted into its pod. While 1Password isolates secrets (API keys, tokens), Obsidian isolates working knowledge: architectural decisions, research findings, accumulated context. No agent can read another agent's vault. The operator has read-only visibility into all vaults for oversight, but agents themselves operate with strict knowledge boundaries. This gives each agent persistent memory without shared state — the same scoping principle applied to knowledge that 1Password applies to credentials.

**Lateral traffic is denied at the kernel level.** Agents cannot talk to each other directly. No TCP connections between pods. No shared filesystem. No shared network namespace. This is not enforced by application-level middleware that could be bypassed — it is enforced by nftables rules outside the pod's network namespace. If an agent's process attempts to reach another agent's endpoint -- whether through a coding error, a misconfigured tool call, or a deliberate escape attempt -- it hits a kernel-level deny with no route to any other agent.

**All inter-agent communication routes through a durable Communication Adapter** — not a stateless proxy like Pattern 2's control plane, but a durable service with at-least-once delivery guarantees, a SQLite WAL-backed message store, dead-letter queues, and static group channels (`all-agents`, `product`, `platform`). Messages are persisted before acknowledgment. Gaps in sequence numbers trigger replay. The adapter is the only path between agents.

**The Orchestrator is itself an untrusted agent.** This is the key differentiator from Pattern 2, where the control plane is trusted infrastructure. In Multiclaw, the Orchestrator runs in the same isolation model as every other agent — dedicated user, rootless pod, scoped vault. Under the Scenario (1) baseline, policy integrity is enforced through root-owned immutable baseline files, overlay version/expiry checks, and approval gates the Orchestrator cannot bypass. Full Sigstore/cosign verification by sidecars is explicitly deferred to Scenario (2+). The Orchestrator cannot modify its own policy overlay. It cannot modify its own Linux user identity. It has append-only access to the audit store — no modify, no delete.

**Governance is architectural, not procedural:**

- **Separation of duties**: Orchestrator proposes, Scenario (1) integrity gates validate (root-owned baseline + version/expiry + approval), Human approves
- **Immutable baselines**: Human-signed policy bundles cannot be modified by any agent
- **Human veto**: Three operational modes — active approval, bounded autonomy (routine actions proceed if human is unreachable for 15 minutes in Scenario 1), safe-halt (system freezes if human is unreachable for 4 hours)
- **Audit integrity**: Scenario (1) baseline uses append-only audit controls; hash-chained tamper evidence is deferred to Scenario (2+)

Pattern 3 subsumes Patterns 1 and 2. Each agent in the ensemble can internally use Pattern 1 or 2 for its own tool execution. Scenario (1) does not require the WASM inner sandbox baseline (deferred to v2), but the architecture leaves room to add it as defense-in-depth without changing the ensemble model.

---

## Comparative Table

| Dimension | Pattern 1: Isolate Tool | Pattern 2: Isolate Agent | Pattern 3: Isolate Ensemble |
|-----------|------------------------|-------------------------|---------------------------|
| **Trust model** | Agent trusted; tool sandbox untrusted | Agent untrusted; control plane trusted | All agents untrusted; no single trusted entity |
| **Credential custody** | Agent holds credentials | Control plane holds all credentials | Per-agent vaults; no entity holds all credentials |
| **Lateral traffic** | N/A (single agent) | N/A (single agent) | Kernel-level deny (nftables); adapter-mediated only |
| **Communication model** | HTTP to sandbox | HTTP to control plane | Durable adapter with at-least-once delivery, WAL, dead-letter queues |
| **Governance** | Application code | Control plane code | Root-owned immutable baselines + approval-gated overlays (cryptographic sidecar verification deferred to Scenario 2+) |
| **Blast radius** | Tool sandbox only | Single agent VM/container | Single agent pod + user namespace |
| **State durability** | Backend database | Control plane database | Communication Adapter SQLite WAL + per-agent session state |
| **Knowledge/state persistence** | Backend database | Control plane database | Per-agent Obsidian vaults (knowledge) + Communication Adapter SQLite WAL (messages) |
| **Orchestrator trust** | N/A | Control plane is trusted infra | Orchestrator is an untrusted, isolated agent |
| **Audit** | Application logs | Control plane logs | Append-only audit baseline (hash chaining deferred to Scenario 2+) |
| **Human oversight** | Implicit (developer) | Implicit (operator) | Explicit (active / bounded autonomy / safe-halt modes) |

---

## Decision Tree

**When to use each pattern:**

**Start with Pattern 1** if you have a single agent executing code or browser actions as part of a larger application. Your agent is trusted — it is your code running your logic. You need to sandbox the dangerous tools (code execution, terminal, browser) so that LLM-generated code cannot access your credentials. Early-stage products, internal tools, and prototypes fit here. E2B, Modal, and early Browser Use all serve this pattern well.

**Move to Pattern 2** if your agent itself is untrusted or you are running agents for multiple tenants. You cannot afford a compromised agent loop having access to real credentials. The agent should be disposable: suspend it, kill it, restart it from control plane state. Browser Use's Unikraft micro-VM architecture is the current gold standard here. Anthropic's computer use sandboxes operate on similar principles.

**Move to Pattern 3** if you have multiple cooperating agents that need different credentials, different trust levels, and explicit governance. The moment you have autonomous agents that must talk to each other while being isolated from each other, Patterns 1 and 2 are insufficient. An Engineering agent that accidentally writes to the wrong API endpoint should not be able to affect the QA agent's state. An Orchestrator that makes a reasoning error about provisioning should not be able to bypass rate limits. You need lateral traffic controls, durable inter-agent communication, per-agent credential isolation, and governance that contains every agent's mistakes -- including the orchestrator's.

**Important: Pattern 3 subsumes Patterns 1 and 2.** Agents within a Pattern 3 ensemble can internally use Pattern 1 or Pattern 2 for their own tool execution. In Multiclaw, Scenario (1) keeps WASM inner sandboxing as deferred v2 hardening; the patterns remain complementary, not competing.

```
Is your agent a single agent?
├── Yes
│   ├── Agent is trusted code (your infra) → Pattern 1
│   └── Agent is untrusted (sandbox needed) → Pattern 2
└── No (multiple cooperating agents)
    └── Need different credentials per agent? → Pattern 3
        └── Need governance over agent collaboration? → Pattern 3
            └── Need blast-radius isolation between agents? → Pattern 3
```

---

## Conclusion

The agent isolation landscape is evolving quickly. Pattern 1 (isolate the tool) is well-established and well-served by platforms like E2B and Modal. Pattern 2 (isolate the agent) has a strong exemplar in Browser Use's Unikraft architecture and is the right choice for most single-agent products shipping today.

Pattern 3 (isolate the ensemble) addresses a problem that most teams have not hit yet: what happens when you need multiple agents, with different trust levels, collaborating under governance? The answer cannot be "run seven Pattern 2 agents and hope they coordinate." You need lateral isolation, durable communication, per-agent secrets, and governance that constrains even the orchestrator.

I built Multiclaw because I needed this. The architecture is open, and the next three articles in this series go deep on the specific layers: credential isolation with per-agent 1Password vaults, network-level lateral deny with nftables, and the durable Communication Adapter that makes inter-agent collaboration possible without shared trust.

The single-agent isolation problem is largely solved. The multi-agent ensemble isolation problem is just getting started.

---

## Series Navigation

- **Article 1: The Two Patterns of Agent Isolation (and Why You Need a Third)** (you are here)
- Article 2: [Seven Users, Seven Vaults: Defense-in-Depth for Multi-Agent Systems](/blog/2026-02-27-defense-in-depth/)
- Article 3: [The Communication Problem: Why Multi-Agent Messaging is Harder Than You Think](/blog/2026-02-27-communication-problem/)
- Article 4: [Governance by Architecture: How to Stop Your Orchestrator From Going Rogue](/blog/2026-02-27-governance-architecture/)
- Article 5: [Two Deployment Paths: When Your Multi-Agent Host Should Leave Your Laptop](/blog/2026-03-06-aws-path-vs-local-host/)
