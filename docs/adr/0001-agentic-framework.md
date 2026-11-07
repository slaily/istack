# Agentic Framework

Status: accepted

Date: 2026-07-31

Last updated: 2026-08-07

## Context and Problem Statement

As we start building agents on the AgentCore platform, individual projects require specific and varied AWS resources (e.g., S3, EventBridge) alongside the core platform (Harness, Runtime). Working together with OCTO/DevOps, we need to establish a foundational infrastructure strategy to help development teams independently provision these project-specific resources without bottlenecking developer velocity or compromising the security of the core agent platform.

Two tools can provision this, and neither covers a whole project on its own. The `agentcore` CLI is the better developer experience — `agentcore dev` runs the real entrypoint locally, so developers iterate before anything touches AWS. But the CLI only creates what its `agentcore.json` schema describes, and that schema has no field for a custom IAM statement, no way to provision the VPC and S3 access point a filesystem mount needs, and no declarative handling of secrets. Terraform can create all of it and already runs the rest of our stack — but Terraform only provisions infrastructure. It never runs the agent.

## Decision Drivers

* Need for high developer velocity without waiting for another team to provision the infra.
* Requirement for flexibility to use any AWS service dictated by specific agent logic.
* Constraint to maintain strict guardrails and governance over core AgentCore platform.
* Desire to minimize the continuous engineering and maintenance burden on our joint OCTO/DevOps efforts.
* Developers must keep running and inspecting the agent locally before deploying, whichever tool provisions it.
* Unwillingness to own AWS CDK. When the AgentCore CLI's schema does not cover something, the only way forward is to maintain and extend the CDK stack it generates — a third tool we do not want to take on.

## Considered Options

### Option 1: Centralized Infra

All infrastructure lives in a central `agentic-infra` repo. Teams submit requests, and the OCTO/DevOps team writes the Terraform and provisions the resources.
* **Trade-offs:** Provides maximum security and full visibility of resources, but introduces a massive bottleneck. The wait times will grow huge, and the infra team must constantly gain context on app logic to provision correctly.

### Option 2: Decentralized Infra

Teams own the infrastructure for their AgentCore projects in their own repos, provisioning it themselves rather than filing requests, with the tool that fits the project.
* **Trade-offs:** Grants developers complete autonomy and zero dependencies, but requires teams to know their provisioning tool and increases the risk of cloud waste/misconfiguration if automated guardrails are lacking.

### Option 3: YAML driven Infra

Teams define their infrastructure needs in a simple YAML file. A custom orchestrator built by the OCTO/DevOps team parses the YAML, generates Terraform, and provisions it.
* **Trade-offs:** Offers an excellent developer experience requiring zero infra knowledge, but requires massive engineering effort to build and maintain the custom system. Furthermore, no YAML schema can cleanly cover 100% of custom AWS use cases.

## Decision Outcome

Chosen option: "Option 2: Decentralized Infra", because it is the only option that completely resolves the operational bottleneck (Decision Driver 1) while minimizing the maintenance burden on the OCTO/DevOps team (Decision Driver 4), who are freed from writing wrappers for standard AWS services.

To bridge the gap in infrastructure knowledge, OCTO/DevOps builds and maintains exactly two things:

1. **Reusable infrastructure** — the shared GitHub Actions workflows for deploy, invoke etc., and a Terraform module for AgentCore resources.
2. **A scaffolding Skill** — it generates the entire project for a given use case, not a template to copy. The agent running it has access to the reusable infrastructure above, and to examples in our repos when it needs to work out what to create.

Teams do not clone a template and pick pieces out of it. They describe the use case and the Skill produces the project — AgentCore, Terraform, and workflows already wired together.

Since neither tool fits every project, we do not standardise on one. A project provisions by one of two paths, and the Skill picks the path at bootstrap and records it in the repo:

* **Path A — the `agentcore` CLI provisions.** `agentcore.json` is the source of truth. The CLI synthesizes and deploys the generated CDK stack.
* **Path B — Terraform provisions.** Terraform owns the AgentCore resources and everything around them in one state. No generated CDK is involved.

The Skill chooses **Path B** when the project needs anything the CLI cannot create: custom IAM statements, VPC or networking, S3 access points or filesystem mounts, resources shared with non-agent systems, or all infrastructure in a single state for org policy. It chooses **Path A** for everything else — projects whose resources the schema already covers (env vars, memories, knowledge bases, gateways, evaluators, payments) and whose IAM needs nothing beyond CLI defaults.

Both paths install the `agentcore` CLI, because it does two separate jobs: it provisions, and it gives developers their local loop. Path B replaces only the first job with Terraform. Developers still run `agentcore dev`, validate their config, invoke the agent, and read logs exactly as they would on Path A.

## Consequences

* Good, because teams have maximum flexibility and zero dependencies to provision exactly what they need using the official AWS provider.
* Good, because the OCTO/DevOps team can focus strictly on maintaining and securing the AgentCore modules, ensuring core platform governance.
* Good, because the scaffolding Skill generates a whole working project instead of a starting point to assemble, so setup time drops and every project begins on the reusable infrastructure rather than a copy of it.
* Good, because every project keeps the same local dev loop regardless of how it deploys, so onboarding and debugging are identical across both paths.
* Good, because no project is ever forced to edit generated CDK to get the IAM or networking it needs — that requirement routes it to Path B instead.
* Good, because CLI-first AgentCore features (harnesses, payments, evaluators) are usable on day one via Path A, instead of waiting on Terraform provider releases.
* Bad, because developers are forced to learn and write raw Terraform for standard AWS resources, increasing their cognitive load.
* Bad, because the Skill becomes a critical dependency — when it generates a project wrong, or lags a change to the reusable infrastructure, every new project inherits the problem.
* Bad, because allowing raw AWS provider usage increases the risk of cloud waste or misconfigurations, necessitating a heavy reliance on automated governance and guardrails like Terraform Sentinel and AWS Service Control Policies (SCPs).
* Bad, because we build and test everything twice — the workflows, the Skill's generation logic, and the docs each need a Path A version and a Path B version.
* Bad, because a project that starts on Path A and later needs custom IAM has to move to Path B, and moving means importing already-deployed AgentCore resources into Terraform state rather than just changing a config file.
