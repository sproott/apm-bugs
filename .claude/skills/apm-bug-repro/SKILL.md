---
name: apm-bug-repro
description: End-to-end workflow for reproducing and reporting an APM CLI bug in this repo — dive into the @source/apm submodule to find the root cause, build a minimal runnable reproduction directory, verify it, and write the issue + README entry. Use when adding a new bug to apm-bugs, reproducing suspected APM CLI misbehavior, or turning an observation into a filed issue.
---

# Authoring an APM Bug Reproduction

The full loop for adding a bug to this repo. See the `repo-orientation`
instructions for where the APM source lives and the repo layout; use the
`issue-authoring` skill for the write-up step (6).

## 1. Reproduce the raw symptom

Build the smallest APM project that shows the misbehavior (a root `apm.yml`, and
nested `package/` etc. as needed). Run the real CLI and capture exact output.
Note the observed `apm --version`.

## 2. Find the root cause in source

Read the APM source at `@source/apm/src/apm_cli/` — never infer behavior from
memory. Trace the exact functions involved:

- Follow the command entry point (`commands/<cmd>.py`) to the helpers it calls.
- Distinguish **detection** from **resolution**: inspect the consumer
  `apm.lock.yaml` to see what was actually installed vs what a command merely
  _reports_. A phantom entry in `apm deps list` that is absent from the lockfile is
  a detection/reporting bug, not a resolution bug.
- Compare divergent code paths when two commands disagree (e.g. `prune` vs
  `deps list` computing "orphaned" differently) — pin the specific function and
  predicate that differs.

Confirm the hypothesis by manipulating the repro (flip a dep type, add/remove a
file) and re-checking the lockfile + command output.

## 3. Decide issue boundaries

- One issue + repro **per root cause**. Same-cause variants → one issue, multiple
  `make` targets. Different cause → separate issue, or a `handoff-<slug>.md` if
  it's a lead you're deferring.

## 4. Build the reproduction directory

- Name the dir after the root cause. Give each distinct scenario its own
  subdirectory (flat if there's only one).
- Check in the `apm.yml` files; a thin top-level `Makefile` dispatches into the
  scenario(s) with `clean` + reproduce targets (`audit`, `prune`, …). `clean`
  regenerates state and never touches checked-in `apm.yml`.
- Run every target and confirm it exits 0 and reproduces before moving on.

## 5. Check for existing / duplicate issues

Before writing it up, search the upstream tracker with `gh search issues`:

```bash
# Searches all states by default. This gh's --state takes only
# {open|closed}; --state all errors, so omit it.
gh search issues --repo microsoft/apm "local transitive dependency orphaned"
gh search issues --repo microsoft/apm "orphaned" --match title
gh search issues --repo microsoft/apm "get_canonical_dependency_string"
```

- Search the **offending symbol / key** (`get_canonical_dependency_string`,
  `_local/sub-package`), not just prose.
- Read the closest hits with `gh issue view <n> --repo microsoft/apm`. Closed
  issues are included — check them (already-reported or already-fixed).
- Genuine dup → don't file; note it. Adjacent issues (same family, different
  root cause) → cite them in the write-up.

## 6. Write it up

Write `issue-<slug>.md` and the README entry — use the **`issue-authoring`**
skill, which carries the canonical template and the write-up conventions (short
title, source links folded into "Describe the bug", surface-level "Expected
behavior", real captured output in "Logs", finding-as-a-whole with no
investigation narrative).

## Red flags

- Writing the issue from `apm deps list` output alone without checking the
  lockfile — you may mislabel a detection bug as a resolution bug.
- Claiming "data loss" for an `apm prune` deletion — it removes only the installed
  copy under `apm_modules/`; the source is intact.
- Folding a second, different root cause into one issue "for convenience."
