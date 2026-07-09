# [BUG] `apm deps tree` self-nests a remote package's sub-packages recursively

**Describe the bug**

When a project depends on a **remote** package that itself declares local
`path:` dependencies on sibling sub-packages in the same repo, `apm deps tree`
renders those sub-packages recursively nested inside one another — each
sub-package (and every skill it carries) reappears under every other, expanding
until the hard-coded depth-5 recursion cap. Resolution is correct; only the tree
**display** is wrong.

Root cause: the tree builder identifies parent and child by **`repo_url`**, which
is not unique across virtual sub-packages of the same repo. All sub-packages of
one repo share a single `repo_url`, so they collapse into one `children_map`
bucket and each becomes a child of every other.

- A remote parent's local `path:` deps are expanded to inherit the parent's
  `repo_url`, differing only by `virtual_path`
  ([`apm_resolver.py` `_inherit_remote_parent_fields`](https://github.com/microsoft/apm/blob/main/src/apm_cli/deps/apm_resolver.py)).
  The lockfile then records every sub-package with the **same** `repo_url` and a
  `resolved_by` that is also just that `repo_url`.
- [`_build_dep_tree`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/deps/cli.py)
  keys `children_map` by `dep.resolved_by` (the repo_url), so all of the repo's
  transitive sub-packages land in `children_map["<repo_url>"]` together.
- [`_add_tree_children`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/deps/cli.py)
  looks up `children_map[parent_repo_url]` and recurses with `child_dep.repo_url`
  — always the same repo_url — so the same bucket is re-emitted at every level:

  ```python
  def _add_tree_children(parent_branch, parent_repo_url, children_map, has_rich, depth=0):
      kids = children_map.get(parent_repo_url, [])
      for child_dep in kids:
          child_name = _dep_display_name(child_dep)
          child_branch = parent_branch.add(...) if has_rich else child_name
          if depth < 5:  # Prevent infinite recursion
              _add_tree_children(child_branch, child_dep.repo_url, children_map, has_rich, depth + 1)
  ```

  The `depth < 5` guard is the only thing stopping true infinite recursion.

The lockfile itself is correct: the three sub-packages each have a distinct
`virtual_path` and correct `depth`, and `apm deps list` (flat) shows exactly
three packages. Only `apm deps tree` — which keys on `repo_url` instead of
`get_unique_key()` (repo_url + virtual_path) — is wrong.

**To Reproduce**

Reproduction directory:
[`deps-tree-subpackage-recursion/`](deps-tree-subpackage-recursion/). The
consumer depends on a fixture published as a remote virtual sub-package
(`sproott/apm-bugs/deps-tree-subpackage-recursion/fixture#main`); the fixture
chains three same-repo sub-packages via local paths: root → `./packages/inner`
→ `../leaf` (leaf carries one skill).

1. `cd deps-tree-subpackage-recursion`
2. `make tree` — clean, `apm install`, then `apm deps tree`
3. The three sub-packages appear self-nested five levels deep; `fixture-leaf`
   (and its skill) is repeated at every level.

**Expected behavior**

`apm deps tree` shows each installed package once in its true parent→child
position: root → inner → leaf, three nodes, no repetition — matching what the
lockfile records and what `apm deps list` prints.

**Environment (please complete the following information):**
 - OS: Linux
 - Python Version: 3.12
 - APM Version: 0.24.0 (b915f8d)
 - VSCode Version (if relevant): n/a

**Logs**

Install resolves exactly three packages, each distinct:

```
$ apm deps list
│ sproott/apm-bugs/deps-tree-subpackage-recursi… │ 1.0.0 │ github │ … │ - │
│ sproott/apm-bugs/deps-tree-subpackage-recursi… │ 1.0.0 │ github │ … │ - │
│ sproott/apm-bugs/deps-tree-subpackage-recursi… │ 1.0.0 │ github │ … │ 1 │   # leaf: 1 skill
```

Lockfile (correct — distinct `virtual_path`, correct `depth`):

```yaml
- repo_url: sproott/apm-bugs
  name: fixture-root
  virtual_path: deps-tree-subpackage-recursion/fixture
- repo_url: sproott/apm-bugs
  name: fixture-inner
  virtual_path: deps-tree-subpackage-recursion/fixture/packages/inner
  depth: 2
  resolved_by: sproott/apm-bugs
- repo_url: sproott/apm-bugs
  name: fixture-leaf
  virtual_path: deps-tree-subpackage-recursion/fixture/packages/leaf
  depth: 3
  resolved_by: sproott/apm-bugs
```

`apm deps tree` (excerpt — full output nests to depth 5):

```
deps-tree-subpackage-recursion (local)
└── sproott/apm-bugs/deps-tree-subpackage-recursion/fixture@1.0.0
    ├── …/fixture/packages/inner@1.0.0
    │   ├── …/fixture/packages/inner@1.0.0
    │   │   ├── …/fixture/packages/inner@1.0.0
    │   │   │   ├── …/fixture/packages/inner@1.0.0
    │   │   │   │   ├── …/fixture/packages/inner@1.0.0
    │   │   │   │   └── …/fixture/packages/leaf@1.0.0
    │   │   │   └── …/fixture/packages/leaf@1.0.0
    │   │   │       ├── …/fixture/packages/inner@1.0.0
    │   │   │       └── …/fixture/packages/leaf@1.0.0
    │   …
    └── …/fixture/packages/leaf@1.0.0
        ├── …/fixture/packages/inner@1.0.0
        └── …/fixture/packages/leaf@1.0.0
            …
```

**Additional context**

Display-only; resolution and the lockfile are correct. The same repo_url-vs-unique-key
granularity issue that produces the wrong tree here also underlies orphan-detection
mismatches filed separately for local sub-package layouts.
