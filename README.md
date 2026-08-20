# ia-sistem — AI-augmented engineering as a system

A personal, multi-agent operating layer built on top of an LLM coding agent, where the human is
the **orchestrator** and a set of specialized AI **identities** do scoped work under hard,
fail-closed guardrails. It is not a chatbot wrapper: it is an engineered system with its own IPC,
adversarial review, durable state, and governance-as-code.

> This is an abstract case study. It contains no credentials, hosts, IPs, client names, or
> employer-identifying details, and it does not publish the private control plane.

## The problem

An LLM agent is powerful but stateless, over-eager, and easy to over-trust. Used naively on real
infrastructure it will confidently do the wrong, irreversible thing. I wanted the leverage of an
AI agent **without** giving up the disciplines that keep production safe: review before acting,
least privilege, reversibility, and an honest record of what was done.

So I built a system around the agent that encodes those disciplines as structure, not as good
intentions.

## Architecture

```
                    ┌──────────────────────────────────────────────┐
   human            │  ORCHESTRATOR (me) — decides; agents advise    │
 (orchestrator) ───▶│  approves irreversible actions; routes work    │
                    └───────────────┬──────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
  ┌───────────┐   file-bus    ┌───────────┐   file-bus   ┌───────────┐
  │ identity  │◀─────────────▶│ identity  │◀────────────▶│ identity  │   … (specialized roles:
  │  (build)  │  HMAC-signed  │ (review)  │  atomic      │  (ops)    │     build / adversarial
  └─────┬─────┘  async inbox  └───────────┘  rename+watch └─────┬─────┘     review / infra / …)
        │                                                       │
        ▼                                                       ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │ GUARDRAILS (hooks): pre-exec 4-checks · scrub-on-write (fail-closed)│
  │ identity-anchor · capability / device-approval / push guards        │
  └──────────────────────────────────────────────────────────────────┘
        │                                                       │
        ▼                                                       ▼
  ┌───────────────────────┐                         ┌───────────────────────┐
  │ KNOWLEDGE SUBSTRATE    │   offline BM25 + MCP    │ DURABILITY             │
  │ markdown vaults + git  │◀───────(gspark)────────▶│ auto-commit+push ·     │
  │ 40+ ADRs · 50+ methods │                         │ evacuate · recovery    │
  └───────────────────────┘                         └───────────────────────┘
```

| Layer | What it is | Status |
|---|---|---|
| Orchestrator + identities | Specialized AI roles (build, adversarial review, infra, docs, research, security) coordinated by a human orchestrator | Operational (daily use) |
| Mesh (IPC) | Async file-bus between identities: per-role inbox dirs, typed frontmatter, HMAC-signed messages, atomic delivery (write-temp → rename), event-driven watcher with fallback | Operational |
| Knowledge substrate | Markdown "brains" under git, each with an entry map; the substrate an LLM reads to work | Operational |
| Retrieval | `gspark` — a dependency-free Go binary: offline BM25 search + an MCP server, so any agent can query the vault locally with no cloud | Operational |
| Guardrails | Hooks that intercept actions: a pre-execution checklist for destructive commands, fail-closed secret-scrubbing on write, identity anchoring, capability/device-approval/push guards | Operational |
| Durability | Auto-commit + push after every edit; an evacuation script and a disaster-recovery drill | Operational |
| Governance | 40+ architecture decision records, 50+ named working methods, a work→method classifier | Operational |
| Freshness pipeline | Triple-gate ingestion (authenticity / maturity-soak / rollback) with "ingest ≠ adopt" | Design + early build |

## Design principles

- **Orchestrator, not autopilot.** At every decision point an identity presents options and a
  recommendation; the human decides. Irreversible actions require explicit, routed approval.
- **Adversarial by construction.** A distinct *review* identity exists to refute proposals before
  they run — a second perspective with a different mandate, not a rubber stamp.
- **Fail-closed guardrails.** Secret-scrubbing, destructive-command checks, and device approval
  block by default; the safe path is the default path.
- **Reversible or gated.** Additive "move, don't delete"; last-known-good pointers; trivial,
  tested rollback before anything risky.
- **Anti-lock-in.** Knowledge is plain markdown; retrieval is a local binary; the LLM is a
  *swappable head*. The system runs offline and is not bound to any single provider.
- **Governance as code.** Decisions live as ADRs and named methods, so the *way* of working is
  itself versioned, reviewed, and reused.

## Three war stories (the actual engineering judgment)

These are the part that matters — where the system caught what a naive agent would have missed.
Details are generalized; the mechanism is the point.

**1. Adversarial ratchet caught a repeat incident.**
Before executing a change on critical infrastructure, the plan went to an adversarial review
identity. It cross-checked the plan against a memory of past incidents and flagged a step that
would have repeated a prior failure — a cascade of false alerts. The plan was changed to a
read-only probe first. The guardrail wasn't an automated test; it was a *second identity with a
different role, a memory of failures, and a mandate to refute.*

**2. Empirical method on a live system.**
Raising the severity of monitoring on critical services, I found the alert rules were
auto-generated and not directly editable. Instead of forcing it, I used the native mechanism
scoped by expression to the exact targets, and verified every probe against known-good data
*before* arming the alert. The pattern that fell out: **recon → probe → verify the real value →
only then arm.**

**3. Declared coverage ≠ real coverage.**
An audit showed effective monitoring coverage was a small fraction of what was assumed. The
lesson became a standing guardrail: **never sign off "100%"** — reframe to *verified critical
coverage* plus an explicit backlog of the rest. Honesty about gaps is a deliverable.

## What it demonstrates

- Designing and operating a **multi-agent system** with real inter-process communication, not a
  prompt chain.
- **AI safety in practice**: adversarial review, fail-closed guards, least privilege, human-in-
  the-loop on irreversible actions.
- **Retrieval / RAG without the cloud** (offline BM25 + MCP) and provider-agnostic design.
- **Systems judgment**: reversibility, durability, disaster recovery, and governance-as-code.

## Honest status

The core (orchestrator, mesh, substrate, retrieval, guardrails, durability, governance) is
operational and in daily use. The knowledge-freshness pipeline is partly built and partly
designed. Some specialized identities are still scaffolds. Nothing here is presented as more
finished than it is.

---

Built by Zuriel Vázquez. Abstract case study — the private control plane, keys, and
environment details are intentionally omitted.
