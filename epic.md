# Description

In this new era of Gen AI and agent supporting workflow, there is a need to explore how LLM can aid platform engineer in their day 2 operations or development.

The goal is to explore tooling that can transform and improve the way DE does their day to day operations which may include

- Moving files across bastions
- Uploading image to quay
- Deploying via GitOps
- Checking cluster health
- Debugging cluster issue
- Having a repo to enable agent to help

The intent is to move human up the stack to higher level work like policies, guardrail and design work while AI does the execution work in a safe manner.

The key concerns revolving having agents

- Limited blast radius: What are the guardrails? Are there ways to vet agent execution?
- Security: Who can access the agent?
- Cost: What is the requirements of having the agent?
- Risk: What are the risk?
- Auditability: Who ran what on the agent? How is identity translated?
- Portability: Being locked into OpenShift has it cons as we go multi-cloud. This not mean OpenShift solution is to be avoided but alternative explored

Example solution: [OpenShift lightspeed](https://docs.redhat.com/en/documentation/red_hat_openshift_lightspeed/1.0/html/about/ols-about-openshift-lightspeed)


# Out of scope

- Hosting of LLM (Assuming endpoint provided)
- What LLM to use

# Key Tasks

- Profile day to day chores and cluster and give recommendation for area of improvement (4~ weeks)
  - What chores are there?
  - How can AI help in this area?
  - What should we focus on?
- PoC AI solution and Proposal (7~ weeks)
- Gen AI for Platform Engineer 101 (Presentation) (a week)

# Acceptance

- [ ] Area of improvement highlight and presented to team
- [ ] Gen AI presentation in markdown
- [ ] Proposal of solution

