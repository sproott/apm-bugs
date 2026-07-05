# [BUG] devDependencies are invisible to orphan detection

**Describe the bug**

`apm install` installs packages declared under `devDependencies.apm` and
records them in the lockfile (`is_dev: true`), but every consumer of the
"declared dependencies" set only looks at the production bucket, so any
installed dev dependency is seen as orphaned. This happens whether or not the
project also declares production dependencies:

- `apm prune` treats the just-installed dev dep as orphaned and **deletes it**
  from `apm_modules/` — even though its own help says it removes "packages
  *not listed in apm.yml*" (dev deps are listed), and there is no
  `--production`/`--omit=dev`-style flag that would make this intentional.
  `apm install` → `apm prune` → `apm install` oscillates forever.
- `apm audit --ci` fails the `no-orphaned-packages` check on the dev dep's
  lockfile entry and exits 1. The suggested remedy ("run 'apm install' to
  clean up") does nothing — install correctly keeps the dev dep — so the CI
  gate fails permanently.

(When the project declares *only* dev dependencies, the same prod-only view
also mislabels it as having no dependencies at all — `lockfile-exists: "No
dependencies declared -- lockfile not required"` — but the prune deletion and
orphan failure above do not depend on that.)

Single root cause: `APMPackage.get_apm_dependencies()`
([`models/apm_package.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/models/apm_package.py))
returns only `dependencies["apm"]`, and its sibling
`get_dev_apm_dependencies()` is never consulted by any of these paths:

- [`commands/prune.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/prune.py):
  `declared_deps = apm_package.get_apm_dependencies()` → dev deps missing from
  `expected_installed` → `safe_rmtree`'d as orphans.
- [`policy/ci_checks.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/policy/ci_checks.py):
  `_check_no_orphans` builds `manifest_keys` from `get_apm_dependencies()`;
  a dev dep's lockfile entry is direct (`resolved_by: null`), so it satisfies
  the orphan predicate. (`_check_lockfile_exists` gates on the equally
  prod-only `has_apm_dependencies()`, producing the "No dependencies
  declared" message in the dev-only case.)
- [`commands/_helpers.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/_helpers.py):
  `_check_orphaned_packages()` (the on-disk scan) makes the identical call.

**To Reproduce**
1. Declare a local package under `devDependencies.apm` in `apm.yml`
   (and nothing under `dependencies.apm`).
2. Run `apm install` — succeeds; package lands in `apm_modules/`, lockfile
   entry written with `is_dev: true`.
3. Run `apm audit --ci`:

```
│ [+]      │ lockfile-exists        │ No dependencies declared -- lockfile not │
│          │                        │ required                                 │
│ [x]      │ no-orphaned-packages   │ 1 orphaned package(s) in lockfile -- run │
│          │                        │ 'apm install' to clean up                │

  no-orphaned-packages details:
    - ./sub-package

[x] 1 of 5 check(s) failed
```

Exit code 1; re-running `apm install` does not clear it.

4. Run `apm prune` (or `--dry-run`):

```
[>] Analyzing installed packages vs apm.yml...
[!] Found 1 orphaned package(s):
[!]   - _local/sub-package
[i] + Removed _local/sub-package
[*] Pruned 1 orphaned package(s)
```

`apm_modules/` is now empty; the next `apm install` reinstalls the package.

**Expected behavior**

A dependency declared in `devDependencies.apm` and installed by `apm install`
is not an orphan: `apm prune` leaves it installed (npm parity: `npm prune`
keeps dev deps unless `--omit=dev` is passed), and `apm audit --ci` passes
right after a clean install.

**Environment**
 - OS: Linux (Arch Linux)
 - Python Version: 3.12
 - APM Version: 0.23.1

**Logs**

See the audit table and prune output above.

**Additional context**

Observed side inconsistency: after the (wrongful) prune, the `apm.lock.yaml`
entry for the removed package survives, so the lockfile and `apm_modules/`
disagree until the next install.

Minimal reproduction: https://github.com/sproott/apm-bugs (`make audit` and
`make prune` in `dev-dependencies/`).
