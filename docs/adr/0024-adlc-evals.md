# 0024 - How we test and evaluate the ADLC

**Date:** 2026-10-08
**Status:** Accepted
**Context:** the ADLC from [0016](0016-autonomous-pr-review-fix-and-merge.md) reaches teams as Workspace Library templates ([0012](0012-versioning-the-templates-teams-copy.md), [0015](0015-propagating-template-upgrades.md)): the `pr-review` orchestration and its `code-review`, `fix-review-comments` and `merge-eligibility` workflows.
- **Only the skills are tested.** Their evals run in the marketplace on every skill PR. The orchestration and the workflows have no tests at all.
- **Changes are checked by hand.** A person reads the output of a real run, as was done for [0023](0023-decouple-ci-from-overcut.md); [lifecycle.md](../lifecycle.md) says "there is no automated gate".
- **Model changes are guesses.** Nothing shows whether a new model would be cheaper, faster or better for the ADLC.

## Decision

**We will test the ADLC end to end in our own test environment, not in teams repos and PRs. There we catch regressions and experiment with new models and changes to the pipeline's logic, and decide with data.**

- **Tests and evals.** Two kinds of check, one for facts and one for quality.
  - **Tests** are assertions that pass or fail. When the orchestration finishes on a PR, a test asserts that:
    - code review and the fixer left their comments on the PR.
    - the PR has the expected labels, such as `ready-for-merge`, `auto-merge-eligible` or `human-required`.
    - the PR was merged by the expected account.
  - **Evals** are graded by an LLM, for what has no single right answer: is a review finding valid, is a fix correct, is the handoff briefing useful. Each assertion is still pass or fail, never a score. Results vary between runs, so each assertion is reported as how many of its runs it passed.
- **An eval testing framework.** Eval suites and eval cases are written in Markdown, and each follows a fixed contract.
  - An **eval suite** follows **Setup, Run, Teardown**:
    - **Setup** creates the environment: the Overcut orchestration, workflows and agents from the Workspace Library, with any change under test.
    - **Run** runs a list of eval cases.
    - **Teardown** removes the environment.
  - An **eval case** follows **Arrange, Act, Assert**:
    - **Arrange** prepares the input the ADLC starts from: today a branch with changes, later also a Jira ticket.
    - **Act** starts the ADLC on it, for example by opening a PR and triggering the orchestration.
    - **Assert** checks the outcome a person sees.
  - Eval suites reference eval cases instead of copying them, so one eval case can be reused by eval suite that needs it.
  - Both live in `adlc-evals` repo, one file each:
    ```
    evals/
    ├── suites/   # one eval suite per file, e.g. reviewer-opus-5-5.md
    └── cases/    # one eval case per file, e.g. 0001-sql-injection-flagged.md
    ```
- **Real pull requests.** `pr-review` starts only on a PR event, so every check starts with a real PR: `adlc-evals` pushes an eval case's branch to `adlc-testing-repo` and opens the PR there.
- **Two new repos: one runs the checks, the other is what they run against.**
  - `YotpoLtd/adlc-evals` holds the eval suites, the eval cases and the GitHub workflows that run them. It is not onboarded to ADLC, so changing a check never starts the orchestration.
  - `YotpoLtd/adlc-testing-repo` is onboarded to ADLC like a team repo, and holds only sample code.
- **A dedicated Overcut project, `ADLC Evals`.**
  - It is the only project connected to `adlc-testing-repo`, so no team's pipeline ever starts on a test PR.
  - `adlc-evals` may commit there without asking, to install and remove each eval suite's copies. This applies only to `ADLC Evals`.
- **Each eval suite run gets its own isolated Overcut resources.** The eval suite prepares an environment in `ADLC Evals`: copies of the orchestration, workflows and agents from the library, with its changes applied, such as a different model for an agent, or a workflow with a step added, removed or edited. All its eval cases run in that environment, and at the end it is removed, so nothing leaks into the next run. Each environment's PR trigger is scoped to its eval suite's branches, so a PR starts only its own eval suite's orchestration, and eval suites can run in parallel. Apart from the eval suite's changes, the scope is the only difference from the library. Each copy's name carries its eval suite's ID, as described under Naming.
- **Naming.**
  - A name says what makes the thing different, in as few words as stay unambiguous, and never repeats what its context already says. Names are lowercase kebab-case.
  - An eval suite is named after what it tests, in at most three words. One change is named `<subject>-<change>`, such as `reviewer-opus-5-5`. Several changes are named after their shared goal or release, such as `fewer-loops` or `release-v0.7.0`; the changes themselves are listed in the eval suite, not in its name. A name is never reused or renamed once it has run, because results are tracked by it.
  - An eval case is named after the behaviour it asserts: `NNNN-<situation>-<expected-result>`, for example `0001-sql-injection-flagged`. The number is never reused.
  - Overcut copies are named `eval-<suite>-<library name>`. For `reviewer-opus-5-5`:
    - orchestration: `eval-reviewer-opus-5-5-pr-review-v0.6.0`
    - workflows: `eval-reviewer-opus-5-5-code-review-v0.6.0`, and the same for the others
    - agents: `eval-reviewer-opus-5-5-code-reviewer-v0.1.0`, and the same for the others
  - In `ADLC Evals` this replaces the rule that installed copies keep the library name verbatim; the library name stays in the copy's name, so the version under test is still visible.
- **Scheduled runs catch regressions.** The same eval cases run on a fixed schedule, so a regression shows up even when nothing changed on our side, for example after an Overcut release or a model update.
- **Manual runs on demand.** Any eval suite can also be started by hand, by triggering the GitHub workflow in `adlc-evals` or with the `run-evals` skill from the `yotpo-overcut` plugin, without waiting for the schedule.
- **Model comparison.** An eval suite sets the candidate model on temporary agent copies, so an experiment never changes what teams or scheduled runs use. It is compared with the library as it is, run back to back on the same eval cases, which shows whether the candidate is cheaper, faster or better.
- **Changing the model teams use.** The model is set on the Overcut Workspace Library Agents, which every team shares, so a change reaches all teams at once. It is made only after a model comparison favours it, and by the Office of the CTO, who own the library.
- **A PAT for GitHub access.** `adlc-evals` will open PRs in `adlc-testing-repo` and read their comments, labels and CI results. GitHub's built-in workflow token works only in its own repo, so `adlc-evals` will use one fine-grained PAT with access to `adlc-testing-repo` only.
- **Results in Databricks, on an Evals dashboard page.** Test and eval results go to `dev.ai_dlc_metrics`, next to the PR-loop data from [0017](0017-pr-review-loop-ground-truth-in-databricks.md), and to an Evals page on the `AI-DLC Metrics` dashboard. The page shows:
  - **Regressions:** the pass rate of each assertion per eval case over time.
  - **Model comparison:** cost, speed and quality of each model on the same eval cases.
  - **Cost and speed trends:** tokens and duration per run, to spot the pipeline getting slower or more expensive.

## Rejected

- **Keeping the checks in `adlc-testing-repo`.** That repo is onboarded to ADLC, so every change to a check would start the pipeline it is meant to test.
- **Installing once and changing the copies in place for experiments.** An experiment would leak into the next scheduled run.
- **Keeping library names verbatim in `ADLC Evals`.** Parallel eval suites would create resources with the same name, impossible to tell apart in the UI and the logs.
- **Keeping changed orchestrations, workflows and agents as exported JSON in the repo.** Exports are hard to read and review. An eval suite describes the resources and their changes in words, and an agent creates the environment from that with the Overcut skills and API.
- **evalkit to run the checks.** evalkit runs a prompt in a local sandbox, but `pr-review` starts only on a real PR. It stays the tool for skill evals.
- **A GitHub App instead of the PAT, for now.** An App belongs to the organization and never needs rotating, but an org admin has to create and approve it, and the first checks should not wait for that. Revisit if the checks ever gate publishing, since a gate should not depend on one person's account.

## Consequences

- **Every run costs money.** It spends Overcut tokens, workspace slots and CI minutes, and each extra model in a comparison is another full run.
- **`adlc-evals` needs two more credentials, each limited to what it uses.**
  - An Overcut API token that can read runs, and create, commit and remove orchestrations, workflows and agents, in `ADLC Evals` only.
  - A Databricks credential that can write the eval tables in `dev.ai_dlc_metrics`.
- **Evals compete with teams for Overcut capacity.** Workspace slots are shared, so parallel eval suites can delay teams' pipelines. Eval runs cap their parallelism and are scheduled off-hours.
- **`adlc-testing-repo` must keep up with the ADLC contract.** When a library version needs repo changes, the test repo needs them too, or every eval case fails for a reason unrelated to the change under test.