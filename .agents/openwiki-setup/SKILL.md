---
name: openwiki-setup
description: Use when asked to set up or onboard a repo to OpenWiki ("add OpenWiki to this repo", "set up openwiki for yotpo-api", "generate the wiki for this repo").
---

## Procedure

1. Find the repo to onboard:
   - If the user names a repo, use that repo.
   - If the user names no repo, use the current directory if it is a git repo. Confirm it
     with the user.
2. Choose the schedule. Count the PRs merged in the last 30 days:
   `gh pr list --repo <owner>/<repo> --state merged --search "merged:>=<date 30 days ago>" --limit 100 --json number --jq length`
   Ask the user with `AskUserQuestion`. Give four options, each at 03:MM UTC. This time is
   outside work hours for most teams:
   - Daily: `MM 3 * * *`
   - Twice a week: `MM 3 * * 1,4`
   - Weekly: `MM 3 * * 1`
   - Monthly: `MM 3 1 * *`

   For MM, pick a random minute from 1 to 59. Do not use 0, 15, 30 or 45. GitHub delays
   scheduled runs at busy minutes, and all repos share one Bedrock rate limit.

   Recommend the option that runs about as often as the code changes:
   - If there are 20 or more merges, recommend Daily.
   - If there are 8 to 19 merges, recommend Twice a week.
   - If there are 4 to 7 merges, recommend Weekly.
   - If there are fewer than 4 merges, recommend Monthly.
3. Add the OpenWiki workflow at `.github/workflows/openwiki.yml`:
   - Read the example:
     `gh api repos/YotpoLtd/ci-utils/contents/docs/workflows/openwiki.md -H "Accept: application/vnd.github.raw"`
   - Copy the YAML of the example. Keep its `openwiki.v<N>` tag.
   - Replace the cron line with the schedule from step 2.
4. Write `openwiki/INSTRUCTIONS.md`. OpenWiki reads this file on each run.
   Write fewer than 500 words. Use this template:

   ```markdown
   # OpenWiki

   ## What to document
   1. <area> — <what the code does not show>
   2. <area> — <...>

   ## Out of scope
   - <topic> — <reason>

   ## Rules
   - Link to `AGENTS.md`, `CLAUDE.md` and `docs/` for conventions. Do not restate them.
   - Never include customer data or personal data.
   - Name the entry files of each area. Do not copy code.
   ```

   - Put the most important area first. Write one line for each area. Say what the code
     does not show.
   - Write topics in `## Out of scope`, not paths. Put the paths in `.openwikiignore`.
   - Do not name modules or terms. They change. OpenWiki finds them in the code.
   - Do not repeat what OpenWiki already does. OpenWiki does not read secrets or `.env`
     files.
5. Write `.openwikiignore` at the repo root. Add the paths that are generated, vendored
   or large data. Give a reason for each rule. OpenWiki never reads an ignored path.
   Write the rules to a temporary file. Check them with
   `git ls-files -ci --exclude-from=<temporary file>`.
6. If `.claude/protected-files.json` exists, add `openwiki/` to its `auto-safe` tier.
7. Hand off to the user. The wiki does not run until the user does the next steps. End
   with these two parts, in this order:
   - **What I added:** one line per changed file, saying what it is for.
   - **Your next steps:** open with "OpenWiki won't run until you do these." Then give a
     numbered list. Each step says where to do it, as a link or a settings path:
     1. Ask the user how to do the first run. Use `AskUserQuestion`. Give two options:
        - "Run locally now (Recommended)": the run uses the Claude subscription of the
          user.
        - "Wait for CI": the next scheduled run generates the wiki.

        If the user waits for CI, offer to commit the setup changes, push the branch and
        open a pull request.
        If the user runs it locally, install OpenWiki and connect it to Claude Code:
        `npm install -g openwiki && openwiki integrations install claude`
        Then give the user these steps:
        1. Pull the latest changes. Stay on the setup branch. OpenWiki reads
           `INSTRUCTIONS.md` and `.openwikiignore` on the run.
        2. Restart Claude Code.
        3. Ask: "Initialize this repository's OpenWiki from the current source and tests."
        4. Commit `openwiki/` with the setup changes, and merge them in one PR.
     2. Give the repo its AWS role for Bedrock. In the Roadie self-service job, enter the
        repo name and product line:
        https://github.com/YotpoLtd/github-actions-permissions#roadie-self-service
     3. In Settings → General, enable "Allow auto-merge", squash merging and
        "Automatically delete head branches".
     4. If the default branch requires reviews, add a bypass for `SystemYotpo` on
        `openwiki-update` in Settings → Rules → Rulesets.
     5. Review the first wiki PR. Check that it is correct, has no secrets, and describes
        only the code. If pages are wrong or noisy, run this skill again to tune
        `INSTRUCTIONS.md` and `.openwikiignore`.

## Gotchas

- The ci-utils workflow has its own `concurrency:` group. A second group in the repo's
  workflow can block the run. Do not add one.
- A local first run must start from the latest default branch. OpenWiki saves this
  commit in `openwiki/.last-update.json`. Scheduled updates compare against it, so it
  must stay on the default branch after a squash merge.
- The workflow writes the `openwiki/` pages. It also writes the `<!-- OPENWIKI:START -->`
  block in `AGENTS.md` and `CLAUDE.md`, and creates `AGENTS.md` if it is missing. Put the
  index line outside this block.
- `.openwikiignore` stops OpenWiki from reading a path.
- Only tracked files go into a run, because CI uses a clean checkout. Do not copy
  `.gitignore` rules into `.openwikiignore`.
- The next run overwrites edited wiki pages. OpenWiki reads `INSTRUCTIONS.md` and
  `.openwikiignore` on each run and never rewrites them. To change the wiki, change these
  two files.
