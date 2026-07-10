# [BUG] `apm init --target` writes a `targets:` value that `apm install` then rejects

**Describe the bug**

`apm init --target <list>` persists the `--target` flag's *parsed* value straight
into `apm.yml`'s `targets:` field. But the flag parser and the manifest validator
do not share a target vocabulary, so `init` can write a token that `install` then
refuses to read back — a project that is broken the moment it is scaffolded. Two
tokens leak:

- `vscode` — from alias resolution of `copilot`/`agents`/`vscode` in any
  **multi-token** `--target` (`--target claude,copilot` → `targets: [claude, vscode]`).
- `agents` — passed through verbatim from a **single-token** `--target agents`,
  a CLI-accepted (and deprecated) alias that is not manifest-canonical.

Consequence: the scaffolded `apm.yml` cannot be installed. `apm install` aborts
with `Unknown target 'vscode'` (exit 2) and the user must hand-edit a manifest
they never wrote by hand. The manifest `init` writes even advertises the bad
tokens — its header comment reads `Accepted values: vscode, agents, copilot, ...`,
though `vscode` and `agents` are exactly the two values `parse_targets_field()`
rejects.

**Root cause: target vocabulary is fragmented across the codebase**

The `init` leak is a symptom of a deeper problem: the CLI carries several
different notions of a "canonical" target, with no single source of truth.

Two disjoint canonical sets disagree on the copilot/vscode family's canonical
spelling:

- [`ALL_CANONICAL_TARGETS`](https://github.com/microsoft/apm/blob/main/src/apm_cli/core/target_detection.py)
  (`core/target_detection.py`) — canonical is `vscode`.
- [`CANONICAL_TARGETS`](https://github.com/microsoft/apm/blob/main/src/apm_cli/core/apm_yml.py)
  (`core/apm_yml.py`, the manifest validator) — canonical is `copilot`; `vscode`
  is not in it.

Three-plus alias maps, some inverting each other:

- `TARGET_ALIASES` (`target_detection.py`): `copilot`/`agents`/`vscode` → `vscode`.
- `RUNTIME_TO_CANONICAL_TARGET` (`integration/targets.py`):
  `vscode`/`agents`/`intellij` → `copilot` — the **opposite** direction.
- `_VSCODE_TARGET_ALIASES` (`compilation/agents_compiler.py`): `("copilot",
  "agents")`, applied inline.
- `_CROSS_TARGET_MAPS` (`bundle/lockfile_enrichment.py`).

Plus roughly a dozen hand-rolled checks that re-derive the same mapping in situ —
`if target == "vscode"`, `"vscode" if target in ("copilot", "agents") else
target`, `"vscode" in target` — spread across `compile/cli.py`,
`mcp_integrator.py`, `hook_integrator.py`, `registry/operations.py`, and
`lockfile_enrichment.py`.

[`parse_target_field()`](https://github.com/microsoft/apm/blob/main/src/apm_cli/core/target_detection.py)
resolves aliases only for multi-token input (a documented CLI-contract
asymmetry). In-memory consumers tolerate the unresolved token;
[`init`](https://github.com/microsoft/apm/blob/main/src/apm_cli/commands/init.py)
is the one path that writes it to disk, where `parse_targets_field()` validates
against the *other* canonical set and aborts.

**Recommended fix**

Point-patching `init` (resolve to a manifest-canonical token before writing)
closes this repro but leaves the fragmentation that produced it. The durable fix
is a single source of truth: one canonical target set, one alias→canonical
resolver, one direction of canonicalization — with `init`, `install`, and
`compile` all routing target values through it. That also subsumes the
consumption-side bugs (#1746, #820) noted below, which are the same fragmentation
surfacing on the read side.

**To Reproduce**

1. In an empty directory containing one primitive (e.g.
   `.apm/instructions/demo.instructions.md`, so `install` gets far enough to
   validate `targets:`), run `apm init --target claude,copilot --yes`. The flag
   is echoed already renamed:

```
[i] Targets set: claude, vscode (via --target flag)
```

2. Inspect the generated `apm.yml` — `copilot` was written as `vscode`:

```yaml
targets:
- claude
- vscode
```

3. Run `apm install` — it rejects its own scaffold, exit 2:

```
[>] Installing dependencies from apm.yml...
[x] Unknown target 'vscode'

Valid targets: antigravity, claude, codex, copilot, cursor, gemini, kiro,
opencode, windsurf
[!] Install interrupted after 7.0s.
```

4. Same failure via the single-token alias path: `apm init --target agents --yes`
   writes `targets: [agents]`, and `apm install` aborts with `Unknown target
   'agents'` (exit 2).

5. Control: `apm init --target copilot --yes` writes `targets: [copilot]` and
   `apm install` succeeds. The same token survives alone and is renamed in a list.

**Expected behavior**

`apm init --target <list>` writes exactly the targets the user asked for, and the
resulting `apm.yml` installs. `--target claude,copilot` yields
`targets: [claude, copilot]`, and `apm install` exits 0. Any `--target` value the
CLI accepts must produce a manifest the CLI accepts — or be rejected up front by
`apm init` rather than at the next command.

**Environment**
 - OS: Linux (Arch Linux)
 - Python Version: 3.14
 - APM Version: 0.24.0

**Logs**

See the captured `apm init` and `apm install` output above.

**Additional context**

Minimal reproduction: https://github.com/sproott/apm-bugs
(`init-target-alias-leak/` — `make control`, `make multi`, `make agents`).

#1746 (`--target claude,copilot` silently drops Copilot from the deploy) and #820
(mixed `target: opencode,claude,copilot,agents` deploys nothing) are the same
vocabulary fragmentation surfacing on the *consumption* side — the alias-resolved
value is mishandled after parsing, so it fails silently. This issue is the
*production* side: `init` persists a CLI-vocabulary token into a manifest
validated against a narrower vocabulary, so it fails loud (`Unknown target`, exit
2). A unified target vocabulary would remove the root cause of all three.
