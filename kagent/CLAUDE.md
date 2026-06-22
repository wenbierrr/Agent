# CLAUDE.md — kagent

Guidance for working on the **kagent candidate** specifically. Only relevant when the task is about kagent. The kagent design itself lives in [`kagent.md`](kagent.md).

> **Framing:** these are design notes for **one candidate only — the kagent agent approach**. They describe how that tool *would* work *if chosen*; they are **not** a decision that an agent is the answer. The tool comparison in the parent RFC ([`../rfc-ai-cluster-tooling.md`](../rfc-ai-cluster-tooling.md)), not this design, drives the recommendation. Do **not** let kagent-specific detail leak back into the RFC — keep the comparison open.

# Key design decisions

  - **Diagnostician (orchestrator):** the agent the DE talks to. *No cluster access at all* — pure reasoning. Calls the Collector for the initial bundle, correlates evidence → root cause → suggested fix, calls the Collector again for follow-ups. A genuine `gap` (data unobtainable by anyone) → it lowers confidence, never re-requests. Returns the finding directly to the DE.
  - **Collector (evidence tool):** the ONLY agent with cluster read tools + a ServiceAccount. On a request from the Diagnostician, gathers the evidence bundle + blast radius (upstream dependencies + downstream dependents) and returns it. **Never concludes.**
  - **Evidence bundle (Collector → Diagnostician):** fixed-schema JSON carrying *evidence + a `source` on every item* (provenance = citable diagnosis), never conclusions. NOT versioned (parked).
  - **Read-only:** the agent suggests; the DE verifies the cited evidence and applies the fix.
- **Phasing philosophy — make it work → make it right → make it good.** Currently in **make-it-work**.

# Writing rules for kagent.md

- **Keep all kagent-specific content here in `kagent.md`** — never inline it back into the RFC. A supervisor should never hold a kagent question in their head waiting for an answer later.
- **Workflow first** — the DE's user journey (the Mermaid sequence diagram) before the detailed tool lists.

# Related deployment config

- `../helm/sf-values.yaml` — Helm override values for deploying kagent on a resource-constrained OpenShift single-node cluster (Groq Cloud as the LLM provider via the OpenAI-compatible API).
