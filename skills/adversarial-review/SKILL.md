---
name: adversarial-review
description:
  "Run an in-session adversarial review of Shannon work using a fresh-context
  reviewer. Use for experiment design gates, experiment result gates, risky Rust
  changes, Nushell/Reedline subtree upgrades, and workflow compliance."
---

# Adversarial Review

Use this skill when work needs a skeptical, independent review before it is
trusted. The reviewer should try to reject the design or result on evidence, not
act as a rubber stamp.

## When To Use

- Before implementing an experiment design.
- After recording an experiment result, before the result commit.
- For Nushell or Reedline subtree upgrades.
- For changes touching command dispatch, shell state synchronization, process
  handling, signal handling, config loading, or terminal behavior.
- When the user asks for an adversarial, skeptical, or red-team review.

## Reviewer Contract

The reviewer is read-only. It may inspect files, diffs, logs, and test output.
It must not edit files, stage, commit, push, or run commands that mutate the
working tree.

The reviewer holds the work to:

- `AGENTS.md`
- the relevant `issues/{NNNN}-{slug}/README.md`
- the relevant `NN-{experiment}.md` file
- the stated user request

## What To Check

- **Correctness:** logic errors, bad state transitions, wrong env propagation,
  panics on user input, shell quoting bugs, race conditions, signal mishandling,
  and regressions in nu/bash mode behavior.
- **Upstream fidelity:** for Nushell/Reedline upgrades, verify Shannon's fork
  surface was intentionally preserved and upstream code was not accidentally
  half-merged.
- **Scope:** the experiment does exactly what it claims, with no unrelated edits
  or hidden refactors.
- **Verification:** tests or manual checks actually prove the result and have
  concrete pass/fail criteria.
- **Workflow:** experiment file exists, README links it with the correct status,
  design/result review is recorded, and plan/result commits are separate when
  required.
- **Maintainability:** names, comments, and structure make future upstream syncs
  and debugging easier rather than harder.

## Prompt Template

```text
Review this Shannon experiment with fresh context. Do not edit anything.

Focus:
- design review | result review | diff review

Artifacts:
- AGENTS.md
- issue README: issues/{NNNN}-{slug}/README.md
- experiment file: issues/{NNNN}-{slug}/NN-{slug}.md
- relevant diff: {git diff command or files}
- verification output: {commands/output if available}

Try to reject the work on evidence. Return:

VERDICT: APPROVED | CHANGES REQUIRED

Findings, most severe first:
[Required] file:line - problem. Evidence: ... Fix: ...
[Optional] file:line - improvement. Evidence: ... Fix: ...
[Nit] file:line - trivial cleanup. Fix: ...
```

## Verdict Rules

- `APPROVED` only when there are zero Required findings.
- Required findings must cite concrete evidence.
- Optional findings are useful but non-blocking.
- Nits should be rare.
- If nothing is found, state what was checked so the approval is legible.
