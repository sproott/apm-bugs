# [BUG] `apm uninstall` rejects the `_local/<name>` key that `apm deps list` prints for a local dependency

**Describe the bug**

A local (filesystem-path) dependency is displayed by `apm deps list` under the
key `_local/<name>`, but `apm uninstall _local/<name>` reports
`not found in apm.yml` and removes nothing. The only accepted key is the exact
declared path — `../mypkg` in a project, an absolute path at user scope — so the
identifier the CLI shows the user cannot be pasted back into the CLI to remove it.

The two commands key a local dependency on different fields:

- `apm deps list` renders it from the lockfile/reference `repo_url`, which for
  every local dep is the synthetic `_local/<name>`
  ([`commands/deps/cli.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/deps/cli.py),
  `_dep_display_name` falls back to `dep.repo_url`; the installed-scan path prints
  `dep.repo_url` directly).
- `apm uninstall` matches each argument against the `apm.yml` entries by
  `DependencyReference.get_identity()`, which for a local dep is the **`local_path`**
  (`../mypkg`), not the `repo_url`
  ([`models/dependency/reference.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/models/dependency/reference.py),
  `get_identity` returns `self.local_path` when `is_local`).

`_local/<name>` is never recognized as a local path (`is_local_path` only matches
`./`, `../`, `/`, `~`, or a Windows drive), so `apm uninstall` parses it as a
remote `owner/repo` with identity `_local/<name>`, which no local dep's identity
(`../mypkg`) ever equals — hence "not found". All three forms share the same
`repo_url`, which is exactly why the displayed key is ambiguous:

```
'../mypkg'      -> is_local_path=True   identity='../mypkg'      repo_url='_local/mypkg'
'/abs/mypkg'    -> is_local_path=True   identity='/abs/mypkg'    repo_url='_local/mypkg'
'_local/mypkg'  -> is_local_path=False  identity='_local/mypkg'  repo_url='_local/mypkg'
```

This affects **both scopes**. User scope (`-g`) is worse: local deps must be
installed by absolute path (relative paths are rejected there), so the only
accepted uninstall key is a machine-specific absolute path that appears **nowhere**
in `apm deps list -g` output. The failed uninstall also exits `0`, so a script
that removes by the displayed key silently no-ops.

**To Reproduce**

Project scope:

1. A package `consumer/apm.yml` declares a local dep by relative path: `- ../mypkg`.
2. `cd consumer && apm install`
3. `apm deps list` → the Package column shows `_local/mypkg`.
4. `apm uninstall _local/mypkg` → `_local/mypkg - not found in apm.yml` (exit 0, nothing removed).
5. `apm uninstall ../mypkg` → succeeds.

User scope:

1. `apm install -g /abs/path/to/mypkg`
2. `apm deps list -g` → the Package column shows `_local/mypkg`.
3. `apm uninstall -g _local/mypkg` → `not found in apm.yml` (exit 0).
4. `apm uninstall -g /abs/path/to/mypkg` → succeeds (the absolute path is shown nowhere).

**Expected behavior**

The identifier `apm deps list` prints for a local dependency is accepted by
`apm uninstall` to remove that dependency, in both project and user scope.
Passing a key that matches no installed dependency exits non-zero.

**Environment (please complete the following information):**

- OS: Linux (Arch, kernel 7.1.4)
- Python Version: 3.14.6
- APM Version: 0.26.0 (64a7fb5)

**Logs**

Project scope:

```
$ apm deps list
                              APM Dependencies (Project)
┏━━━━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━┳ ...
┃ Package      ┃ Version ┃ Source ┃ ...
┡━━━━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━╇ ...
│ _local/mypkg │ 1.0.0   │ local  │ ...
└──────────────┴─────────┴────────┴ ...

$ apm uninstall _local/mypkg
[>] Uninstalling 1 package(s)...
[!] _local/mypkg - not found in apm.yml
[!] No packages found in apm.yml to remove
$ echo $?
0

$ apm uninstall ../mypkg
[>] Uninstalling 1 package(s)...
[+] ../mypkg - found in apm.yml
[i] Removed ../mypkg from dependencies.apm in apm.yml
[i] Removed ../mypkg from apm_modules/
[*] Uninstall complete: Removed 1 package(s) from apm.yml, Removed 1 package(s) from apm_modules/
```

User scope:

```
$ apm deps list -g
                               APM Dependencies (Global)
│ _local/mypkg │ 1.0.0   │ local  │ ...

$ apm uninstall -g _local/mypkg
[i] Uninstalling from user scope (~/.apm/)
[>] Uninstalling 1 package(s)...
[!] _local/mypkg - not found in apm.yml
[!] No packages found in apm.yml to remove
```

**Additional context**

Minimal reproduction: [`uninstall-local-key-mismatch/`](uninstall-local-key-mismatch/)
(`make project`, `make user`).

Related, different root cause: #2270 (uninstall reachability preservation —
*which* transitive deps to keep), and the `deps list` vs `prune`/lockfile local-key
divergence in the same `_dep_display_name` / `repo_url` area.
