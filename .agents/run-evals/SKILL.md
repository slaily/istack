---
name: run-evals
description: Use when asked to run an eval suite of Overcut workflows ("run the <name> eval suite", "run evals/suites/<name>.md", "run cases 1-5 of <suite>"). Runs the suite's Setup, Run and Teardown, grades the assertions, and writes results to evals-workspace/. To learn what evals are, or how to set up and write a suite, read docs/evals.md in YotpoLtd/overcut-handbook (the overcut-golden-path skill covers it).
---

The format of a suite and a case, and how to write them, is in
[`docs/evals.md`](https://github.com/YotpoLtd/overcut-handbook/blob/main/docs/evals.md). If the
current repo has no `evals/suites/` folder, do not improvise one: point the user to "Before you
start" in that page and stop.

## Procedure

1. Read the request: `<suite> [--cases 1,3-5] [--runs N]`. A case number is the id prefix of a case
   file (`0001`). By default, select all the cases of the suite and use its run count. If the
   request has no suite, list `evals/suites/*.md`. If there is one suite, use it and tell the user its
   `description`. If there are two or more, ask the user with `AskUserQuestion`. Show the
   `description` of each suite as the description of its option.
2. Set N to the highest existing `evals-workspace/<suite>/iteration-N` plus one. Use this N for all
   the runs of the request.
3. Do the `Setup` of the suite. Use `overcut:overcut-api` for the Overcut calls.
4. For each selected case and run, do the `Arrange` and the `Act` of the case. Do one run at a time.
   Number the runs from 1. The `Act` says when a run is finished. Wait until then. Then do steps 5
   and 6 for this run.
5. Do the `Assert` of the case: grade each assertion from the live output of the run. Record
   `passed` and `evidence`.
6. Save the results of the run. See Output for the files and their formats.
7. After the last run, show the report to the user. See Report. Then ask the user with
   `AskUserQuestion` if you can do the `Teardown`. Do it only if the user agrees.

## Output

Save these files in `evals-workspace/<suite>/iteration-N/<case>-r<run>/`. This folder is local and
gitignored.

- `grading.json`: the verdict of each assertion.
- `metrics.json`: the time and the tokens of the workflow runs.
- `summary.md`: for a person who investigates the run.

### grading.json

Use the name of the case file as the key. Write one entry for each assertion.

```json
{
  "<case file name>": [
    { "id": <number>, "description": "<first sentence of the assertion>", "passed": true|false, "evidence": "<what you found>" }
  ]
}
```

- `id`: the number of the assertion in the case.
- `description`: the first sentence of the assertion.
- `passed`: `true` or `false`.
- `evidence`: what you found.

### metrics.json

Get the values of each workflow run of the instance with this query:

```graphql
workflowRun(where:{id}) {
  duration
  tokenUsage { inputTokens outputTokens cachedInputTokens cacheCreationTokens callCount }
}
```

```json
{
  "total_tokens": <number>,
  "total_duration_seconds": <number>,
  "workflows": {
    "<workflow name>": { "duration_seconds": <number>, "input": <number>, "output": <number>, "cache_read": <number>, "cache_write": <number>, "llm_calls": <number> }
  }
}
```

- `total_tokens`: the sum of `input` and `output` of all the workflow runs.
- `total_duration_seconds`: the sum of the `duration_seconds` of all the workflow runs.
- `duration_seconds` of a workflow: `duration`, from milliseconds to whole seconds.
- `input`: `inputTokens`. It includes `cache_read` and `cache_write`.
- `output`: `outputTokens`.
- `cache_read`: `cachedInputTokens`.
- `cache_write`: `cacheCreationTokens`.
- `llm_calls`: `callCount`.

### summary.md

```markdown
# <case file name>, <run>

Links: <instance id>, <what the run produced, such as a PR url>
Base: <commit of the default branch>
Steps:
1. <workflow name>, <run id>: <what the step produced>
```

### Report

Show this report in the terminal after the last run:

```markdown
<suite>, iteration <N>: <passed>/<total> assertions passed

| Case | Passed | Duration | Tokens |
|------|--------|----------|--------|
| <case number> | <passed>/<total> | <duration> | <tokens> |

Failed:
- <case number> #<assertion id>: <evidence>

Results: evals-workspace/<suite>/iteration-<N>/
```

- Write one row for each case.
- `Passed`: the assertions that passed, of all the assertions of the case. With more than one
  run, count the assertions of all the runs and add the number of runs: `10/12 (3 runs)`.
- `Duration`: the sum of `total_duration_seconds` of the runs of the case, as `14m 20s`.
- `Tokens`: the sum of `total_tokens` of the runs of the case, as `1.2M` or `850K`.
- `Failed`: one line for each assertion that failed, with its `evidence` in one sentence. With
  more than one run, add the run: `0001 #3, r2: …`. If no assertion failed, do not write `Failed`.
- Take all the values from `grading.json` and `metrics.json`.

## Gotchas

- `AskUserQuestion` needs 2 to 4 options. A question with one option is rejected, so do not ask when
  there is only one choice. State the choice you are taking and continue.
- The Teardown deletes the Overcut copies and cancels the instance, so a person can no longer
  investigate the run in Overcut. What the run produced outside Overcut stays, such as a closed PR
  with its comments, reviews and commits.
  `workflowRun` kept its `duration` and `tokenUsage` after deletion, but its step logs were not
  checked. Save `grading.json`, `metrics.json` and `summary.md` first.
- `consumption { tokenUsage }`, which the `overcut-api` docs show, is `null` for every run. Use
  `workflowRun.tokenUsage`.
- `summary.md` is a copy for people and can be incomplete. `evidence` must come from the source.
- Cancelling an instance in Teardown overwrites its `endedAt`. Take the durations in `metrics.json`
  from each `workflowRun`, or read the instance before cancelling.
- An instance parked at `WaitingForHuman` or `WaitingForDecision` has finished its work. It is the
  expected end for a workflow that needs a person, not a stall, so stop waiting.
- A foreground `sleep` is blocked in Bash. Poll in a background command that writes to a file.
- `duplicateAgent` duplicates inside the same project, so on a library agent the copy lands in the
  library. To copy it into a project, read the agent, `createAgent` there with the same fields,
  then `assignSkillsToAgent`.
