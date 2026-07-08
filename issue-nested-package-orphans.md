# [BUG] A dependency's subpackage is reported as an orphan and pruned

**Describe the bug**

When an installed package's source tree contains a nested `apm.yml` (e.g. a
package that declares a dependency on a relative subpath such as
`./sub-package`, whose source lives inside it), the on-disk package scan treats
that nested directory as a **separate, top-level orphaned package**:

- `apm deps list` reports it as a direct `orphaned` project dependency.
- `apm prune` deletes it out of the installed parent package in `apm_modules/`,
  so `apm install` → `apm prune` → `apm install` oscillates forever.
- `apm audit --ci` passes throughout — its `no-orphaned-packages` check is
  lockfile-based and the nested directory has no lockfile entry — so `prune`
  and the CI gate disagree.

The nested subpackage is never actually a dependency of the consumer: the
consumer lockfile records only the parent (`./package`). It is only misreported
as one because the scan walks the filesystem into the installed package.

Root cause:
[`_scan_installed_packages()`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/deps/_utils.py)
does `apm_modules_dir.rglob("*")` and returns every nested directory with an
`apm.yml`/`.apm`, without calling its sibling guard `_is_nested_under_package()`.
That guard exists, but is only wired into the
[`deps` display scan](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/deps/cli.py)
and even there is gated on `has_skill_md and not has_apm_yml` — it suppresses
nested *skill* dirs only, never a nested `apm.yml`. So
[`prune`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/prune.py)
`safe_rmtree`s the subdirectory as an orphan, while
[`no-orphaned-packages`](https://github.com/microsoft/apm/blob/main/src/apm_cli/policy/ci_checks.py)
(lockfile-based) sees nothing wrong.

**To Reproduce**
1. Have a project depend on a local `./package` whose `apm.yml` declares a
   subpath dependency (e.g. `devDependencies.apm: [./sub-package]`), with the
   source at `package/sub-package/apm.yml`.
2. Run `apm install` — installs `./package` into `apm_modules/_local/package/`;
   its `sub-package/` subfolder is copied along as part of the source.
3. Run `apm deps list`:

```
│ _local/package             │ 1.0.0   │ local    │ ... │
│ _local/package/sub-package │ 1.0.0   │ orphaned │ ... │
[!] 1 orphaned package(s) found (not in apm.yml):
[!]   - _local/package/sub-package
[i] Run 'apm prune' to remove orphaned packages
```

4. Run `apm audit --ci` — passes (`no-orphaned-packages` green), contradicting
   the list above.
5. Run `apm prune`:

```
[!] Found 1 orphaned package(s):
[!]   - _local/package/sub-package
[i] + Removed _local/package/sub-package
[*] Pruned 1 orphaned package(s)
```

   `apm_modules/_local/package/sub-package/` is now gone. The next `apm install`
   restores it; `prune` removes it again.

**Expected behavior**

A directory that lives inside an already-installed package is part of that
package, not a separate top-level dependency. `apm prune` must not delete it,
`apm deps list` must not report it as an orphan, and this must agree with what
`apm audit` reports.

**Environment**
 - OS: Linux (Arch Linux)
 - Python Version: 3.14
 - APM Version: 0.24.0

**Logs**

See the `apm deps list` and `apm prune` output above.

**Additional context**

Minimal reproduction: https://github.com/sproott/apm-bugs
(`nested-package-orphans/` — `make audit` and `make prune`).

Adjacent but distinct: #1050 (`apm compile` falsely flags the **parent**
directory of a subdirectory virtual package as orphaned) is the same
`_scan_installed_packages` scanner but the opposite topology — there the false
orphan is an *ancestor* of an expected install path, fixed by expanding expected
paths with their ancestors. Here the false orphan is a *descendant* nested
**inside** an installed package (`_local/package/sub-package`), which ancestor
expansion does not cover; the `_is_nested_under_package` guard that would suppress
it is only applied to nested skill dirs, not a nested `apm.yml`. Reproduces on
0.24.0 (after #1050's fix).
