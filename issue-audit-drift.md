# [BUG] `apm audit` reports phantom drift immediately after a clean install

**Describe the bug**

Right after a successful `apm install`, `apm audit --ci` reports several hook
files as `modified` and exits non-zero. Re-running `apm install` does not clear
it.

Capturing the audit's replay scratch tree and diffing it against disk shows the
only difference on each flagged file is the `_apm_source` ownership marker:

```diff
# .claude/apm-hooks.json  (audit replay  vs  what `apm install` wrote)
-      "_apm_source": "hooks"
+      "_apm_source": "_local/hooks"
```

`HookIntegrator._get_hook_source_marker()` returns `"_local/<name>"` for the
project's own root-local content and the bare `<name>` otherwise, gated on:

```python
def _is_root_local_package(package_info, project_root):
    return Path(package_info.install_path).resolve() == Path(project_root).resolve()
```

- At **install** time `project_root` is the real project dir and equals the
  self-package's `install_path` → `True` → marker `"_local/hooks"`.
- During the **audit replay**, writes are redirected to a scratch dir, so the
  integrator's `project_root` is the scratch tmpdir while `install_path` still
  points at the real project → `False` → bare `"hooks"`.

So the replay stamps a different marker than install did, and every hook entry
carrying `_apm_source` differs byte-for-byte from disk. Normalization
(`utils/normalization.py`) does not neutralize the marker, so it surfaces as
drift. (The one hook file with no `_apm_source` marker — the Copilot file, which
never gets one — is the only file that does *not* drift.)

**To Reproduce**
1. Install a package that deploys hooks to targets which persist `_apm_source`
   (claude, codex, cursor, gemini, windsurf): `apm install`.
2. Run `apm audit --ci`.
3. See drift reported on the just-installed files; exit code 1.

```
[!] Drift detected: 5 file(s)
  modified (5):
    - .claude/apm-hooks.json
    - .codex/hooks.json
    - .cursor/hooks.json
    - .gemini/settings.json
    - .windsurf/hooks.json
```

**Expected behavior**

A file just written by `apm install` is not reported as drifted by the next
`apm audit`. The replay should compute the same root-local marker install did
(compare `install_path` against the original project root, not the scratch
redirect target), or `_apm_source` should be excluded from the drift comparison.

**Environment**
 - OS: Linux (Arch Linux)
 - Python Version: 3.12
 - APM Version: 0.23.1

**Logs**

See the drift block above.

**Additional context**

Minimal reproduction: https://github.com/sproott/apm-bugs (`make audit` in `hooks/`).
