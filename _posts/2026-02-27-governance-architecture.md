---
title: "Governance by Architecture: How to Stop Your Orchestrator From Going Rogue"
date: 2026-02-27
categories: [multi-agent-systems]
tags: [governance, security, multiclaw, agents]
image:
  path: a4-header.png
  alt: "A chess board with an AI king piece constrained by architectural barriers and a human hand holding veto power"
media_subpath: /assets/img/posts/multiclaw-series/
---

> **Series**: Multi-Agent Isolation (Article 4 of 4)

---

## Introduction

Browser Use's control plane is deterministic Python code. It routes tasks to browser instances, manages session lifecycle, handles retries. It does exactly what it is programmed to do. You can read the source, trace the logic, and predict every decision it will ever make. The control plane is trusted because it is inert.

Multiclaw's Orchestrator is not inert. It is an AI agent. It runs on an LLM. It makes autonomous decisions about infrastructure provisioning, policy configuration, and agent lifecycle management. It owns Linux identity boundaries (Layer 1), runtime orchestration (Layer 2), policy distribution (Layer 5), and bootstrap provisioning (Layer 6). It decides which agents exist, what policies govern them, which network endpoints they can reach, and when to spin up new capacity.

Now ask the uncomfortable question: what happens when it makes a mistake?

Not "what if it goes rogue." Not "what if an attacker jailbreaks it." The more likely scenario: what if it makes a routine reasoning error at 3am and provisions an agent with overly permissive network egress? What if it miscalculates a policy overlay and accidentally grants broader credential access than intended? What if it enters a reasoning loop and provisions agents faster than a human can review?

These are not exotic attacks. These are the everyday failures of autonomous systems operating on LLMs. The Orchestrator will process thousands of decisions. Some of them will be wrong. The architecture must ensure that wrong decisions are caught, contained, and reversible.

This concern extends to deliberate compromise too -- prompts can be jailbroken, behavioral guardrails can be circumvented, and the history of LLM safety research is a history of bypasses. But the primary design motivation is simpler and more honest: I do not unconditionally trust any autonomous agent, including the Orchestrator, not to make mistakes. The architecture exists to contain those mistakes.

The thesis of this article --- and the culminating argument of this series --- is simple: when the orchestrator is AI-powered, governance must be architectural. Separation of duties, immutable baselines, human veto. Enforced by infrastructure, not by instructions.

---

## The Orchestrator Problem

Let us be precise about the threat model. The Orchestrator owns four of the seven layers in Multiclaw's architecture:

- **Layer 1 (Linux Identity)**: Creates dedicated Linux users per agent role
- **Layer 2 (Runtime Orchestration)**: Manages rootless Podman pods and systemd user units
- **Layer 5 (Policy Distribution)**: Constructs and attaches policy overlays per agent within Scenario (1) integrity controls
- **Layer 6 (Provisioning)**: Bootstraps PM and execution agent pods, registers communication channels

Without architectural checks, an Orchestrator that makes a mistake -- or is deliberately compromised -- could:

1. **Accidentally over-provision** --- a reasoning error leads it to spin up agents with overly permissive policy overlays, creating infrastructure with broader access than intended
2. **Inadvertently expand its own access** --- a policy overlay construction error grants it broader egress, additional tool capabilities, or elevated network reach that it was never meant to have
3. **Corrupt the audit trail** --- if it controls the audit store, a software bug or deliberate tampering could modify or delete entries, destroying the human's ability to review what happened
4. **Overwhelm governance capacity** --- provision agents faster than a human can review, whether through a reasoning loop, a misconfigured rate, or intentional evasion
5. **Accumulate unintended knowledge** --- if it could access other agents' knowledge vaults, even accidental reads would leak intelligence about other agents' reasoning, research, and context

Notice that every item above has both an accidental and a deliberate variant. The architecture handles both, but the accidental variants are far more likely. An Orchestrator will make thousands of provisioning decisions. Some will be wrong. The infrastructure must catch the wrong ones.

This is not a novel attack pattern. This is the classic separation-of-duties violation documented in NIST SP 800-53 AC-5: the entity that performs an action must not be the sole entity that authorizes and audits that action. Every compliance framework, from SOC 2 to FedRAMP, encodes this principle. Multiclaw simply applies it to a domain where the "entity" happens to be an AI agent rather than a human employee.

The CI/CD analogy makes this concrete. You do not trust your deployment pipeline not to push broken code to production. Not because Jenkins is malicious -- because pipelines are automated systems that will, eventually, execute something you did not intend. A misconfigured branch rule. A test that passes locally but fails in CI. A merge conflict that slips through. You put branch protection rules on main that the pipeline cannot bypass. You require code review approvals that the pipeline cannot forge. The pipeline has the mechanical capability to deploy anything, anywhere. The architecture makes unintended deployment physically impossible -- whether the cause is a bug, a misconfiguration, or a deliberate attack. That is the standard we need for AI orchestrators: contain the mistakes, and you contain the attacks as a byproduct.

---

## Separation of Duties by Design

![Separation of duties — five controls constraining the Orchestrator with dual labels: Scenario (1) ownership/permission integrity and Scenario (2+) Sigstore/cosign verification](a4-separation-of-duties.png)

Multiclaw enforces five architectural controls that make Orchestrator overreach structurally impossible, not merely discouraged.

### (a) Orchestrator proposes, independent integrity gates verify

The Orchestrator constructs policy overlays for each agent. But overlays do not take effect on construction. In Scenario (1), overlay integrity is enforced by root-owned policy files, explicit approval gates, and overlay version/expiry checks that the Orchestrator cannot bypass.

Scenario (2+) adds independent Sigstore/cosign cryptographic verification by sidecars. That future control is important, but it is deferred and should not be represented as active in Scenario (1).

Additionally, each overlay carries a monotonic version counter and an expiry timestamp. The Gateway rejects overlays with version counters lower than the currently applied version (rollback prevention) and overlays past their expiry (staleness prevention). Even if an attacker replayed a previously valid overlay, it would fail both checks.

### (b) Orchestrator cannot modify its own identity or policy

The Orchestrator MUST NOT modify its own policy overlay or its own Linux user identity. Full stop. This is a hard architectural constraint, not a soft behavioral guideline.

Why does this matter? Because self-modification is the canonical path to privilege escalation. If the Orchestrator can rewrite its own policy, it can grant itself broader egress rules, additional tool capabilities, or elevated provisioning authority. By making self-modification structurally impossible at the Linux identity and policy verification layers, this eliminates the entire class of self-escalation attacks.

Knowledge vault ownership follows the same separation-of-duties principle: each agent's Obsidian vault is owned by its Linux user. The Orchestrator cannot read or modify another agent's accumulated knowledge, preventing knowledge poisoning as an attack vector. The Orchestrator cannot gather intelligence from other agents' research findings or decision records --- another boundary limiting its ability to accumulate undue influence.

### (c) Immutable human-owned baseline

The baseline policy bundle is human-owned and immutable by any agent, including the Orchestrator. In Scenario (1), this is enforced by Unix ownership and file permissions (`root:root` read-only for agent users). This baseline defines the foundational constraints: which agent roles exist, what their identity boundaries are, what the maximum provisioning rate is, what egress destinations are categorically forbidden.

Think of this as the constitution. Agents operate within it. They cannot amend it. Only the human can issue a new baseline.

### (d) Append-only audit with no Orchestrator delete (Scenario 1 baseline)

All provisioning actions emit audit events to a store where the Orchestrator has append-only access. No modify. No delete. Scenario (1) baseline uses append-only paths and retention controls. Hash-chained tamper-evident audit is a stronger Scenario (2+) control, and should be tracked as deferred.

This means the Orchestrator can be as creative as it wants with provisioning. Every action is permanently recorded in a chain that it cannot alter after the fact. The audit trail is the human's evidence base, and the Orchestrator cannot tamper with it any more than a defendant can edit court transcripts.

### (e) Provisioning rate limits and circuit breakers

Even with all the above controls, velocity matters. An Orchestrator operating within its legitimate authority but at abnormal speed can cause damage before a human reviews the audit trail.

Multiclaw enforces three rate-limiting mechanisms:

- **Maximum N agents per hour** (configurable) --- hard cap on provisioning velocity
- **Roster boundary enforcement** --- provisioning any agent beyond the declared roster requires explicit human approval
- **Anomalous pattern circuit breaker** --- unusual provisioning patterns (burst creation, rapid teardown-and-rebuild cycles, off-hours activity) trigger an automatic freeze and human alert

The circuit breaker is the critical piece. It means that even if the Orchestrator finds a creative way to operate within each individual rate limit, anomalous aggregate behavior still triggers a halt.

---

## Three Operational Modes

Architectural controls are necessary but not sufficient. The system also needs a governance model for the human-in-the-loop that handles the reality that humans are not always available.

### Active Mode

All policy changes and novel provisioning require human approval before execution. The Orchestrator proposes, the human approves or vetoes, the system acts. This is the tightest governance posture.

In practice, Active mode is how the system operates during initial deployment, after security incidents, and during any period where the human wants direct oversight of every decision.

### Bounded Autonomy Mode

If the human is unreachable for more than 15 minutes (configurable), the system enters Bounded Autonomy. Pre-approved routine actions --- things like restarting a crashed agent pod, rotating a scheduled credential, re-registering a channel after adapter recovery --- proceed automatically. Novel actions that have no pre-approved template queue in a bounded queue and wait for the human.

This is the steady-state operating mode. The system handles routine operational maintenance without human intervention, but any genuinely new decision waits.

### Safe-Halt Mode

If the human is unreachable for more than 4 hours (configurable), the system enters Safe-Halt. All non-critical changes freeze. No new provisioning. No policy modifications. Agents continue operating under the last-known-good policy. The system is in maintenance mode, not dead --- existing agents still serve their functions --- but no architectural state changes occur.

Safe-Halt is the fail-safe. It reflects a core design principle: when in doubt, do nothing new. A frozen system with stale-but-valid policy is safer than an autonomous system making unchecked decisions.

### Delegation Tokens (Scenario 2+ Option)

For situations where the human wants to grant temporary expanded authority, delegation tokens are a Scenario (2+) option. A delegation token is a signed artifact with three properties: scope constraints (what actions it authorizes), expiry (when it becomes invalid), and the human's cryptographic signature.

The Orchestrator can use a delegation token to take actions that would normally require Active mode approval, but only within the token's scope and only before its expiry. This gives the human a way to say "I trust you to handle credential rotations for the next 8 hours while I sleep" without granting permanent blanket authority.

---

## The Governance Hierarchy

![Governance hierarchy with three operational modes — ACTIVE (full oversight, human approval required), BOUNDED AUTONOMY (human unreachable >15 min, routine proceeds), SAFE-HALT (human unreachable >4 hours, non-critical changes freeze)](a4-governance-hierarchy.png)

The authority hierarchy in Multiclaw is: **Human Governor > Orchestrator > PM > Execution Agents**.

But this is deeply misleading if you read it as a permission hierarchy. It is not. It is a constraints hierarchy. And the constraints flow in the opposite direction from what you might expect.

The Orchestrator has MORE constraints than execution agents, not fewer. It is the most watched entity in the system. It is the most audited. It is the most restricted. Every action it takes is permanently recorded in append-only audit paths. It cannot modify its own policy. It cannot bypass Scenario (1) policy ownership controls. It cannot exceed provisioning rate limits without triggering a circuit breaker.

Execution agents, by contrast, have relatively simple constraint profiles: follow your policy overlay, communicate only through the adapter, stay within your egress allowlist. They have fewer constraints because they have less power. The Orchestrator has more constraints precisely because it has more power. Constraint is proportional to authority.

There is another critical feature of this hierarchy: execution agents can refuse PM assignments. If a PM assigns a task that would require the execution agent to violate its policy overlay constraints --- say, accessing a network endpoint outside its allowlist, or modifying files outside its workspace --- the agent can and must refuse. Refusal triggers mandatory escalation to both the PM and the Human Governor. The PM cannot override a policy-based refusal without human approval.

This means governance flows in both directions. The hierarchy is not a command chain where orders flow down. It is a constraint mesh where violations are caught at every level.

---

## Why This Matters Beyond Multiclaw

Every multi-agent system with an AI orchestrator faces this problem. It does not matter whether you are building on LangGraph, CrewAI, AutoGen, or a custom framework. The moment you put an LLM in charge of provisioning, policy, or agent lifecycle decisions, you have an entity that might exceed its intended authority.

The pattern I am advocating is simple: **make governance architectural, not behavioral**.

Behavioral governance means relying on system prompts, RLHF alignment, and output filtering to keep the orchestrator in line. These are valuable. They are also insufficient -- not primarily because of adversarial bypass, but because behavioral controls cannot prevent the Orchestrator from making honest mistakes within its authorized scope. A perfectly aligned Orchestrator that makes a reasoning error about policy construction produces the same damage as a jailbroken one. System prompts cannot prevent reasoning errors. RLHF alignment cannot eliminate edge cases in decision-making. Output filters cannot catch a logically valid but contextually wrong provisioning decision. Behavioral controls are a defense-in-depth layer, not a primary control.

Architectural governance means enforcing constraints through infrastructure that the AI cannot modify. nftables rules that block unauthorized egress cannot be jailbroken. Root-owned policy files and approval gates in Scenario (1) cannot be socially engineered by prompt text. Append-only audit stores cannot be retroactively edited by the Orchestrator. Linux user isolation that prevents cross-boundary file access cannot be bypassed through clever prompting.

The CI/CD analogy is the clearest way to explain this to engineering teams. Consider your deployment pipeline. It has credentials to push to production. It has access to your container registry. It has the mechanical capability to deploy any artifact to any environment at any time. You do not rely on the pipeline's "alignment" to prevent unauthorized deployments. You do not write a system prompt for Jenkins that says "please only deploy reviewed code." You put branch protection rules that the pipeline cannot bypass. You require N approvals from humans the pipeline cannot impersonate. You use signed commits that the pipeline cannot forge.

That is the level of control we need for AI orchestrators. Not "please behave." Instead: "the infrastructure makes misbehavior physically impossible."

The specific mechanisms will vary by deployment context. Scenario (1) uses Linux user isolation, rootless Podman pods, root-owned policy integrity controls, bearer-token adapter auth, and append-only audit paths. Scenario (2+) can add Sigstore/cosign verification, mTLS, and hash-chained audit. Your system might use different primitives. The principle is constant: governance through infrastructure, not through instructions.

---

## Conclusion

The question is not whether your AI orchestrator will exceed its intended authority. It will. Not necessarily through malice -- more likely through a reasoning error, an edge case in its decision logic, a misinterpretation of an ambiguous instruction. It will be fine for a thousand runs and then, at 3am on a Saturday, it will make a provisioning decision you did not intend.

The question is whether your architecture contains that mistake before it matters.

If your orchestrator can modify its own policy, you have a governance gap. If your orchestrator is the sole authority that verifies the policies it constructs, you have a governance gap. If your orchestrator can delete or modify audit entries that record its own actions, you have a governance gap. If your orchestrator can provision agents at unlimited velocity without human checkpoints, you have a governance gap.

Close the gaps with architecture. Separation of duties enforced by independent integrity gates. Immutable baselines enforced by ownership and permission requirements. Audit integrity enforced by append-only stores. Provisioning velocity enforced by rate limits and circuit breakers. Human authority enforced by graceful degradation modes that freeze rather than improvise when the human is unavailable.

Behavioral alignment is a feature. Architectural governance is a requirement.

---

## Series Navigation

- Article 1: [The Two Patterns of Agent Isolation (and Why You Need a Third)](/blog/2026-02-27-isolation-patterns/)
- Article 2: [Seven Users, Seven Vaults: Defense-in-Depth for Multi-Agent Systems](/blog/2026-02-27-defense-in-depth/)
- Article 3: [The Communication Problem: Why Multi-Agent Messaging is Harder Than You Think](/blog/2026-02-27-communication-problem/)
- **Article 4: Governance by Architecture: How to Stop Your Orchestrator From Going Rogue** (you are here)
