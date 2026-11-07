# 0001. Company knowledge base for agents

- **Status:** Proposed
- **Date:** 2026-11-02
- **Ticket:** CTO-3114 (epic CTO-2921)

## Context

Yotpo's knowledge is scattered, and some of it isn't written down at all. Agents can't search it in one place, so they work with partial context or rebuild knowledge that already exists. 

Agents need one place to find what the company knows, and teams need an easy way to add what is missing.

## Decision

### 1. One monorepo for the brain

Team and company-wide knowledge live in one repo, `yotpo-brain`:

```text
yotpo-brain/
├── teams/
│   └── <team>/
│       ├── README.md           ← who the team is and what it owns
│       └── pages/              ← the team's knowledge, organized as it likes, for example:
│           ├── onboarding/
│           ├── development/
│           └── operations/
└── org/                        ← company-wide knowledge
```

Each team owns its folder.

One repo gives everyone one place to find and add company knowledge, and a new team needs only a folder.

### 2. What goes into the brain

The brain holds knowledge that changes slowly and that the whole company may read. Live data stays in its own system, and agents reach it through that system's MCP server.

Goes into the brain:

- **Code wikis:** how each repo is built and how its flows work.
- **Team knowledge:** runbooks, architecture notes, decisions, specs, design docs and onboarding.
- **Company-wide knowledge:** product lines, domain concepts, cross-team decisions and engineering standards.

Stays out:

- **Live data:** Jira, Coralogix, Salesforce, Gong, Slack and incident.io, reached through their own MCP servers.
- **Sensitive data:** secrets, credentials, customer data and personal data. Every publish is checked for them before indexing.

Live systems are the source of truth for current data and have their own access rules. A copy in the brain would go stale and let every agent read what those rules protect. Knowledge that changes slowly can be reviewed in git and keeps search results focused.

### 3. One managed knowledge base for agents

All published knowledge is indexed by one Amazon Bedrock Managed Knowledge Base, which reads from one S3 bucket. Each repo publishes its OpenWiki to the bucket, and `yotpo-brain` publishes team and company-wide knowledge, always from git:

```text
s3://yotpo-brain/
├── repos/
│   └── <repo>/          ← each repo's OpenWiki (the OpenWiki ADR)
├── teams/
│   └── <team>/          ← from yotpo-brain/teams/<team>/
└── org/                 ← from yotpo-brain/org/
```

We use one knowledge base so that a single search ranks results across all repos and teams. Its size doesn't require a split (Appendix: sizing). We use a managed one because AWS runs the index, embeddings and reranking, and it is the only kind that AgentCore Gateway can expose directly to agents.

### 4. One MCP exposing the knowledge base to agents

Agents search the brain through one MCP, provided by AgentCore Gateway. An agent with access (decision 5) can search the whole brain.

A single MCP lets agents add the brain like any other tool, and is the only way into the knowledge base. Nothing in the brain is restricted, because it holds only what the whole company may read (decision 2), and agents answer better when they can see across teams.

### 5. One IAM user per agent

Each agent gets its own IAM user that can only call the gateway, with an access key. Agents already reach the AWS MCP this way.

A separate user shows which project uses the brain, and lets us revoke one without affecting the others.

## Consequences

- Agents search knowledge from every repo and team in one place, through one MCP.
- Every page comes from git, so every change is tracked and every result links back to its file.
- A team joins the brain by adding a folder.
- Everything published is visible to every agent and to everyone who uses those agents. A secret or customer data that slips past the publish checks is exposed company-wide until it is removed and re-indexed.
- Knowledge that only some teams may see can't go into the brain; it needs its own knowledge base.
- The brain is only as good as what teams write and keep current. Stale or wrong pages can mislead agents, so evals must show that agents answer better with the brain than without it.
- Each agent project holds a long-lived access key that must be rotated and could leak.

## Alternatives considered

- **One repo per team.** Each team keeps its knowledge in its own repo. Teams get full control, but every repo needs its own permissions, its own AWS access to publish to the knowledge base, and its own copy of the checks. Rejected.
- **One knowledge base per team.** Each team gets its own knowledge base. It keeps teams apart, but every team means new AWS resources to create and connect, and the gateway would have to route each question to the right one. Too complex for now. If retrieval quality drops as the brain grows, we can revisit it.
- **Our own vector store and MCP server.** More control, but indexing, search and access at company scale are hard problems. We would have to build, run, scale and maintain all of it ourselves, instead of using what AWS already runs. Rejected.
- **Team-scoped visibility.** Each agent sees only its team's knowledge. The brain holds only company-wide knowledge, so this would mean building identity and filtering for a need we don't have. Rejected.
- **Short-lived Okta tokens.** Agents log in with tokens instead of access keys. More secure, but every agent platform needs Okta setup and must refresh the tokens. Not now: the IAM users work today, and tokens can replace them later without changing the MCP.

## References

- The OpenWiki ADR, OpenWiki for agents (ADR 0001 in yotpo-ai-marketplace): https://github.com/YotpoLtd/yotpo-ai-marketplace/blob/main/plugins/yotpo-common/docs/adr/0001-openwiki-for-agents.md
- Amazon Bedrock Knowledge Bases, connecting through AgentCore Gateway: https://docs.aws.amazon.com/bedrock/latest/userguide/kb-gateway-target.html

## Appendix: sizing

Measured in the pilot (3 repos) and projected to 100 repos with about 50 pages each, plus 50 teams with about 30 docs each. Team docs are assumed to be the same size as wiki pages, which is likely high.

| | Pilot (measured) | 100 repos + 50 teams (projected) | Knowledge base limit |
|---|---|---|---|
| Pages | 87 | ~6,500 (5,000 wiki + 1,500 team) | — |
| Total size | 2.4 MB | ~180 MB | 10 TB |
| Largest page | 57 KB | — | 50 MB per file |

Size is not a problem, the projection uses a tiny fraction of the limit. What we don't know yet is search quality, because many wikis have similar pages. We will learn it by using the brain when agents start searching it, we will see whether they find the right pages.
