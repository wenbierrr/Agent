# RFC for AI Tooling for Kubernetes/OpenShift Operations

## Table of contents

- [Motivation](#motivation)
  - [Scope](#scope)
- [Design](#design)
  - [The four tools evaluated](#the-four-tools-evaluated)
  - [Proposal](#proposal)
  - [Kagent: Agent Architecture and Key Concerns](#kagent-agent-architecture-and-key-concerns)
    - [Part 1 — Proposed Architecture (Primary + Secondary Agent pattern)](#part-1--proposed-architecture-primary--secondary-agent-pattern)
    - [Part 2 — Key Agent Concerns](#part-2--key-agent-concerns)
  - [Limitation & concerns](#limitation--concerns)
- [Open questions](#open-questions)

## Motivation

The goal is AI assistance for Kubernetes/OpenShift operations — diagnosing issues, answering "why is this broken", and ideally remediating. This RFC surveys four candidates, then evaluates **kagent** in depth because it is the only option that crosses into autonomous *write* territory — which is both its value and its risk.

### Scope

**In scope:** 

- comparison of K8sGPT, kubectl-ai, OpenShift Lightspeed, and kagent
- evaluation of kagent across blast radius, security, cost, risk, auditability, and portability

**Out of scope:**
- Hosting of the LLM (an inference endpoint is assumed to be provided).
- Which LLM model to use.

## Design

### The four tools evaluated

| Dimension | K8sGPT (CLI) | Kubectl-ai (CLI) | OpenShift Lightspeed | Kagent |
|---|---|---|---|---|
| **Type** | AI Scanner | AI Assistant | AI Assistant | AI Agent (read + write) |
| **How it works** | Deterministic analyzers scan cluster/namespace for issues → parse findings to LLM → LLM explains findings | Natural language → kubectl command. Retains conversation context within a session. | Conversational chat in OCP console | Custom agents run as K8s CRDs; LLM decides actions (can make use of mcp tools as well)|
| **Unique features** | There is an operator but little stars | Context-aware | Context-aware | A2A multi-agent orchestration |
| **Tool integrations (where can it pull data from)** | Kube API | Kube API via kubectl | OCP console, OCP API | Via MCP tools. A generic k8s tool reaches any resource on the Kube API — including CRDs like Istio/Argo — bounded by RBAC.<br><br>Domain MCP tools (istio, argo) add specialised ops (e.g. `istioctl analyze`) beyond plain CRD CRUD |
| **Blast radius** | Nil — doesn't write to cluster | Can generate and execute write kubectl commands | Nil — doesn't write to cluster | Configurable — write-capable; can have guardrail (RBAC scoping per agent) |
| **Auditability** | None | K8s API request will appear in K8s audit log under the user we oc log in as | Tied to OCP user identity; questions asked are not logged | can implement built-in tracing of every agent action (OTEL) |

1. **[K8sGPT](https://github.com/k8sgpt-ai/k8sgpt)** — a diagnostic scanner. Deterministic analyzers find problems; the LLM only explains them. Read-only, zero blast radius, but also no action and no memory.

2. **[Kubectl-ai](https://github.com/GoogleCloudPlatform/kubectl-ai)** — a natural-language-to-kubectl translator running on a human's workstation. It executes under *your* kubeconfig, so writes are possible and land in the audit log under your identity. Powerful but unscoped — it inherits whatever you can do.

3. **OpenShift Lightspeed** — a read-only chat assistant embedded in the OCP console. Convenient and tied to OCP identity, but OpenShift-only, can't act, and doesn't log the questions asked.

4. **[Kagent](https://github.com/kagent-dev/kagent)** — the only true *agent*: it can take actions, not just suggest them. The rest of this RFC focuses here.

   > **Note:** Kagent is a new project (created in 2025) and still alpha (v0.x). Further experimentation is needed to confirm it is fully functional and production-ready for our use cases.

### Proposal

Kagent is not committed as the production solution yet. It is alpha and unproven in the target environment, so an upfront commitment would be premature. The proposal is a **sequential two-track approach** within an **8-week** window — capture immediate value with a mature tool first, then validate the kagent.

**Track 1 — kubectl-ai for immediate value (~ 1-2 weeks)**
- Test and deploy [kubectl-ai](https://github.com/GoogleCloudPlatform/kubectl-ai) in the environment.
- Provides DEs a working natural-language-to-kubectl assistant immediately, with no new platform overhead


**Track 2 — kagent PoC (~ 6-7 weeks)**
- Build a minimal **Primary + Secondary** agent targeting **one** failure type to start — **CrashLoopBackOff only** as POC
- The goal is validation, not coverage: confirm kagent's **A2A orchestration works in the OpenShift environment**, and stress-test the six concerns ([Part 2](#part-2--key-agent-concerns)) against a real deployment.
- Runs **read-only** (diagnosis only) — though whether the PoC should also cover write actions is an [open question](#open-questions).
- Architecture in [Part 1](#part-1--proposed-architecture-primary--secondary-agent-pattern).

**Decision point** — after both tracks, the question is whether kagent's multi-agent model delivers enough over kubectl-ai to justify operating it. Only then is there a decision to invest in kagent as the strategic, write-capable platform.

### Kagent: Agent Architecture and Key Concerns

#### Part 1 — Proposed Architecture (Primary + Secondary Agent pattern)

Two-tier agent design for diagnosing pod failures.

- **Primary** = first responder who arrives at the scene, assesses what kind of incident this is, and briefs the specialist.
- **Secondary** = the specialist who does the actual deep investigation with the full toolkit.

**Primary Agent (Orchestrator)**
- In: pod failure report from DE (pod + namespace). Tools (minimal): `kubectl get pod`, `get events`.
- Classifies failure type (CrashLoopBackOff, ImagePullBackOff, OOMKilled, Pending, CreateContainerConfigError…).
- Out: structured brief --parse to--> Secondary Agent (pod, namespace, failure type, key observation, investigation focus); then synthesises Secondary's findings into final root cause + suggested fix for DE.

**Structured brief (Primary → Secondary)**

A fixed-schema JSON handoff — not free text — so the Secondary Agent receives a focused, machine-parseable starting point rather than raw cluster output:

```json
{
  "pod": "checkout-7d9f8c-abc12",
  "namespace": "payments",
  "failureType": "CrashLoopBackOff",
  "keyObservation": "Container 'checkout' exits with code 1, 14 restarts in 6m",
  "investigationFocus": "Inspect container logs and verify referenced secret/configmap exist"
}
```

| Field | Description |
|---|---|
| `pod` | Failing pod name (from DE report) |
| `namespace` | Pod namespace |
| `failureType` | Classified failure (CrashLoopBackOff, ImagePullBackOff, OOMKilled, Pending, CreateContainerConfigError…) |
| `keyObservation` | The signal that drove the classification (exit code, event reason, restart count) |
| `investigationFocus` | Directs the Secondary Agent to the likely area — narrows its tool use |

**Secondary Agent (Deep Investigator)**
- In: the structured brief. Tools (full set via MCP):
  - Core diagnostics: `kubectl logs`, `describe pod`, `get/describe secret`, `get configmap`, `get events`, `get node`, `get pvc`, `oc get scc`, Quay registry API.
  - **Istio MCP tool** — service-mesh diagnostics beyond plain CRD reads: `istioctl analyze` (mesh config validation), proxy-config / Envoy config dumps, sidecar injection and proxy-sync status.
  - **Argo MCP tool** — GitOps/rollout diagnostics: Argo CD app sync + health status, drift detection, and Argo Rollouts progression/abort state.
- Out: structured root cause finding --parse to --> Primary.

**Why split it**
- Primary stays narrow and fast — only enough context to classify.
- Secondary reasons over a focused brief, not raw cluster output → higher accuracy.
- RBAC scoped per agent; each gets only what it needs.
- New failure patterns = extend Secondary's tool set, Primary untouched.

#### Part 2 — Key Agent Concerns

- **Blast radius / vetting** — Phase model: agents start read-only, gain write only after explicit promotion. RBAC scoped per ServiceAccount. PoC keeps both agents read-only; writes (e.g. pod restart) come in a later phase behind a DE approval gate.
- **Security / access** — RBAC tied to the agent's ServiceAccount; least-privilege, scoped to inspected namespaces. DEs invoke via a defined interface (CLI or API endpoint).
- **Cost** — Kagent is free/OSS. Cost = LLM endpoint (internal/private endpoint for air-gapped, no external spend) + engineering effort to build and maintain agents and MCP tool servers.
- **Risk** — Read-only: LLM hallucination → wrong root cause misleads DE. Write phases: bad LLM decision mutates state. Mitigations: confidence-threshold gate on output, DE approval before any write, K8s audit logging of all agent API calls.
- **Auditability** — OTEL trace per agent action; all Kube API calls logged under the agent SA. Gap: audit log shows the SA, not the triggering DE. Bridge by logging DE identity + OTEL trace ID per invocation → chain: DE → invocation → agent actions → API calls.
- **Portability** — Runs on any Kubernetes; not OpenShift-specific. Redeployable on OKE with no architectural change; the only OCP-specific tool (`oc get scc`) is conditionally excluded off-OpenShift. Strongest candidate for the multi-cloud path.

### Limitation & Concerns

- Kagent is **alpha (v0.x)** — APIs and behaviour may change.
- **No native approval gate** before writes (see blast radius).
- **Split-log auditability** — identity correlation across kagent OTEL + K8s audit is an operational burden we must own.
- Higher **resource and operational cost** than the CLI assistants.

## Open questions

1. Should the PoC stay read-only (diagnosis only), or also cover write actions (e.g. pod restart)? Read-only is lower risk and faster to validate; including writes proves the full remediation loop but pulls the approval-gate and blast-radius work into the PoC.
2. Do we require human-in-the-loop approval before agent writes? If so, build/adopt an approval MCP tool, or restrict agents to read-only and pair with kubectl-ai for vetted writes?
2. Can we automate the kagent-OTEL ↔ K8s-audit correlation into a single queryable trail (e.g. inject session/user id as a trace attribute that also lands in audit annotations)?
3. Target topology: one broadly-scoped agent, or many narrowly-scoped per-namespace/per-domain agents (smaller blast radius, more to manage)?
4. Where do read-only tools (K8sGPT / Lightspeed) sit alongside kagent — feeders/diagnostics vs. the action layer?

