# [FEATURE] `--recursive` on `apm install` / `apm audit` for monorepos with nested `apm.yml` packages

**Is your feature request related to a problem? Please describe.**

In a monorepo holding several APM packages — a root `apm.yml` plus
`packages/*/apm.yml`, each package with its own deployed mirror — `apm install`
and `apm audit` operate only on the directory they run from. Neither walks into
nested `apm.yml` directories, so editing a nested package's `.apm/` sources and
running `apm install` at the root refreshes the **root** mirror while the
package's own (git-tracked) mirror silently stays stale, and `apm audit --ci` at
the root passes anyway.

Minimal layout:

```
monorepo-root/
  apm.yml                        # dependencies.apm: [./packages/pkg]
  packages/pkg/
    apm.yml
    .apm/skills/demo/SKILL.md    # contains "marker: v1"
```

Both directories installed once (each has its lockfile and `.claude/` mirror at
`v1`). Then the nested package's skill source is edited to `marker: v2` and
`apm install` is re-run at the root:

```
$ apm install
[>] Installing dependencies from apm.yml...
  [+] ./packages/pkg (local)
  |-- 1 skill(s) integrated -> .claude/skills/

$ grep -rH 'marker:' .claude packages/pkg/.claude
.claude/skills/demo/SKILL.md:marker: v2
packages/pkg/.claude/skills/demo/SKILL.md:marker: v1    <- silently stale
```

And `apm audit --ci` at the root is blind to the stale nested mirror — only the
same command run *inside* the package catches it:

```
$ apm audit --ci
[*] All 9 check(s) passed                # exit 0

$ cd packages/pkg && apm audit --ci
[!] Drift detected: 1 file(s)
  modified (1):
    - .claude/skills/demo/SKILL.md  [.]
[x] 1 of 9 check(s) failed               # exit 1
```

The net effect: monorepos silently commit stale deployed mirrors. The root copy
looks right, per-package mirrors rot, and a root-level CI audit gate reports
green.

The inconsistency is that the root already **knows** these packages — they are
declared as local-path dependencies in the root `apm.yml`, and root `apm
install` resolves them and deploys their primitives into the root mirror on
every run. It just never refreshes their own mirrors, and audit never checks
them.

**Describe the solution you'd like**

A `--recursive` flag on both `apm install` and `apm audit`:

- discover every directory containing an `apm.yml` under the current directory
  (skipping `apm_modules/`),
- run the command in each of them, printing a per-directory status line,
- aggregate exit codes: non-zero if any directory fails (so `apm audit --ci
  --recursive` is a single CI gate for the whole monorepo).

Most of the plumbing already exists: `apm install --root DIR`
([`commands/install.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/install.py))
already installs "into DIR instead of $PWD", and its own help text cites
`npm install --prefix` as precedent — recursive install is essentially a foreach
over that primitive, in the same spirit as `pnpm -r` / `npm --workspaces`.

Open sub-decision worth settling in the design: should recursive `audit`
**classify** drift by origin — first-party (the package's own `.apm/` sources)
vs dependency (third-party content under `apm_modules/`)? In practice a root
mirror can carry long-lived drift in dependency-owned files, which makes a
root `apm audit --ci` gate permanently red; teams want to gate hard on
first-party drift while reporting dependency drift separately. A shell wrapper
cannot make that distinction; native audit can, since the lockfile records
which package owns each deployed file.

**Describe alternatives you've considered**

- A wrapper script that walks every `apm.yml` directory and runs
  `apm install` / `apm audit --ci` in each, with per-directory `OK`/`FAIL` and
  an aggregated exit code. This works (we run one today), but it has to be
  maintained twice for cross-platform teams (POSIX shell + PowerShell), and it
  can only aggregate exit codes — it cannot classify drift by origin as
  described above.
- Running the commands manually in each package directory — this is exactly the
  step that gets forgotten; the failure mode is silent (nothing warns that
  sibling mirrors exist and are stale).

**Additional context**

Observed on `apm` v0.24.1 (Linux, Python 3.14). The current flag surface has no
recursive/workspace mode anywhere: `apm --help`, `apm install --help`, and
`apm audit --help` contain no `--recursive`, `--workspace`, or foreach-style
option. Runnable demonstration: https://github.com/sproott/apm-bugs
(`make stale` / `make audit` in `recursive-install-audit/`).
