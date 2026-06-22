# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Project Context

This is a planning and documentation repository for a Kubernetes/OpenShift cluster-troubleshooting initiative (a.k.a. "AI Cluster Tooling").

**The core problem:** help platform/devops engineers (DEs) troubleshoot the cluster — diagnose *why applications and nodes aren't behaving as expected* and return a **root cause backed by evidence, plus a suggested fix**.

**Why the RFC exists:** to give the supervisor (the lead platform engineer) what they need to **decide which tool to incorporate into the cluster**. The RFC is a decision aid for that person — not a pitch for any single tool. Every section should move the supervisor closer to that decision.

No application code lives here — the repo contains epics, RFCs, and deployment configuration.

# Repository Structure

- `epic.md` — top-level initiative scope, key tasks, and acceptance criteria
- `rfc-ai-cluster-tooling.md` — the main RFC comparing K8sGPT, kubectl-ai, OpenShift Lightspeed, and kagent. Tool-agnostic: the kagent-specific design is **not** inlined here
- `kagent/kagent.md` — the kagent candidate's full design (Collector + Diagnostician architecture, workflow, success criteria, guardrails, rough agent definitions). Kept separate so the RFC stays an open comparison
- `kagent/CLAUDE.md` — guidance for kagent-specific work (only relevant when the task is about kagent)
- `helm/sf-values.yaml` — Helm override values for deploying kagent on a resource-constrained OpenShift single-node cluster (Groq Cloud as the LLM provider via OpenAI-compatible API)
- `template_rfc.md` — RFC template to follow when creating new RFCs

> **Kagent is one candidate, not the answer.** Do not pull kagent-specific detail into this file or the RFC — it lives under `kagent/`. See [`kagent/CLAUDE.md`](kagent/CLAUDE.md) only when the task is explicitly about kagent.

# Problem Statement

The RFC is scoped to **cluster troubleshooting — diagnosing when applications (pods/workloads) and nodes aren't behaving as expected**, producing a root cause with evidence and a suggested fix. Other ops (file transfers, Quay uploads, GitOps deploys) are out of scope for now.

The three problems being solved:
1. **Impact + error correlation** — when a pod fails, a DE must manually trawl the cluster to understand blast radius and piece together related error messages. The agent should do this in parallel and surface a consolidated picture.
2. **True root cause, not hotfixes** — DEs often apply a hotfix (e.g. restart the pod) without identifying why the error appeared in the first place. The agent should dig past the symptom to the underlying cause.
3. **Knowledge levelling** — the DE on call changes daily. An inexperienced DE may miss the true root cause entirely. The agent provides a consistent, senior-level diagnostic regardless of who is on shift.


# RFC Structure & Flow

The RFC must read as **one funnel that ends in a decision** — the supervisor (lead platform engineer) reads top-to-bottom and comes out able to choose which tool to bring into the cluster. Each section answers exactly one question and hands off to the next, with no backtracking, and must not presuppose the agent approach is the answer. Current section order (keep it):

1. **Motivation** — the problem (status ambiguity + the three DE problems). Lead here.
2. **Scope** — in / out.
3. **Tool landscape** — the four-tool comparison. State each tool's capability (e.g. "only kagent can act") once, here.
4. **Proposed approaches** — the plan (full-focus-on-kagent OR two-phase). High-level; link out to the kagent design, don't inline it.
5. **Open questions** — what's still open.

Flow rules:
- **Keep kagent-specific content OUT of the RFC** — it lives in `kagent/kagent.md`. The RFC stays a tool-agnostic comparison; link out to the kagent design rather than inlining it.

# Rules

**Top priorities (override everything below):**

- **Security is the first filter.** A tool's security posture is evaluated before anything else. **If a tool does not pass a Trivy scan — whether a container image or a binary — it is an absolute no-go and is dropped from consideration, regardless of how good it otherwise looks.** State scan results explicitly when comparing tools.
- **Do not zero in on the agent as the solution.** Keep the comparison genuinely open. The agent (kagent) is one candidate among several; the RFC's job is to help the supervisor *choose*, not to argue a predetermined answer.

**Working rules:**

- Always ask clarifying questions before starting a complex task
- Show your plan and steps before executing
- Cite sources when doing research
- Write the RFC in third person POV only
- When writing the RFC, MAKE SURE information are mentioned not more than once. DO NOT REPEAT INFORMATION.
- Honour the phasing: make it work → make it right → make it good
- When writing the RFC, don't be long winded. Go straight to the point and be concise about whatever you write.
