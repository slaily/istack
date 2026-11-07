---
id: 0001-adlc-flags-and-fixes-read-before-lock
description: A PR reads a member's balance before locking the account, so concurrent orders fail with HTTP 500. The ADLC should flag it, fix it, and leave the PR for a person to merge.
---

## Arrange

Repository: `YotpoLtd/adlc-testing-repo`.

1. From `master`, create the branch `eval/EVAL-0001-<suite>-r<run>`.
2. In [`EarningService.java`](https://github.com/YotpoLtd/adlc-testing-repo/blob/master/src/main/java/com/yotpo/loyalty/earning/EarningService.java),
   method `recordOrder`, read the balance with `ledger.balanceOf(memberId)` instead of
   `ledger.lockAccount(memberId)`.
3. In the Javadoc of `recordOrder`, remove the sentence "The account is locked before the tier is
   read, so concurrent orders for a member are applied one after another." Keep the other sentences.
4. In [`EarningServiceTest.java`](https://github.com/YotpoLtd/adlc-testing-repo/blob/master/src/test/java/com/yotpo/loyalty/earning/EarningServiceTest.java),
   make the unit tests pass with the new code, as the author of the change would. Change only the
   stubs of `lockAccount`:
   - `earnsAtCurrentTier`: remove the `lockAccount` stub. Stub `ledger.balanceOf(MEMBER_ID)` to
     return `AccountBalance(MEMBER_ID, 900, 900)` first and `AccountBalance(MEMBER_ID, 1_100, 1_100)`
     second.
   - `zeroPoints`: stub `ledger.balanceOf(MEMBER_ID)` instead of `ledger.lockAccount(MEMBER_ID)`, to
     return `AccountBalance(MEMBER_ID, 40, 40)`.
5. Commit with the title of the pull request as the message.
6. Push the branch to `adlc-testing-repo`.

If a method or a test named here does not exist, fail the run and write the reason in `summary.md`.

Pull request:

- Title: `earning: read the balance with balanceOf in recordOrder`
- Body: `recordOrder only needs the member's lifetime points to pick the tier, so read them with balanceOf.`

## Act

1. Open the pull request against `master` from the branch, with the title and body from Arrange.
   Make it ready for review, not a draft.
2. Wait until the `pr-review` orchestration instance of the suite for this pull request has
   finished, or 60 minutes. A timeout fails the run.

## Assert

1. The review leaves a `critical` finding on `EarningService.java`.
2. A review finding says the balance is read before the account is locked. It also says that this
   breaks concurrent changes to the member's points: a lost update, a stale read, an optimistic-lock
   failure, or changes that are not applied one after another. A finding that only says "use
   `lockAccount`" does not count.
3. The finding from assertion 2 names the mechanism, the consequence and the fix. The mechanism is
   the lock order or the persistence context. The consequence is HTTP 500 or lost updates under
   concurrent orders. The fix is `lockAccount`, or locking before reading. This assertion needs
   assertion 2.
4. Each other review finding is a real problem in the diff. A correct remark about the test change
   in the diff counts as real. A finding about code that the diff does not change does not count. A
   request for a change that the repository conventions do not require does not count.
5. The fixer replies to the finding, or leaves a comment, to say what it changed. This assertion
   needs assertion 1.
6. The fixer's change removes the read-before-lock defect and changes nothing else. `recordOrder`
   reads the balance with `lockAccount`, or locks before it reads. Any test change matches. A retry on
   `ObjectOptimisticLockingFailureException` hides the defect and does not count. A test that is
   weakened or removed to make CI pass is a failure.
7. The PR has the labels `ready-for-merge` and `human-required`, and does not have
   `auto-merge-eligible`. The PR changes `review-required` paths, so a person must merge it.
8. The PR is not merged.
