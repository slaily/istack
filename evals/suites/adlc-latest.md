---
name: adlc-latest
description: The ADLC as the Workspace Library ships it, at the latest versions, with no change.
---

## Setup

In the Overcut project `ADLC Evals`, using the `overcut:overcut-api` skill:

1. Check the project is connected only to `YotpoLtd/adlc-testing-repo` and holds no copies named
   `eval-adlc-latest-…`. Stop if it does: another run of this suite is in progress, or an
   earlier run did not tear down.
2. Install the latest library workflows `code-review`, `fix-review-comments` and
   `merge-eligibility`, each named `eval-adlc-latest-<library name>`, where the library name
   includes the version, e.g. `eval-adlc-latest-code-review-v0.7.1`. Leave the agents as the
   library agents.
3. Install the latest library orchestration `pr-review` as `eval-adlc-latest-<library name>`,
   the library name with its version as in step 2, with every library workflow mapped to its
   copy from step 2.
4. Scope every pull-request trigger of the orchestration and the workflows to head branches matching the regular
   expression `^eval/EVAL-[0-9]{4}-adlc-latest-r[0-9]+$`.
5. Run the context-parameter resolution preview for every copy; the base branch resolves to
   `master`. Stop on any parameter without a value.
6. Commit the copies.

## Run

Run each eval case once, with run number `r1`:

- [`0001-adlc-flags-and-fixes-read-before-lock`](../cases/0001-adlc-flags-and-fixes-read-before-lock.md)

## Teardown

1. Close every pull request this run left open in `adlc-testing-repo`, and delete its branch.
2. Delete every copy named `eval-adlc-latest-…` from `ADLC Evals`.
3. Check no copy named `eval-adlc-latest-…` is left.
