---
title: "Seven Users, Seven Vaults: Defense-in-Depth for Multi-Agent Systems"
date: 2026-02-27
categories: [multi-agent-systems]
tags: [defense-in-depth, security, multiclaw, agents]
image:
  path: a2-header.png
  alt: "Seven glowing vault doors arranged in a circle, each a different metallic color with unique keyholes"
media_subpath: /assets/img/posts/multiclaw-series/
---

> **Series**: Multi-Agent Isolation (Article 2 of 5)

---

## Introduction

Browser Use published one of the best write-ups in the agent infrastructure space this year. Their sandbox model strips agent environments down to three variables -- `SESSION_TOKEN`, `CONTROL_PLANE_URL`, `SESSION_ID` -- reads them into Python, then deletes them from `os.environ`. The agent runs inside a Unikraft micro-VM with nothing to leak. It is a genuinely smart approach for single-agent isolation.

Multiclaw goes further: no two agents share *any* credential. And the isolation does not stop at the container boundary.

Browser Use solves a different problem than Multiclaw does. They isolate one agent from the world. Multiclaw isolates seven agents from *each other* -- and from the world -- simultaneously. When you have an Engineering agent, a QA agent, a PM agent, and four others all running on the same host, the threat model shifts. The question is no longer "can the agent escape its sandbox?" It becomes "when one agent makes a mistake -- an unintended API call, a resource leak, a reasoning error that sends it outside its intended scope -- can that mistake cascade to another agent's context, credentials, or communication channel?"

I do not unconditionally trust any of these agents. Not because they are malicious, but because autonomous agents operating on LLMs *will* make mistakes. They will misinterpret instructions. They will call the wrong endpoint. They will attempt operations they were not meant to attempt. The architecture must contain those mistakes at the boundary of the agent that made them.

My answer is defense-in-depth: six isolation layers stacked so that any single agent's failure -- accidental or deliberate -- is contained by the remaining layers. No single layer needs to be perfect. The blast radius of any single agent's mistakes just needs to be limited to that agent alone.

---

## The Onion Model

Most agent platforms draw one strong boundary around the agent and call it done. A VM boundary. A container boundary. A VPC network policy. These are good boundaries. But they are *single* boundaries.

Multiclaw stacks six layers of isolation. A compromised agent must breach all of them to reach another agent's data:

```
Layer 5: Per-Agent 1Password Vaults (credential isolation)
Layer 4: nftables Deny-by-Default Egress (network enforcement)
Layer 3: Communication Adapter Mediation (traffic routing)
Layer 2: Rootless Podman Pods (container isolation)
Layer 1: Dedicated Linux Users (identity isolation)
Layer 0: Host Baseline (physical/OS foundation)
```

Compare this to Browser Use's model: the agent sits inside a Unikraft micro-VM, and the control plane sits outside it. That gives you two boundaries -- the VM boundary and the VPC network policy separating the VM from the control plane. Both are strong. But there are two of them, and they protect a single agent.

In Multiclaw's model, each of the seven agents sits inside its own onion. Peeling one layer exposes the next.

![Concentric ring comparison — Browser Use's single VM boundary vs Multiclaw's six uniquely labeled isolation layers with Scenario (1) bearer-token auth annotation](a2-onion-comparison.png)

---

## Layer 1: Linux Users

The foundation of Multiclaw's isolation model is the oldest trick in the Unix playbook: separate users.

Multiclaw provisions seven dedicated Linux users, one per agent role:

- `orchestrator`
- `pm`
- `engineering`
- `qa_validation`
- `research_design`
- `growth_gtm`
- `comm_adapter`

Each user has non-overlapping subuid/subgid ranges of 65,536 IDs. This means the rootless user namespaces for each agent map to completely disjoint ID ranges on the host. The `pm` user's UID 0 inside its container maps to host UID 165536. The `engineering` user's UID 0 maps to host UID 231072. They cannot access each other's files, processes, or IPC resources at the kernel level.

No shared execution user exists. The Communication Adapter runs as its own user. The Orchestrator runs as its own user. Every agent is a first-class Linux citizen with its own home directory, its own systemd user instance (via `loginctl enable-linger`), and its own process tree.

This matters because Linux user isolation is enforced by the kernel. It does not depend on container runtime correctness. It does not depend on policy configuration. It is the floor beneath every other layer.

---

## Layer 2: Rootless Podman Pods

Each agent runs inside a rootless Podman pod -- a pod launched by an unprivileged user, with no root daemon involved. Each pod has a fixed two-container composition:

1. **OpenClaw Runtime** -- the agent framework container
2. **Gateway Sidecar** -- the egress policy enforcement container

The hardening profile for every container is identical and aggressive:

| Control | Setting | Purpose |
|---------|---------|---------|
| Root filesystem | Read-only | No persistent writes to container image layers |
| Privilege escalation | `no-new-privileges` | Blocks setuid/setgid binaries |
| Capabilities | `--cap-drop=ALL` | Zero Linux capabilities by default |
| Core dumps | `LimitCORE=0` | No crash dumps that might leak secrets |
| File descriptors | `LimitNOFILE=1024` | Bounded resource consumption |
| Seccomp | Custom restrictive profile | Blocks `mount`, `ptrace`, `clone` (with `CLONE_NEWUSER`), `unshare`, and namespace-manipulation syscalls |
| AppArmor | Per-agent confinement profile | Mandatory access control on file and network operations |

The Seccomp profile deserves attention. By blocking `mount`, `ptrace`, `clone` with namespace flags, and `unshare`, this closes the most common container escape vectors. An attacker who gains code execution inside the container cannot create new namespaces, trace other processes, or mount new filesystems. They are stuck in a read-only box with no capabilities and no debugging tools.

**How does this compare to Browser Use?** Browser Use uses Unikraft micro-VMs, which provide stronger per-agent isolation than containers -- a VM boundary is harder to escape than a container boundary. But that stronger boundary comes at higher resource cost. Multiclaw trades slightly weaker per-agent boundaries for the ability to run seven agents on a single host with modest hardware (32-128 GB RAM), while compensating with the additional layers described below.

---

## Layers 3-4: Network Isolation

Network isolation in Multiclaw operates at two levels that work together but have very different trust properties.

### Layer 3: Communication and Knowledge Mediation

Layer 3 encompasses two sub-concerns that together govern how agents exchange information and retain state:

**Layer 3a: Communication Adapter Mediation.** All inter-agent traffic flows through the Communication Adapter -- a durable message routing service with at-least-once delivery, WAL-backed persistence, dead-letter queuing, and sequence-gapped replay. Agents cannot send messages directly to each other. The adapter is the only path.

This is not a stateless proxy like Browser Use's control plane. It is a durable mediator that maintains thread continuity, enforces delivery guarantees, and provides backpressure (bounded per-agent queues of 500 messages with overflow to dead-letter). Messages survive adapter restarts via SQLite WAL.

**Layer 3b: Obsidian Knowledge Vaults.** Each agent has a dedicated Obsidian knowledge vault -- a directory of Markdown files filesystem-bind-mounted into its pod. The agent has read-write access to its own vault only; the operator has read-only access to all vaults. Isolation is enforced by the same Linux user ownership mechanism described in Layer 1. The Communication Adapter handles transient operational messages; Obsidian vaults handle persistent knowledge -- decisions, research findings, and context that outlive individual message threads.

### Layer 4: nftables Deny-by-Default Egress

Here is where the real enforcement lives. Every agent user has nftables rules applied at the network namespace level that implement deny-by-default egress. The only allowed destinations are explicitly listed in per-agent allowlists.

Critical details:

- **DNS port 53 is blocked** (both UDP and TCP). DNS is an uncontrolled exfiltration channel. Agents use static `/etc/hosts` files with pre-resolved addresses for approved destinations.
- **Direct lateral pod-to-pod traffic is denied.** One agent cannot open a TCP connection to another agent's pod, period.
- **The gateway sidecar is a convenience layer; nftables is the enforcement boundary.** This is a direct response to hardening finding F-01: all containers in a Podman pod share the same network namespace, so a sidecar's iptables redirect rules can be bypassed. The nftables rules are applied *outside* the pod at the rootless-netns level, where individual containers cannot manipulate them.

This distinction matters. If the gateway sidecar is compromised or misconfigured, the nftables rules still enforce the policy. The sidecar makes policy convenient to manage. The kernel makes it real.

---

## Layer 5: Per-Agent 1Password Vaults

This is where Multiclaw diverges most sharply from the single-agent model.

In Browser Use's architecture, one set of credentials lives in the control plane and serves all agent sessions. The control plane is the trusted broker. This works well for their model because all agents are ephemeral, stateless, and interchangeable -- they do not need distinct identities or distinct secret sets.

Multiclaw's agents are not interchangeable. The Engineering agent needs GitHub tokens. The Growth agent needs analytics API keys. The PM agent needs project management credentials. No agent should have access to another agent's secrets.

Multiclaw's secrets model:

- **Each agent role has a dedicated 1Password service account** scoped to a single vault containing only that agent's secrets.
- **No shared service account.** The Orchestrator cannot read the Engineering agent's vault. The PM cannot read the QA agent's vault.
- **No human-account token delegation.** The operator's personal 1Password account is never used for agent operations.
- **Token lifecycle:** each service account token has a 180-day TTL (Scenario 1 baseline) with scheduled rotation. Abnormal access patterns trigger alerting.
- **Runtime integration:** OpenClaw's `SecretRef` system uses an `exec` provider that wraps `op read`, so secrets are retrieved at runtime from 1Password rather than stored in environment variables or files.

The result: regardless of what goes wrong with a single agent -- a reasoning error, a resource leak, or even a full container escape -- the damage is bounded to that agent's credentials alone. There is no golden key that unlocks all vaults.

Credential isolation is complemented by knowledge isolation. Each agent's accumulated knowledge -- decisions, research findings, context -- lives in a dedicated Obsidian vault owned by the agent's Linux user. While 1Password protects secrets (API keys, tokens), Obsidian vaults protect working knowledge. Both follow the same per-agent scoping principle: no cross-agent access.

![Credential flow comparison — Browser Use's 3 shared variables vs Multiclaw's per-role named vaults (orchestrator, pm, engineering, qa, research, growth, comm_adapter) with 180-day TTL annotation](a2-credential-flow.png)

---

## The Layering Math

Defense-in-depth is not about any one layer being impenetrable. It is about ensuring that an agent's mistakes -- or, in the worst case, a deliberate compromise -- cannot cascade beyond that agent's boundaries.

Consider what happens when the Engineering agent goes wrong. Maybe it makes an unintended API call due to a reasoning error. Maybe it attempts to access a file path outside its workspace. Maybe it exhausts its memory allocation. Maybe, in the worst case, it has been compromised through prompt injection. The layers respond the same way regardless of cause:

1. **The container boundary holds.** The read-only root filesystem, Seccomp filters, AppArmor confinement, zero capabilities, and no-new-privileges prevent the mistake from affecting anything outside the container. An unintended file write fails silently. A resource leak is bounded by cgroup limits.

2. **The user namespace holds.** Even if the agent's process somehow escapes the container, it lands as the `engineering` Linux user, not as root. The `pm` user's files and processes are owned by a completely different UID range. The mistake cannot cross user boundaries.

3. **The network policy holds.** nftables rules block all egress except the approved allowlist. An unintended outbound connection attempt -- whether from a misconfigured tool call or a deliberate exploit -- is denied at the kernel level. DNS is blocked, so even reconnaissance is impossible.

4. **The adapter authentication holds.** The Communication Adapter verifies per-agent bearer tokens. The Engineering agent's token cannot be used to send messages as the PM agent, whether by accident or by design.

5. **The credential vault holds.** The `engineering` service account is scoped to the `engineering` vault. Even if every other layer failed, 1Password's server-side access controls deny reads to any other vault.

Each layer contains a different class of failure. Together, they guarantee that one agent's problem stays one agent's problem. This is equally true whether the root cause is a reasoning error at 2pm or a sophisticated prompt injection attack at 3am.

Compare this to the Browser Use model: escape the Unikraft micro-VM, then bypass VPC network policy. Two strong boundaries. But in a multi-agent context, the question is not "which single boundary is stronger?" It is "can one agent's failure affect another agent?" Multiclaw's answer is: five independent containment layers, each enforced by a different subsystem, each containing a different class of failure.

---

## Conclusion

Defense-in-depth is not about any one layer being perfect. It is about ensuring that when an autonomous agent makes a mistake -- and autonomous agents *will* make mistakes -- the blast radius is contained to that single agent.

Browser Use's approach is excellent for what it does: isolating single agents with a strong VM boundary and a minimal credential surface. I respect the engineering behind it. But multi-agent systems introduce cross-agent contamination as a failure class that single-agent isolation does not address.

Multiclaw's six-layer model -- Linux users, rootless Podman pods, communication adapter mediation, nftables enforcement, and per-agent 1Password vaults, all on a hardened host baseline -- means that one agent's failure stays one agent's failure. An unintended API call does not leak another agent's credentials. A resource leak does not degrade another agent's performance. A reasoning error does not corrupt another agent's state. And yes, a deliberate compromise does not enable lateral movement either.

That is the promise of defense-in-depth: not perfection at any single layer, but guaranteed containment across all of them.

---

## Series Navigation

- Article 1: [The Two Patterns of Agent Isolation (and Why You Need a Third)](/blog/2026-02-27-isolation-patterns/)
- **Article 2: Seven Users, Seven Vaults: Defense-in-Depth** (you are here)
- Article 3: [The Communication Problem: Why Multi-Agent Messaging is Harder Than You Think](/blog/2026-02-27-communication-problem/)
- Article 4: [Governance by Architecture: How to Stop Your Orchestrator From Going Rogue](/blog/2026-02-27-governance-architecture/)
- Article 5: [Two Deployment Paths: When Your Multi-Agent Host Should Leave Your Laptop](/blog/2026-03-06-aws-path-vs-local-host/)
