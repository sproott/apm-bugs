---
name: issue-authoring
description: Write or refine a bug issue or feature request and its README entry in the apm-bugs repo — the canonical bug and feature-request templates plus the write-up conventions (short title, surface-level expected behavior, current-state-only). Use when turning a reproduced/understood bug into issue-<slug>.md, writing a feature request as feature-<slug>.md, editing an existing issue, or adding a bug's README entry. Referenced by the apm-bug-repro skill for its write-up step.
---

# Issue & README Authoring

How to write up a bug once it is reproduced and understood. (The reproduce →
root-cause → repro-dir steps are in the `apm-bug-repro` skill.)

## The issue file

Create `issue-<slug>.md` from the canonical template at the repo root
(`issue-template.md`):

```markdown
# [BUG] xxxxx

**Describe the bug**
A clear and concise description of what the bug is.

**To Reproduce**
Steps to reproduce the behavior:
1. Run command '...'
2. With parameters '....'
3. See error

**Expected behavior**
A clear and concise description of what you expected to happen.

**Environment (please complete the following information):**
 - OS: [e.g. macOS, Linux, Windows]
 - Python Version: [e.g. 3.12.0]
 - APM Version: [e.g. 0.1.0]
 - VSCode Version (if relevant): [e.g. 1.80.0]

**Logs**
If applicable, add any error logs or screenshots.

**Additional context**
Add any other context about the problem here.
```

Conventions when filling it in:

- **Title**: short, single-clause. Describe **what** the user observes (the
  symptom), not **why** it happens (the root cause) and not a two-clause summary.
  Save the mechanism for **Describe the bug**.
- **Describe the bug**: fold the root-cause analysis and source blob links here
  (`https://github.com/microsoft/apm/blob/main/src/apm_cli/...`).
- **Expected behavior**: user-visible behavior only (commands succeeding, files
  kept, exit 0) — never internal fix details (which function should be called).
- **Environment**: record the `apm --version` you actually observed on.
- **Logs**: real captured command output — never invented.
- **Additional context**: link to the minimal reproduction directory.
- **No local `.md` cross-links inside an issue.** The issue is filed on GitHub, so
  a link to a sibling `issue-<other>.md` is dead there. Cite related bugs by their
  filed **GitHub issue number** (`#1846`). If the related bug is not filed yet, ask
  the user to file it and supply the number (or leave a `#PLACEHOLDER` and flag it).
  This applies to `issue-*.md` only — README.md is the local index and may link
  local files freely.
- Record **current state and the finding as a whole** — no gradual findings, no
  "we chose X to isolate Y", no historical steps. State the fact, not the path to
  it. (The numbered "To Reproduce" steps are reproduction instructions, not
  narrative — those stay.)
- **One issue per root cause.** Same-cause variants stay in one issue (multiple
  `make` targets). A different cause is a separate issue; a deferred lead is a
  `handoff-<slug>.md`, which keeps its investigation framing.

## The feature request file

When the finding is a missing capability rather than broken behavior, write a
feature request instead. Create `feature-<slug>.md` from the canonical template
at the repo root (`feature-request-template.md`):

```markdown
# [FEATURE] xxxxx

**Is your feature request related to a problem? Please describe.**
A clear and concise description of what the problem is. Ex. I'm always frustrated when [...]

**Describe the solution you'd like**
A clear and concise description of what you want to happen.

**Describe alternatives you've considered**
A clear and concise description of any alternative solutions or features you've considered.

**Additional context**
Add any other context or screenshots about the feature request here.
```

Conventions when filling it in:

- **Title**: short, single-clause. Name the capability wanted, not the
  implementation.
- **Problem**: ground it in real friction actually hit while working — concrete
  commands and output, not hypotheticals.
- **Solution**: user-visible behavior only, same bar as a bug's expected
  behavior — never internal implementation details.
- The **no local `.md` cross-links** and **current-state-only** rules from bug
  issues apply here unchanged.

## The README entry

Add both:
- a one-line entry to the top index, and
- a per-bug section: brief setup + a `Reproduce` block listing the `make` targets
  and what each demonstrates.

Keep it minimal and in sync with the issue.
