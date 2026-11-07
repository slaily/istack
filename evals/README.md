# Evals

Eval suites and eval cases that test and evaluate the ADLC, or one of its workflows, against
[`adlc-testing-repo`](https://github.com/YotpoLtd/adlc-testing-repo), as decided in
[ADR 0024](https://github.com/YotpoLtd/overcut-handbook/blob/main/docs/decisions/0024-adlc-evals.md).

```
evals/
├── suites/   # one eval suite per file, e.g. adlc-latest.md
└── cases/    # one eval case per file, e.g. 0001-adlc-flags-and-fixes-read-before-lock.md
```

How to write an eval suite and an eval case, how to write assertions, and how to name them, is in
the handbook: [`docs/evals.md`](https://github.com/YotpoLtd/overcut-handbook/blob/main/docs/evals.md).
Run a suite with the [`run-evals`](https://github.com/YotpoLtd/overcut-handbook/blob/main/skills/run-evals/SKILL.md)
skill, from the `yotpo-overcut` plugin. Results go to `evals-workspace/`, which is gitignored.

## What is specific to the ADLC

- **Setup** creates the copies in the Overcut project `ADLC Evals`, which is connected only to
  `adlc-testing-repo`.
- **Act** opens a PR in `adlc-testing-repo`. The PR triggers the orchestration, or the suite's slash
  command starts a single workflow. The PR title, body and commit message never hint at a planted
  defect.
- **Eval case names** start with what is tested: `adlc` for the whole pipeline, or one workflow such
  as `code-review`.
- **Branches.** Each suite's PR trigger is scoped to its own branches by an exact regular expression
  (`^eval/EVAL-[0-9]{4}-<suite>-r[0-9]+$`), so a suite never matches another suite's branches and
  suites can run in parallel.
