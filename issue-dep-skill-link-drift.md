# [BUG] `apm audit` reports false drift on a dependency's skill files right after install

**Describe the bug**

When an installed package ships a skill whose bundle contains an in-package
relative markdown link (a link to a sibling file that lives elsewhere in the
package, e.g. ``[pkg](../../README.md)``), `apm audit --ci` reports the deployed
skill file as `modified` immediately after a clean `apm install`, and exits
non-zero. Re-running `apm install` does not clear it.

On install, the skill integrator rewrites in-package asset links so the deployed
copy still resolves. `_resolve_in_package_asset_link`
([`compilation/link_resolver.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/compilation/link_resolver.py))
computes the relative path from the deployed skill dir to the linked file's
post-install location under `apm_modules/`. For a package installed at
`apm_modules/_local/package/`, install writes:

```
../../../apm_modules/_local/package/README.md
```

The audit-replay ([`install/drift.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/install/drift.py))
re-integrates from the lockfile into a scratch dir, calling
`integrate_package_primitives(package_info, scratch_root, ...)` so the link
resolver's `base_dir` is the scratch tmpdir, while the dependency's contents are
materialized at their real path (`_materialize_install_path` returns the real
`apm_modules/<...>` for remote deps, and the on-disk source dir for local deps).
The linked file therefore lies **outside** `base_dir`, which trips the
"replay-frame translation" branch added for #1182
([`link_resolver.py` L518–537](https://github.com/microsoft/apm/blob/main/src/apm_cli/compilation/link_resolver.py#L518-L537)):

```python
target_rel = ctx.target_location.relative_to(ctx.base_dir)  # .claude/skills/demo
relpath_anchor = ctx.package_root / target_rel              # <pkg>/.claude/skills/demo
relative_path = os.path.relpath(candidate, relpath_anchor)  # ../../../README.md
```

Re-anchoring onto `package_root` is only correct for the **self-package**, where
`package_root` equals the project root — dropping to a package-relative path then
equals the install-time deploy-relative path. For **any dependency**,
`package_root` is `apm_modules/<owner>/<repo>/` (or `apm_modules/_local/<name>/`),
nested under the project, so the re-anchor strips the whole
`apm_modules/.../` segment that a real install keeps. The replay produces
`../../../README.md` while disk holds `../../../apm_modules/_local/package/README.md`;
they differ byte-for-byte and every dependency skill file carrying an in-package
link is reported as drift. The #1182 fix closed this class of false drift for the
self-package only; it is still open for installed dependencies.

**To Reproduce**
1. `cd drift-dep-skill-links`
2. `make audit` — cleans, `apm install`, then `apm audit --ci`.
3. See `modified` drift on the just-installed skill files; exit non-zero.

```
cd drift-dep-skill-links && apm install && apm audit --ci
```

**Expected behavior**

A skill file just written by `apm install` is not reported as drifted by the
next `apm audit`. A clean install followed by an audit exits 0 with no drift.

**Environment (please complete the following information):**
 - OS: Linux (Arch Linux)
 - Python Version: 3.12
 - APM Version: 0.24.1

**Logs**

`apm audit --ci` right after a clean install:

```
  drift details:
    - modified: .claude/skills/demo/README.md
    - modified: .claude/skills/demo/SKILL.md

[x] 1 of 9 check(s) failed

[!] Drift detected: 2 file(s)

  modified (2):
    - .claude/skills/demo/README.md  [./package]
    - .claude/skills/demo/SKILL.md  [./package]

  [i] Run 'apm install' to re-sync deployed files with the lockfile.
```

The only difference on each flagged file is the rewritten link (deployed on disk
vs what the audit-replay recomputes):

```diff
-- [package README](../../../apm_modules/_local/package/README.md) — package overview
+- [package README](../../../README.md) — package overview
```

**Additional context**

Same root cause reproduces with a remote dependency (e.g. installing
`juliusbrussee/caveman`, which ships seven skills whose READMEs link to sibling
`agents/` files and the package README): six of them drift after a clean
install, the seventh — the one skill with no in-package relative link — does not.

Directly follows up #1182 ("audit-replay rewrites in-package asset links against
scratch dir, producing false drift"), whose fix handled only the self-package
frame. Minimal reproduction: https://github.com/sproott/apm-bugs
(`make audit` in `drift-dep-skill-links/`).
