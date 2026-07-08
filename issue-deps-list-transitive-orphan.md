# [BUG] `apm deps list` flags a local transitive dependency as orphaned

**Describe the bug**

When a local package declares a **production** subpath dependency, that
transitive dep is correctly installed and recorded in the consumer lockfile (with
`depth: 2`), but `apm deps list` labels it `orphaned` while `apm prune` (correctly)
reports the tree clean. The two commands disagree on the same install.

Root cause is a key mismatch between how `deps list` and the filesystem scan
identify a local transitive dep:

- The on-disk scan and
  [`prune`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/prune.py)
  key a local package by its **install path** —
  [`get_install_path()`](https://github.com/microsoft/apm/blob/main/src/apm_cli/models/dependency/reference.py)
  returns `apm_modules/_local/<dir-name>`, i.e. the scan key `_local/sub-package`.
  `prune` builds its expected-install set from that same helper, so it matches.
- [`deps list`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/deps/cli.py)
  has a loop over `lockfile.dependencies.values()` — commented as existing "to
  avoid false orphan flags on transitive deps" — that keys each lockfile dep by
  [`get_canonical_dependency_string()`](https://github.com/microsoft/apm/blob/main/src/apm_cli/deps/lockfile.py).
  For a local dep that resolves to
  [`build_canonical_dependency_string()`](https://github.com/microsoft/apm/blob/main/src/apm_cli/models/dependency/identity.py),
  which returns the **raw `local_path`** (`if is_local and local_path: return
  local_path`) — `./sub-package`, not `_local/sub-package`.

So the lockfile entry lands in `declared_sources` under the key `./sub-package`,
the scan tests the key `_local/sub-package`, the two never match, and the genuine
transitive install is flagged `orphaned`. The loop that was meant to prevent
exactly this false orphan is keyed on the wrong string.

**To Reproduce**
1. Have a project depend on a local `./package` whose `apm.yml` declares a
   **production** dependency on a sibling package (`dependencies.apm:
   [../sub-package]`), with the source at `sub-package/apm.yml`.
2. Run `apm install` — installs `./package` into `apm_modules/_local/package/`
   and the transitive `../sub-package` into `apm_modules/_local/sub-package/`.
   The consumer lockfile records the transitive dep as a legitimate install:

```yaml
- repo_url: _local/sub-package
  name: sub-package
  depth: 2
  resolved_by: _local/package
  source: local
  local_path: ../sub-package
```

3. Run `apm deps list` — flags `_local/sub-package` as orphaned:

```
│ _local/package     │ 1.0.0   │ local    │ ... │
│ _local/sub-package │ 1.0.0   │ orphaned │ ... │
[!] 1 orphaned package(s) found (not in apm.yml):
[!]   - _local/sub-package
[i] Run 'apm prune' to remove orphaned packages
```

4. Run `apm prune --dry-run` — reports no orphans, contradicting the list above:

```
[>] Analyzing installed packages vs apm.yml...
[+] No orphaned packages found. apm_modules/ is clean.
```

**Expected behavior**

A production transitive dependency that is recorded in the consumer lockfile is a
real install, not an orphan. `apm deps list` must not flag `_local/sub-package`
as orphaned, and must agree with `apm prune`.

**Environment**
 - OS: Linux (Arch Linux)
 - Python Version: 3.14
 - APM Version: 0.24.0

**Logs**

See the `apm deps list` and `apm prune --dry-run` output above.

**Additional context**

Minimal reproduction: https://github.com/sproott/apm-bugs
(`deps-list-transitive-orphan/` — `make deps-list` and `make prune`).

Distinct from the nested-package orphan bug (#2067): that one invents a phantom
`_local/package/sub-package` never in the lockfile and is flagged by both `deps
list` and `prune`; this one mislabels a **real** lockfile entry
(`_local/sub-package`, `depth: 2`) and is flagged by `deps list` only. The sibling
(`../sub-package`) layout keeps the transitive source out of the installed package
dir, so this repro exhibits only the key-mismatch bug.

Adjacent but distinct: #1846 (`no-orphaned-packages` falsely flags **remote**
transitive deps declared by a local-path sub-package) is the audit/CI-check path
and is caused by `resolved_by` being null. Here `resolved_by` is correctly set
(`_local/package`), the affected command is `apm deps list`, and the cause is the
`local_path`-vs-`_local/<name>` key mismatch above.
