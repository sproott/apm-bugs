# [FEATURE] `--global`/`-g` scope flag for `apm prune`

**Is your feature request related to a problem? Please describe.**

Packages installed to user scope with `apm install -g` accumulate orphans under
`~/.apm/apm_modules/` — drop a dependency from `~/.apm/apm.yml`, or `apm update
-g` past a transitive node, and the old package tree stays on disk. There is no
first-class way to clean them up: `apm prune` has no user-scope switch and always
operates on the current project.

```
$ apm prune -g
Usage: apm prune [OPTIONS]
Try 'apm prune --help' for help.

Error: No such option: -g
```

`prune` is the only installed-modules command left out of the scope model. Every
other command that reads or mutates installed packages already takes
`--global`/`-g` (user scope, `~/.apm/`):

```
install    --global   uninstall  --global   update   --global
outdated   --global   lock       --global   compile  --global
prune      (none)
```

The asymmetry is sharpest against its direct sibling: `apm uninstall -g <pkg>`
already removes a *named* package from user scope, so the machinery to resolve
and delete under `~/.apm/` exists and is wired up — `prune` (the graph-diff
removal, "delete whatever is no longer referenced") is simply never offered
there.

The only workaround is to lean on the fact that `~/.apm/apm.yml` doubles as a
project manifest and prune from inside it — but that is **incomplete**. It
removes the `apm_modules/` tree yet leaks the deployed primitive, because
user-scope deployment targets the home deploy root (`~/.claude/`, `~/.agents/`,
…) while a cwd-scoped prune only cleans files under `~/.apm/`:

```
$ cd ~/.apm && apm prune
[>] Analyzing installed packages vs apm.yml...
[!] Found 1 orphaned package(s):
[!]   - _local/pkg
[i] + Removed _local/pkg
[*] Pruned 1 orphaned package(s)

# apm_modules entry:  REMOVED
# deployed skill:      ~/.agents/skills/demo/SKILL.md   <- SURVIVES, now orphaned
```

So after the workaround the user deploy root still carries a skill (or agent,
hook, instruction) that no installed package owns — exactly the state `prune`
exists to prevent, just in the one scope it cannot reach.

The root cause is that `prune`
([`commands/prune.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/prune.py))
never imports `core.scope`. It hardcodes the project layout — `APM_YML_FILENAME`,
`APM_MODULES_DIR`, `Path.cwd()`, `Path(".")` — whereas the scope-aware commands
thread an `InstallScope` and resolve every path through `get_manifest_path`,
`get_modules_dir`, and `get_deploy_root`
([`core/scope.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/core/scope.py)).
It is a plumbing gap, not a documented non-goal: nothing in the command, its
help text, or the scope module marks prune as project-only by design.

**Describe the solution you'd like**

A `--global`/`-g` flag on `apm prune`, behaving exactly like the flag already
carried by `install`/`uninstall`/`update`:

- resolve the manifest, lockfile, installed modules, and deploy root from user
  scope (`~/.apm/apm.yml`, `~/.apm/apm_modules/`, and the home deploy root)
  instead of the current project,
- remove orphaned packages from `~/.apm/apm_modules/` **and** clean their
  deployed files from the user deploy root (`~/.claude/`, `~/.agents/`, …),
  leaving no orphaned primitive behind,
- compose with `--dry-run` (`apm prune -g --dry-run` lists user-scope orphans
  without deleting), matching the existing project-scope behaviour.

The help text should also gain a user-scope example, as `uninstall -g` and
`update -g` already have.

**Describe alternatives you've considered**

- `cd ~/.apm && apm prune` — the closest workaround, but incomplete as shown
  above: it prunes the `apm_modules/` tree and leaves the deployed primitives
  orphaned in the home deploy root, and it relies on the internal `~/.apm/`
  layout doubling as a project rather than any supported interface.
- `apm uninstall -g <pkg>` for each orphan — requires knowing every orphan by
  name and removing them one at a time; it is a targeted removal, not the
  "delete everything no longer in the resolved graph" sweep that `prune`
  provides, so it cannot catch unreferenced transitive nodes.
- Deleting `~/.apm/apm_modules/<...>` by hand and hand-editing the user lockfile
  and deployed mirror — error-prone and defeats the purpose of having a prune
  command at all.

**Additional context**

Observed on `apm` v0.25.0 (Linux, Python 3.14). `apm prune --help` exposes only
`--dry-run`; there is no `--global`, `-g`, `--user`, or scope option anywhere on
the command. Runnable demonstration: https://github.com/sproott/apm-bugs
(`make flag` / `make orphan` in `prune-global-scope/`, which pin an isolated
`HOME` so the demo never touches the real `~/.apm/`).
