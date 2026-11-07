---
name: code-review
description: Use when asked to review code, review a diff or PR, check changes before merging, or when the user says "code review" or "review my changes". Reviews across correctness, readability, performance, and architecture; emits structured findings, a score, and a verdict.
---

# Code Review

Multi-axis review across four dimensions: correctness, readability, performance, and architecture.

## Pin the fixed point

Whatever the invoker said is the fixed point (a commit SHA, branch name, tag, `main`, `HEAD~5`, etc.). If none was specified: on a pull request, use the PR's base branch.

Capture the diff once: `git diff <fixed-point>...HEAD` (three-dot, so the comparison is against the merge-base). Also note the commits via `git log <fixed-point>..HEAD --oneline`.

Before going further, confirm the fixed point resolves (`git rev-parse <fixed-point>`) and the diff is non-empty. A bad ref or an empty diff fails here, not mid-review.

## Read .agents/adlc/CODE_REVIEW.md

Repo specific code review rules live in one file in the repo under `.agents/adlc/CODE_REVIEW.md`. Read it **fresh every run** — its rules change. Where it and this skill overlap, it wins: its axes, severities, weights, points, verdict, and output shape all replace the defaults here. Everything below applies as-is when there is no .agents/adlc/CODE_REVIEW.md, and fills any gap the file leaves.

Follow the files `.agents/adlc/CODE_REVIEW.md` references, on the conditions it states, and treat their rules with the same precedence.

Also read these when they exist:

- `AGENTS.md` — repo instructions
- `ARCHITECTURE.md` — module map, boundaries, architectural decisions
- `docs/conventions.md` — patterns and anti-patterns

## Review offloading

How the review splits across sub-agents when the runner supports delegation (single-agent
runners apply the axes themselves and skip this section).

**Default split — one sub-agent per axis, one pass each:**

- Spawn exactly four reviewers: correctness, readability, performance, architecture.
  Each gets the same pinned diff and repo rules, reviews **only its axis**, and returns
  findings in the schema below.
- Reviewers record findings **incrementally as they find them** (append to the shared
  scratchpad or emit per finding) — never as one monolithic final message; the
  coordinator assembles the finding list from what was recorded, so a lost final reply
  loses nothing.
- A reviewer that returns nothing recorded gets **at most one retry**; if the retry also
  records nothing, the coordinator reviews that axis itself rather than spawning more.
- The coordinator assembles; it does not re-review or re-verify an axis that
  returned findings.

**Repo override:** a `## Review offloading` section in `.agents/adlc/CODE_REVIEW.md`
replaces this default; the incremental-recording and single-retry rules still hold
unless it says otherwise.

## The review axes

### 1. Correctness

- Are edge cases handled (null, empty, boundary values)?
- Are error paths handled, not just the happy path?
- Any off-by-one errors, race conditions, or state inconsistencies?

### 2. Readability & Simplicity

- Are names descriptive and consistent with project conventions?
- Is control flow straightforward — guard clauses/early returns for edge cases instead of nested ternaries or deep callback/conditional nesting?
- Could this be done in fewer lines? 1000 lines where 100 suffice is a failure.
- Does a special-case `if` get dropped into a function that's about something else, instead of getting its own function?
- Does the same check appear copy-pasted in multiple files instead of written once?

### 3. Performance

- Any N+1 query patterns, or unbounded/unconstrained data fetching?
- Any synchronous operations that should be async?
- Any missing pagination on list endpoints?
- Any large objects created in hot paths?
- Any unnecessary re-renders in UI components?

### 4. Architecture

- Does it follow existing patterns, or introduce a new one? If new, is it justified?
- Are module boundaries clean and dependencies flowing one way (no cycles)?
- Is there duplication that should be shared?
- Is there a class, interface, or config layer built to handle cases that don't exist
  yet?
- Is feature-specific logic sitting in a shared module instead of its own?
- Separation of concerns — is the code split so each unit (module, class, function, component) has one specific role, or has any of them accumulated unrelated responsibilities?

## Output

A review produces three things: **findings**, a **score**, and a **verdict**. Build them
in that order — the score and verdict are both derived from the final finding list, so
they are computed last, never guessed at up front.

### Findings

Emit findings as structured data — one object per finding, never prose.

```json
{
  "file": "src/orders/OrderService.java",
  "line_start": 142,
  "line_end": 142,
  "severity": "critical",
  "tags": ["correctness"],
  "description": "Unparameterized SQL built by string concatenation; `customerId` reaches the query unescaped.",
  "fix_suggestion": "Bind `customerId` as a query parameter instead of interpolating it, so the driver escapes it."
}
```

- `severity` — `critical`, `major`, or `minor`. Only these three.
  - `critical` — breaks correctness, loses or exposes data, takes the system down, or opens a security hole. Blocks regardless of how clean the rest of the change is.
  - `major` — a real defect or structural problem that must be addressed.
  - `minor` — style, preference, small cleanup.
- `tags` — one or more, lowercase. **Always include the axis that surfaced the finding** — `correctness`, `readability`, `performance`, or `architecture` — since the axis sets the weight. Add a tag from the repo specific file's own category names where it defines any (e.g. a repo's `avro`, `outbox`, `migration`, `translations` categories) — don't invent categories of your own beyond the axis.
- `fix_suggestion` — what the conforming version looks like and why that fixes it. One or two sentences, not a patch: name the change and the reason, and leave the code to the reader.

### Score

A finding tagged with several axes is charged **once, on its heaviest axis** — one problem costs one charge, however many tags it carries.

Weight per axis:

| Axis | Weight |
|---|---|
| `correctness` | 3 |
| `architecture` | 2 |
| `performance` | 2 |
| `readability` | 1 |

Points per severity:

| Severity | Points |
|---|---|
| `critical` | 20 (flat, no axis weight) |
| `major` | 8 |
| `minor` | 2 |

Formula:

score = 100 − (20 × critical_count) − Σ(non-critical: finding.severity_points × max(finding.axis_weights)) [floored at 0]

Report only the total and severity counts, nothing else:

```
Score: 68/100 | CRITICAL: 0, MAJOR: 1, MINOR: 4
```

### Verdict

**Any finding at `critical` is `REQUEST_CHANGES`, whatever the score.** Otherwise read
off the score:

| Score | Verdict |
|---|---|
| 90–100 | `APPROVE` |
| 60–89 | `COMMENT` |
| below 60 | `REQUEST_CHANGES` |

Report only the verdict, nothing else:

```
Verdict: REQUEST_CHANGES
```
