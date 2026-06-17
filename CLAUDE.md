# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Project Context

This is a planning and documentation repository for an AI-assisted Kubernetes/OpenShift operations initiative (a.k.a. "AI Cluster Tooling"). The goal is to explore how LLMs can reduce Day 2 toil for platform/devops engineers — tasks like diagnosing pod failures, uploading images, and GitOps deployments.

No application code lives here — the repo contains epics, RFCs, and deployment configuration.

# Repository Structure

- `epic.md` — top-level initiative scope, key tasks, and acceptance criteria
- `rfc-ai-cluster-tooling.md` — the main RFC comparing K8sGPT, kubectl-ai, OpenShift Lightspeed, and kagent; includes the proposed Primary + Secondary agent architecture for kagent
- `sf-values.yaml` — Helm override values for deploying kagent on a resource-constrained OpenShift single-node cluster (Groq Cloud as the LLM provider via OpenAI-compatible API)
- `template_rfc.md` — RFC template to follow when creating new RFCs

# Key Decisions Recorded

- **Proposed tool:** [kagent](https://github.com/kagent-dev/kagent) — the only candidate with autonomous write capability and A2A multi-agent orchestration. Still alpha (v0.x).
- **Two-track approach (8 weeks):**
  - Track 1: kubectl-ai for immediate DE value (~1–2 weeks)
  - Track 2: kagent PoC targeting CrashLoopBackOff diagnosis, read-only (~6–7 weeks)
- **Agent pattern:** Primary (orchestrator/classifier) → Secondary (deep investigator with full MCP tool set). Handoff is a fixed-schema JSON brief.
- **Key concerns to address:** blast radius (RBAC per SA), auditability (OTEL + K8s audit correlation), portability (Kubernetes-native, not OCP-specific), security (least-privilege SA), cost (OSS + internal LLM endpoint).

# Problem Statement

The RFC is scoped to **cluster troubleshooting when pods go down**. Other ops (file transfers, Quay uploads, GitOps deploys) are out of scope for now.

The three problems being solved:
1. **Impact + error correlation** — when a pod fails, a DE must manually trawl the cluster to understand blast radius and piece together related error messages. The agent should do this in parallel and surface a consolidated picture.
2. **True root cause, not hotfixes** — DEs often apply a hotfix (e.g. restart the pod) without identifying why the error appeared in the first place. The agent should dig past the symptom to the underlying cause.
3. **Knowledge levelling** — the DE on call changes daily. An inexperienced DE may miss the true root cause entirely. The agent provides a consistent, senior-level diagnostic regardless of who is on shift.

# RFC Checklist

Questions the epic requires the RFC to answer. Check off as each is addressed.

**Key concerns (from epic)**
- [x] Blast radius — what are the guardrails? Can agent execution be vetted?
- [ ] Security — who can access/invoke the agent? (SA RBAC is covered; DE authentication is not)
- [x] Cost — what does running the agent require?
- [x] Risk — what are the risks and mitigations?
- [x] Auditability — who ran what? How is DE identity translated to agent actions?
- [x] Portability — is the solution tied to OpenShift, or viable multi-cloud?

**Scope questions**
- [x] What DE chore to focus on? → cluster troubleshooting / pod failure diagnosis (decided)
- [x] How can AI help? → answered by tool comparison and proposal
- [ ] What are the most common errors in our cluster? (**prerequisite** to the proposal — without this, the choice of CrashLoopBackOff as the PoC target is assumed, not evidence-based. The profiling exercise from the epic must feed this before the proposal can be finalised.)
- [ ] Does the proposal in the RFC truly solve the three problems (impact correlation, true root cause, knowledge levelling)?

# Rules

- Always ask clarifying questions before starting a complex task
- Show your plan and steps before executing
- Cite sources when doing research
- When writin the RFC, MAKE SURE information are mentioned not more than once. DO NOT REPEAT INFORMATION.
