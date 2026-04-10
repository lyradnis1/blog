---
title: "AI Sandboxing & Agentic Workflows"
date: "2026-04-10"
tag: security
topic: AI Security · Docker · Agent Design
---

## The core tension

Sandboxing AI means restricting what it can touch — file systems, networks, processes, credentials. But agentic AI *needs* to act on the world: call APIs, write files, run code, chain tool calls. These two goals are in direct conflict, which is why most sandbox approaches either lock down too much (useless agents) or too little (security theater).

The real question isn't *"should I sandbox?"* but *"what is the trust boundary, and what crosses it?"*

---

## Why Docker is a good starting point

Docker gives you OS-level process isolation at low operational cost. For AI agents, the key properties are:

| Property | What it buys you |
|---|---|
| Filesystem isolation | Agent can't read/write host machine files |
| Network namespacing | You control exactly which endpoints are reachable |
| Ephemeral containers | Destroy and recreate cleanly between sessions |
| Resource limits | CPU/memory caps prevent runaway loops |
| No root on host | Privilege escalation is significantly harder |

A typical setup: the agent runs inside a container that only has access to a scratch volume and a small allowlist of outbound HTTP endpoints. The host machine (with credentials, databases, production APIs) is never directly reachable.

---

## The "agentic escape hatch" problem

Here's where it gets interesting. If your agent needs to:

- Call your production database
- Send emails on your behalf
- Push code to a GitHub repo
- Invoke paid external APIs

...then *something* has to cross the sandbox boundary. The sandbox doesn't eliminate this — it just forces you to make the crossing explicit.

**The pattern that works:** a thin, auditable broker process on the host that the containerized agent communicates with over a local socket or HTTP. The broker:

1. Validates every request against a policy file
2. Logs all calls with full context
3. Enforces rate limits and scope
4. Never exposes raw credentials to the container

The agent only ever sees: "call this internal endpoint" — not "here is my AWS key."

---

## What Docker doesn't solve

- **Prompt injection**: If an agent reads from the web or user files, malicious content inside that data can hijack its behavior regardless of containerization. The sandbox stops OS-level attacks, not semantic ones.
- **Supply chain**: If the agent's dependencies are compromised, the container image itself is the attack surface.
- **Exfiltration via allowed channels**: If outbound HTTP is permitted, a compromised agent can exfiltrate data through it.

---

## Better sandboxing primitives to explore

- **gVisor** (Google): kernel-level syscall interception, harder to escape than standard Docker
- **Firecracker** (AWS Lambda): microVM isolation with near-container startup time — what Lambda uses under the hood
- **Seccomp profiles**: limit the actual Linux syscalls available inside the container
- **WasmEdge / WASI**: run the agent logic in WebAssembly — no OS access at all unless explicitly granted

---

## My current thinking

Docker is the right first step — not because it's the most secure, but because it forces you to *draw the trust boundary* concretely. You can't hand-wave "the agent just runs" once you're writing `--network`, volume mounts, and env vars explicitly.

The broker pattern is the key insight: the sandbox isn't a wall that prevents action, it's a checkpoint that makes action auditable.

**Next to explore:** How MCP (Model Context Protocol) formalizes this broker pattern — tools as declared capabilities with explicit scopes, rather than arbitrary subprocess spawning.

---

*Entry 001 — AI Security series*
