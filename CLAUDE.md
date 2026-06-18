# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Project Context

This is a planning and documentation repository for an AI-assisted Kubernetes/OpenShift operations initiative (a.k.a. "AI Cluster Tooling"). The goal is to explore how LLMs can reduce Day 2 toil for platform/devops engineers — tasks like diagnosing pod failures, uploading images, and GitOps deployments.

No application code lives here — the repo contains epics, RFCs, and deployment configuration.

# Repository Structure

- `epic.md` — top-level initiative scope, key tasks, and acceptance criteria
- `rfc-ai-cluster-tooling.md` — the main RFC comparing K8sGPT, kubectl-ai, OpenShift Lightspeed, and kagent; includes the proposed Collector + Diagnostician (Primary + Secondary) agent architecture for kagent
- `helm/sf-values.yaml` — Helm override values for deploying kagent on a resource-constrained OpenShift single-node cluster (Groq Cloud as the LLM provider via OpenAI-compatible API)
- `template_rfc.md` — RFC template to follow when creating new RFCs

# Key Decisions Recorded
  - **Diagnostician (orchestrator):** the agent the DE talks to. *No cluster access at all* — pure reasoning. Calls the Collector for the initial bundle, correlates evidence → root cause → suggested fix, calls the Collector again for follow-ups. A genuine `gap` (data unobtainable by anyone) → it lowers confidence, never re-requests. Returns the finding directly to the DE.
  - **Collector (evidence tool):** the ONLY agent with cluster read tools + a ServiceAccount. On a request from the Diagnostician, gathers the evidence bundle + blast radius (upstream dependencies + downstream dependents) and returns it. **Never concludes.**
  - **Evidence bundle (Collector → Diagnostician):** fixed-schema JSON carrying *evidence + a `source` on every item* (provenance = citable diagnosis), never conclusions. NOT versioned (parked).
  - **Read-only:** the agent suggests; the DE verifies the cited evidence and applies the fix.
- **Phasing philosophy — make it work → make it right → make it good.** Currently in **make-it-work**.

# Problem Statement

The RFC is scoped to **cluster troubleshooting when pods go down**. Other ops (file transfers, Quay uploads, GitOps deploys) are out of scope for now.

The three problems being solved:
1. **Impact + error correlation** — when a pod fails, a DE must manually trawl the cluster to understand blast radius and piece together related error messages. The agent should do this in parallel and surface a consolidated picture.
2. **True root cause, not hotfixes** — DEs often apply a hotfix (e.g. restart the pod) without identifying why the error appeared in the first place. The agent should dig past the symptom to the underlying cause.
3. **Knowledge levelling** — the DE on call changes daily. An inexperienced DE may miss the true root cause entirely. The agent provides a consistent, senior-level diagnostic regardless of who is on shift.


# RFC Structure & Flow

The RFC must read as **one funnel** for a supervisor reading top-to-bottom — each section answers exactly one question and hands off to the next, with no backtracking. Current section order (keep it):

1. **Motivation** — the problem (status ambiguity + the three DE problems). Lead here.
2. **Scope** — in / out.
3. **Tool landscape** — the four-tool comparison → why kagent. State "only kagent can act" **once, here.**
4. **Proposed approaches** — the plan (full-focus-on-kagent OR two-phase). High-level; link down into the design, don't inline it.
5. **Kagent design** — everything kagent in one place: Workflow → Architecture → Success criteria → Key concerns & guardrails → Limitations.
6. **Open questions** — what's still open.

Flow rules:
- **Group all kagent-specific content under "Kagent design"** — a supervisor should never hold a kagent question in their head waiting for an answer later.
- **Workflow first inside the design** — the DE's user journey (the Mermaid sequence diagram) before the detailed tool lists.

# Rules

- Always ask clarifying questions before starting a complex task
- Show your plan and steps before executing
- Cite sources when doing research
- Write the RFC in third person POV only
- When writing the RFC, MAKE SURE information are mentioned not more than once. DO NOT REPEAT INFORMATION.
- Honour the phasing: make it work → make it right → make it good
- When writing teh RFC, don't be long winded. Go straight to the point and be concise about whatever you write.
