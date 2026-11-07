# 0001 — OpenWiki for agents

**Date:** 2026-10-22
**Status:** Accepted
**Ticket:** CTO-3043 (epic CTO-2655, initiative CTO-2897)

## Context

Agents working in a repo, whether in a local session, a CI workflow or the ADLC pipeline, know the rules it follows from its harness docs, but not how the code is actually built. They rediscover it every time from the diff and whatever files they open. Changes whose impact sits outside the diff, such as a consumer in another module, are easy to miss, and the exploration costs tokens every time.

OpenWiki is an open-source CLI that generates a Markdown wiki of a repo as built and updates it from git diffs.

## Decision

### 1. We will use OpenWiki to give agents a map of how each repo is built

The purpose of the wiki is to save agents from rediscovering the system on every run, and to show them what a change affects beyond the diff. It covers what the code alone doesn't make obvious: how modules connect, the contracts between them (events, APIs, shared data), invariants, how things fail, and where to start for a given area.

It does not repeat what an agent can read directly from the code, such as directory listings, file-by-file summaries or copied code. It also does not state rules or conventions; those belong in harness docs. Keeping it to this narrow purpose is what makes generated context help rather than hurt.

### 2. We will keep harness docs and OpenWiki docs separate

Harness docs (the `.agents/adlc/` contract, `docs/`, the `CLAUDE.md` / `AGENTS.md` index) are prescriptive: they state how the team wants work done. People write them, and they change through normal PRs.

OpenWiki docs (`openwiki/`) are descriptive: they state how the code is built today. Only the OpenWiki workflow writes them. Outside `openwiki/`, it changes only its own marked block in `AGENTS.md` and `CLAUDE.md` (`<!-- OPENWIKI:START -->` to `<!-- OPENWIKI:END -->`), which points agents to the wiki.

Each repo's `AGENTS.md` / `CLAUDE.md` also tells agents, outside that block, that `openwiki/` is generated: read it, never edit it, and let the next run pick up code changes. Every agent loads these files at the start of a session, so the rule reaches local agents and pipeline agents alike.

The two overlap only in `ARCHITECTURE.md`. It keeps the intended design that people decide: domain ownership, boundaries, forbidden dependencies, constraints and links to ADRs. The as-built map (modules, data flow, dependencies) moves to `openwiki/architecture/`, and `ARCHITECTURE.md` links to it. If the wiki shows something `ARCHITECTURE.md` forbids, the code has drifted from the design. Conventions, ADRs, troubleshooting and contributing docs stay as they are, since the code can't tell OpenWiki any of that.

Each repo's `openwiki/INSTRUCTIONS.md`, a brief that people write and OpenWiki reads but never rewrites, tells it not to restate the written docs and to link to them instead.

Agents read both, and harness docs win when they disagree. Keeping the two apart means generated docs never overwrite a team's rules, and agents never edit generated docs.

### 3. We will refresh each wiki on a schedule, with one shared workflow

Each repo calls one versioned, reusable GitHub workflow on a schedule. It lives in `YotpoLtd/ci-utils` (`.github/workflows/openwiki.yaml`), next to our other shared agent workflows. Runs start at 03:00 UTC, before the workday in most of the countries we work in, on a random minute so repos don't all hit the shared Bedrock rate limit at once. How often a repo runs (daily to monthly) follows how often it merges. A repo can change it or trigger a run by hand. Each run picks up the code merged to the default branch since the previous run, updates only the pages that change affects, and puts the result in a single rolling update PR. If nothing relevant changed, no PR is opened.

### 4. We will give each repo model access through its existing AWS role, with no stored keys

OpenWiki needs an LLM to write the wiki. The workflow uses a model on Amazon Bedrock and authenticates through GitHub OIDC, which exchanges the run's identity for temporary AWS credentials. It uses the repo's own role (`gh-<repo>`) from `github-actions-permissions`, through its shared `aws_credentials` action, the same way our other agent workflows in ci-utils do. That role already allows calling Bedrock models, so registered repos need no AWS changes to adopt OpenWiki.

## Consequences

- Agents start with a current map of each repo, so they spend less time and fewer tokens working out how it's built.
- The wiki lives in its own folder (`openwiki/`), apart from the docs people write. Only the OpenWiki workflow writes there, and it never edits anything outside that folder. Generated content can't overwrite what people wrote, and no other agent changes the wiki.
- The wiki could make agents worse, as generated context sometimes does. We keep it narrow (decision 1) and use evals to compare agents with and without it.
- Repos already registered in `github-actions-permissions` adopt OpenWiki with no AWS changes, and Bedrock spend stays traceable per repo. A repo that isn't registered yet has to run the registration scaffolder once first.
- Bedrock costs can be high on large repos and monorepos. Writing the wiki means the model reads the code, so the cost grows with the size of the repo. The first run reads all of it; later runs read only what changed since the last one.

## Alternatives considered

- **No wiki.** Nothing to build or run, but agents keep spending tokens exploring the repo and keep missing impact outside the diff. This is the baseline the evals compare against.
- **One central generator for all repos.** Faster to roll out, but the wiki lives outside the repo, so owners don't see its changes and local agents can't read it.
- **Run OpenWiki as an Overcut workflow.** Overcut already runs our agent workflows, but only repos set up in Overcut could adopt OpenWiki. A GitHub workflow works in any repo, and OpenWiki supports GitHub Actions out of the box.
- **One shared AWS role for all repos.** No repo would need registering, but it adds a second way for workflows to reach AWS, gives Bedrock access to every repo, including ones that never had it, and mixes all repos' spend under one role. Limiting it to the OpenWiki workflow alone would mean changing the org's OIDC settings, which every existing role depends on.

## References

- OpenWiki: https://github.com/langchain-ai/openwiki
