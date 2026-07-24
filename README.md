# apm-bugs

Minimal reproductions of bugs in [APM (Agent Package Manager)](https://github.com/microsoft/apm),
observed with the `apm` CLI **v0.23.1**. Full write-ups (root cause + fix) are in
the issue files:

- [`issue-copilot-rename.md`](issue-copilot-rename.md) — Copilot hook events are never renamed to camelCase.
- [`issue-audit-drift.md`](issue-audit-drift.md) — `apm audit` reports phantom drift right after a clean install.
- [`issue-devdeps.md`](issue-devdeps.md) — devDependencies are invisible to orphan
  detection: `apm prune` deletes any installed dev dep, `apm audit --ci` fails on it.
- [`issue-nested-package-orphans.md`](issue-nested-package-orphans.md) — a
  dependency's subpackage (a nested `apm.yml` from a subpath dep) is reported as
  a top-level orphan: `apm deps list` and `apm prune` treat it as a direct
  orphaned dep and prune deletes it, while `apm audit --ci` passes — the two
  disagree. (Observed on `apm` **v0.24.0**.)
- [`issue-deps-list-transitive-orphan.md`](issue-deps-list-transitive-orphan.md) —
  a local **production** transitive dep, recorded in the consumer lockfile with
  `depth: 2`, is labelled `orphaned` by `apm deps list` while `apm prune` reports
  the tree clean — the two disagree. Root cause: `deps list` keys its
  lockfile-inclusion loop by the raw `local_path` (`./sub-package`), mismatching
  the filesystem scan key `_local/sub-package`. (Observed on `apm` **v0.24.0**.)
- [`issue-deps-tree-subpackage-recursion.md`](issue-deps-tree-subpackage-recursion.md) —
  `apm deps tree` self-nests a **remote** package's same-repo virtual
  sub-packages (root → inner → leaf) recursively into one another, up to the
  depth-5 cap, repeating every sub-package and its skills at each level. Root
  cause: the tree keys parent↔child on `repo_url`, which is not unique across
  virtual sub-packages of one repo — it should use `get_unique_key()`
  (repo_url + virtual_path). Display-only; the lockfile and `apm deps list` are
  correct. (Observed on `apm` **v0.24.0**.)
- [`issue-init-target-alias-leak.md`](issue-init-target-alias-leak.md) — `apm init
  --target` writes a `targets:` value that `apm install` then rejects: the flag's
  parser resolves `copilot`/`agents` to the internal name `vscode` (multi-token
  input) or passes `agents` through verbatim (single-token), but neither is a
  manifest-canonical target — so the scaffolded `apm.yml` aborts the next
  `apm install` with `Unknown target`. (Observed on `apm` **v0.24.0**.)
- [`issue-dep-skill-link-drift.md`](issue-dep-skill-link-drift.md) — `apm audit
  --ci` reports false `modified` drift on a dependency's skill files right after a
  clean install, whenever a skill bundle contains an in-package relative link
  (e.g. `[pkg](../../README.md)`). Root cause: the audit-replay's link rewrite
  re-anchors onto the dependency's `package_root`, dropping the
  `apm_modules/<...>/` segment the real install keeps — the #1182 fix closed this
  only for the self-package. (Observed on `apm` **v0.24.1**.)
- [`issue-copilot-user-hook-relative-path.md`](issue-copilot-user-hook-relative-path.md) —
  a user-scope (`apm install -g`) Copilot install deploys a hook to
  `~/.copilot/hooks/` but leaves its `command` repo-relative
  (`node .copilot/hooks/scripts/...`), unresolvable from any cwd ≠ `$HOME`. Every
  other user-scope target rewrites hook commands to absolute paths; the Copilot
  branch alone never threads `user_scope` into the rewrite, so the #1310/#1354 fix
  skips it. (Observed on `apm` **v0.25.0**.)
- [`issue-init-noninteractive-metadata.md`](issue-init-noninteractive-metadata.md)
  (**RFE**, not a bug) — `apm init`'s interactive mode prompts for name, version,
  description, and author, but its non-interactive mode (`--yes` / non-TTY) can
  freely set none of them. No `--description`/`--version`/`--author` flags exist,
  and the positional name arg always makes a subdir — so you can't name a project
  in the current dir non-interactively (bare `--yes` forces the dir basename).
  Scripted/CI init is stuck with boilerplate defaults (`description: APM project
  for <name>`) and must hand-edit `apm.yml` — even though sibling `apm marketplace
  init` already has `--name`/`--owner`. (Observed on `apm` **v0.24.0**.)
- [`issue-uninstall-local-key-mismatch.md`](issue-uninstall-local-key-mismatch.md) —
  `apm deps list` prints a local dependency under the key `_local/<name>`, but
  `apm uninstall _local/<name>` reports `not found in apm.yml` and removes nothing
  (exit 0). `deps list` renders local deps from `repo_url` (`_local/<name>`) while
  `uninstall` matches on `get_identity()` (the declared `local_path`, e.g.
  `../mypkg`), so the shown key never round-trips. Both scopes; user scope is worse
  — the only accepted key is a machine-specific absolute path shown nowhere.
  (Observed on `apm` **v0.26.0**.)
- [`feature-recursive-install-audit.md`](feature-recursive-install-audit.md)
  (**RFE**, not a bug) — `apm install` and `apm audit` operate only on the
  directory they run from: in a monorepo with nested `apm.yml` packages, root
  `apm install` refreshes the root mirror but leaves each package's own
  (git-tracked) mirror silently stale, and root `apm audit --ci` passes anyway —
  only a per-package audit catches the drift. Proposes `--recursive` on both,
  reusing the existing `install --root DIR` primitive. (Observed on `apm`
  **v0.24.1**.)

## Hook bugs (`hooks/`)

### Setup

The hook bug reproductions live in the [`hooks/`](hooks/) directory: one hook
file authored in the Claude (PascalCase) shape, deployed to several targets:

```
hooks/.apm/hooks/test.json   # { "hooks": { "PostToolUse": [...] } }
hooks/apm.yml                # name: hooks; targets: claude, codex, copilot,
                             #                       cursor, gemini, opencode, windsurf
```

`make clean` (run inside `hooks/`) removes the lockfile and all generated
(git-ignored) target dirs.

### Reproduce

```bash
apm --version               # 0.23.1
cd hooks

make copilot-rename         # Bug 1: clean + install
                            #   -> casing warning; .github/hooks/hooks-test.json keeps `PostToolUse`
                            #      (in the same run gemini is correctly renamed to `AfterTool`)

make audit                  # Bug 2: clean + install + `apm audit --ci`
                            #   -> phantom drift on 5 just-installed files, exit 1
```

Files generated by an install (all under `hooks/`):

| Target  | File                             | Event key       | `_apm_source`  |
|---------|----------------------------------|-----------------|----------------|
| claude  | `.claude/apm-hooks.json`         | `PostToolUse`   | `_local/hooks` |
| codex   | `.codex/hooks.json`              | `PostToolUse`   | `_local/hooks` |
| cursor  | `.cursor/hooks.json`             | `PostToolUse`   | `_local/hooks` |
| gemini  | `.gemini/settings.json`          | `AfterTool`     | `_local/hooks` |
| windsurf| `.windsurf/hooks.json`           | `PostToolUse`   | `_local/hooks` |
| copilot | `.github/hooks/hooks-test.json`  | `PostToolUse` ❌ | *(none)*      |

## Copilot user-scope hook relative path (`copilot-user-hook-relative-path/`)

### Setup

A local package with a script-backed hook, installed to **user scope**
(`apm install -g`). At user scope Copilot reads `~/.copilot/`, so the hook file
lands there — but its `command` is left repo-relative and is resolved against the
process cwd, not the file that declares it:

```
copilot-user-hook-relative-path/pkg/apm.yml                     # targets: copilot
copilot-user-hook-relative-path/pkg/.apm/hooks/notify.json      # command: node ./scripts/notify.js
copilot-user-hook-relative-path/pkg/.apm/hooks/scripts/notify.js
```

All `-g` commands run under an isolated fake `$HOME`, so the demo never touches the
real `~/.apm/`, `~/.copilot/`, or `~/.claude/`. `make clean` removes it.

### Reproduce

```bash
apm --version               # 0.25.0
cd copilot-user-hook-relative-path

make copilot                # install -g, Copilot target
                            #   -> ~/.copilot/hooks/pkg-notify.json command is
                            #      `node .copilot/hooks/scripts/pkg/scripts/notify.js` (relative)

make claude                 # same package/hook, install -g --target claude
                            #   -> ~/.claude/settings.json command is absolute

make reproduce              # both of the above
```

## devDependencies bug (`dev-dependencies/`)

### Setup

The root [`dev-dependencies/apm.yml`](dev-dependencies/apm.yml) declares a
**dev** dependency — the local package
[`sub-package/`](dev-dependencies/sub-package/) — under `devDependencies.apm`:

```yaml
devDependencies:
  apm:
    - ./sub-package
```

`apm install` installs it into `apm_modules/_local/sub-package/` and locks it
with `is_dev: true`, but prune and audit both build their "declared" set from
the production bucket only, so *any* installed dev dep counts as an orphan —
this happens regardless of whether production dependencies are also declared.
(This reproduction happens to declare only the dev dep, which additionally
makes `lockfile-exists` report "No dependencies declared".)

`make clean` (run inside `dev-dependencies/`) removes the lockfile and all
generated target dirs.

### Reproduce

```bash
apm --version               # 0.23.1
cd dev-dependencies

make audit                  # clean + install + `apm audit --ci`
                            #   -> no-orphaned-packages fails on ./sub-package, exit 1
                            #      (and lockfile-exists claims "No dependencies declared")

make prune                  # clean + install + `apm prune`
                            #   -> _local/sub-package deleted from apm_modules/;
                            #      the follow-up `apm install` puts it right back
```

## Nested-package orphan bug (`nested-package-orphans/`)

### Setup

The root [`nested-package-orphans/apm.yml`](nested-package-orphans/apm.yml) has a
single **production** dependency, the local package
[`package/`](nested-package-orphans/package/), which in turn declares a
dependency on a relative subpath whose source lives inside it at
[`package/sub-package/`](nested-package-orphans/package/sub-package/):

```yaml
# package/apm.yml
devDependencies:
  apm:
    - ./sub-package
```

`apm install` copies `package` into `apm_modules/_local/package/`, and its
`sub-package/` subfolder comes along as part of the source. The subpackage is
**not** a dependency of the consumer (its lockfile lists only `./package`), but
the on-disk scan walks into the installed package and reports the nested
`sub-package/` as a direct `orphaned` project dependency — so `apm prune`
deletes it, while `apm audit --ci` (lockfile-based) reports everything is fine.
The two disagree. Full write-up in
[`issue-nested-package-orphans.md`](issue-nested-package-orphans.md).

Different root cause from the [`dev-dependencies/`](dev-dependencies/) bug above:
that one omits dev deps from the *declared* set; this one invents phantom
packages in the *installed* set.

`make clean` removes the lockfile, installed `apm_modules/`, and generated
target dirs.

### Reproduce

```bash
apm --version               # 0.24.0
cd nested-package-orphans

make audit                  # install + `apm deps list` + `apm audit --ci`
                            #   -> deps list flags _local/package/sub-package as orphaned,
                            #      but apm audit --ci passes: the two disagree

make prune                  # install + `apm prune` + install
                            #   -> _local/package/sub-package deleted (a subfolder of the
                            #      installed package); the next apm install restores it
```

## Transitive-orphan bug (`deps-list-transitive-orphan/`)

### Setup

The root [`deps-list-transitive-orphan/apm.yml`](deps-list-transitive-orphan/apm.yml)
has a single **production** dependency, the local package
[`package/`](deps-list-transitive-orphan/package/), which declares its own
**production** dependency on the sibling package
[`sub-package/`](deps-list-transitive-orphan/sub-package/):

```yaml
# package/apm.yml
dependencies:
  apm:
    - ../sub-package
```

`apm install` resolves the transitive `../sub-package` into the consumer and
locks it as a real install (`_local/sub-package`, `depth: 2`, `resolved_by:
_local/package`). The sibling (`../`) layout keeps its source outside the
installed `package/` dir, so nothing nested is scanned — unlike the
[`nested-package-orphans/`](nested-package-orphans/) bug above. Yet `apm deps
list` labels `_local/sub-package` `orphaned` while `apm prune` reports the tree
clean: the two disagree. Root cause: `deps list` keys its lockfile-inclusion loop
by the raw `local_path`, mismatching the `_local/<name>` filesystem scan key that
`prune` (correctly) uses. Full write-up in
[`issue-deps-list-transitive-orphan.md`](issue-deps-list-transitive-orphan.md).

`make clean` removes the lockfile, installed `apm_modules/`, and generated
target dirs.

### Reproduce

```bash
apm --version               # 0.24.0
cd deps-list-transitive-orphan

make deps-list              # clean + install + `apm deps list`
                            #   -> flags _local/sub-package as orphaned, despite the
                            #      lockfile recording it as a depth-2 install

make prune                  # clean + install + `apm prune --dry-run`
                            #   -> "No orphaned packages found": disagrees with deps list
```

## Tree self-nesting bug (`deps-tree-subpackage-recursion/`)

### Setup

The root
[`deps-tree-subpackage-recursion/apm.yml`](deps-tree-subpackage-recursion/apm.yml)
depends on a fixture published as a **remote** virtual sub-package of this same
repo:

```yaml
# apm.yml
dependencies:
  apm:
    - sproott/apm-bugs/deps-tree-subpackage-recursion/fixture#main
```

The fixture ([`fixture/`](deps-tree-subpackage-recursion/fixture/)) chains three
same-repo sub-packages via local paths — root → `./packages/inner` → `../leaf`
(the leaf carries one skill). Because the parent is **remote**, those local
paths are expanded to inherit the parent's `repo_url`, so all three
sub-packages share one `repo_url` and differ only by `virtual_path`.

`apm install` resolves exactly three packages (distinct `virtual_path`, correct
`depth`), and `apm deps list` prints three rows — resolution is correct. But
`apm deps tree` keys parent↔child on `repo_url` (not `get_unique_key()`), so
every same-repo sub-package becomes a child of every other and the tree
self-nests until the depth-5 recursion cap, repeating `fixture-leaf` and its
skill at each level. Display-only. Full write-up in
[`issue-deps-tree-subpackage-recursion.md`](issue-deps-tree-subpackage-recursion.md).

The fixture is fetched from GitHub, so reproduction needs network access and the
fixture pushed to `sproott/apm-bugs` `main`.

`make clean` removes the lockfile, installed `apm_modules/`, and generated
target dirs.

### Reproduce

```bash
apm --version               # 0.24.0
cd deps-tree-subpackage-recursion

make tree                   # clean + install + `apm deps tree`
                            #   -> root/inner/leaf self-nested five levels deep;
                            #      leaf + its skill repeated at every level

make audit                  # clean + install + `apm deps list` + `apm audit --ci`
                            #   -> deps list shows exactly 3 packages, audit passes:
                            #      resolution is correct, only the tree display is wrong
```


## Dependency skill-link drift (`drift-dep-skill-links/`)

### Setup

The root [`drift-dep-skill-links/apm.yml`](drift-dep-skill-links/apm.yml) has a
single local dependency, the package
[`package/`](drift-dep-skill-links/package/), which ships one skill
([`skills/demo/`](drift-dep-skill-links/package/skills/demo/)) whose bundle links
to a sibling file inside the package (the package README, `../../README.md`).

On install the skill integrator rewrites that in-package link so the deployed
copy still resolves — from `.claude/skills/demo/` down to the file's post-install
location: `../../../apm_modules/_local/package/README.md`. The `apm audit --ci`
drift replay re-integrates from the lockfile into a scratch dir and recomputes
the link as `../../../README.md` (the `apm_modules/_local/package/` segment
dropped), so every deployed skill file carrying an in-package link differs
byte-for-byte from disk and is reported as `modified` drift. Re-running `apm
install` does not clear it. Full write-up in
[`issue-dep-skill-link-drift.md`](issue-dep-skill-link-drift.md).

Distinct root cause from the [`hooks/`](hooks/) phantom-drift bug: that one is
the hook `_apm_source` ownership marker on the self-package; this one is
dependency skill-bundle link rewriting mis-anchored during replay.

`make clean` removes the lockfile, installed `apm_modules/`, and generated
target dirs.

### Reproduce

```bash
apm --version               # 0.24.1
cd drift-dep-skill-links

make audit                  # clean + install + `apm audit --ci`
                            #   -> false `modified` drift on .claude/skills/demo/
                            #      README.md and SKILL.md, exit 1 (drift persists
                            #      across re-install)
```

## init target-alias leak (`init-target-alias-leak/`)

### Setup

Three scenario directories, each holding one checked-in primitive
([`.apm/instructions/demo.instructions.md`](init-target-alias-leak/control/.apm/instructions/demo.instructions.md))
and nothing else — `apm.yml` is *generated* by `apm init`, because it is the
artifact under test.

`apm init --target <list>` writes the `--target` flag's parsed value straight
into `targets:`. The flag's parser accepts the aliases `copilot`/`agents`/`vscode`
and resolves them to the internal name `vscode` (multi-token input only); the
manifest validator accepts neither `vscode` nor `agents`. So `init` can scaffold
an `apm.yml` that the very next `apm install` refuses to read, exit 2. Full
write-up in [`issue-init-target-alias-leak.md`](issue-init-target-alias-leak.md).

`make clean` removes the generated `apm.yml`, lockfile, `apm_modules/`, and
generated target dirs from every scenario.

### Reproduce

```bash
apm --version               # 0.24.0
cd init-target-alias-leak

make control                # clean + `apm init --target copilot` + install
                            #   -> targets: [copilot], install succeeds (exit 0)

make multi                  # clean + `apm init --target claude,copilot` + install
                            #   -> targets: [claude, vscode] -- copilot silently renamed;
                            #      install aborts: Unknown target 'vscode' (exit 2)

make agents                 # clean + `apm init --target agents` + install
                            #   -> targets: [agents] passed through verbatim;
                            #      install aborts: Unknown target 'agents' (exit 2)

make reproduce              # all three in order
```

## Uninstall rejects the `_local/<name>` key (`uninstall-local-key-mismatch/`)

### Setup

A local dependency is declared by path. In the project scenario,
[`consumer/apm.yml`](uninstall-local-key-mismatch/consumer/apm.yml) declares the
sibling package [`mypkg/`](uninstall-local-key-mismatch/mypkg/) by relative path:

```yaml
# consumer/apm.yml
dependencies:
  apm:
    - ../mypkg
```

`apm deps list` renders it in the Package column as `_local/mypkg` (from the
dependency's `repo_url`), but `apm uninstall` matches its arguments on
`get_identity()` — the declared `local_path` (`../mypkg`) — so
`apm uninstall _local/mypkg` reports `not found in apm.yml` and removes nothing
(exit 0). The displayed key does not round-trip. Full write-up in
[`issue-uninstall-local-key-mismatch.md`](issue-uninstall-local-key-mismatch.md).

The `user` target repeats this under user scope with an isolated `HOME=fake-home/`
(the real `~/.apm` is untouched); there the dep is installed by absolute path
(relative paths are rejected at user scope), so the only accepted uninstall key is
a machine-specific absolute path shown nowhere in `apm deps list -g`.

`make clean` removes the installed `apm_modules/`, lockfile, generated target dirs,
and `fake-home/`.

### Reproduce

```bash
apm --version               # 0.26.0
cd uninstall-local-key-mismatch

make project                # clean + install + `apm deps list` + `apm uninstall _local/mypkg`
                            #   -> deps list shows key `_local/mypkg`;
                            #      uninstall by that key: "not found in apm.yml" (exit 0, no-op)

make user                   # same under `-g` with isolated HOME=fake-home/
                            #   -> deps list -g shows `_local/mypkg`;
                            #      uninstall -g by that key: "not found in apm.yml"
```

## Recursive install/audit gap (`recursive-install-audit/`) — RFE

### Setup

A two-level monorepo: the root
[`recursive-install-audit/apm.yml`](recursive-install-audit/apm.yml) declares
the nested package [`packages/pkg/`](recursive-install-audit/packages/pkg/) as
a local-path dependency. The nested package's sentinel skill source
(`packages/pkg/.apm/skills/demo/SKILL.md`, `marker: v1`) is *generated* by the
Makefile, because the reproduction then mutates it to `marker: v2` — the
stand-in for any real edit to a nested package's authored primitives.

Root `apm install` re-resolves the local dep and deploys `v2` into the root
mirror, but never refreshes the nested package's own mirror — and root
`apm audit --ci` never checks it. Only `apm audit --ci` run inside the package
catches the drift. Write-up (as a feature request for `--recursive`) in
[`feature-recursive-install-audit.md`](feature-recursive-install-audit.md).

`make clean` removes lockfiles, `apm_modules/`, generated mirrors, and the
generated sentinel source from both levels.

### Reproduce

```bash
apm --version               # 0.24.1
cd recursive-install-audit

make stale                  # clean + install both dirs + edit nested source + root install
                            #   -> root .claude/ at v2; packages/pkg/.claude/ stale at v1

make audit                  # stale + `apm audit --ci` at root and in packages/pkg
                            #   -> root: all 9 checks pass (exit 0, blind);
                            #      packages/pkg: drift on skills/demo/SKILL.md (exit 1)
```

## `apm prune` has no global scope (`prune-global-scope/`) — RFE

### Setup

A trivial local package
([`prune-global-scope/pkg/`](prune-global-scope/pkg/)) with one skill, installed
to **user scope** (`apm install -g`). Every `-g` command in this demo is pinned
to an isolated `HOME` (`prune-global-scope/fake-home/`, git-ignored) via the
Makefile, so it never touches the real `~/.apm/` or `~/.claude/`.

`apm prune` has no `--global`/`-g` flag and always operates on the current
project, so orphaned packages under `~/.apm/apm_modules/` cannot be pruned — even
though `install`/`uninstall`/`update`/`lock`/`compile` are all scope-aware. The
only workaround (`cd ~/.apm && apm prune`) removes the `apm_modules/` tree but
leaks the deployed primitive in the home deploy root, because user-scope
deployment targets `~/` while a cwd-scoped prune only cleans under `~/.apm/`.
Write-up (as a feature request for `--global`) in
[`feature-prune-global-scope.md`](feature-prune-global-scope.md).

`make clean` removes the isolated fake `HOME` (manifest, `apm_modules/`, deployed
mirror, caches).

### Reproduce

```bash
apm --version               # 0.25.0
cd prune-global-scope

make flag                   # `apm prune -g`
                            #   -> Error: No such option: -g
                            #      (sibling `apm uninstall -g` has it)

make orphan                 # install -g + orphan it + `cd ~/.apm && apm prune`
                            #   -> apm_modules/_local/pkg: REMOVED;
                            #      ~/.agents/skills/demo/SKILL.md: SURVIVES (orphaned)
```
