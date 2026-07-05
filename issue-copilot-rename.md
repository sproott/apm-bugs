# [BUG] Copilot hook events are never renamed to camelCase

**Describe the bug**

The docs state hook files may be authored in the Claude (`PreToolUse`,
`PostToolUse`) or Copilot (`preToolUse`, `postToolUse`) shape and that "events
are renamed per target during merge." For the **copilot** target this rename
never happens: `apm install` warns about a casing mismatch and deploys the
event unrenamed.

Root cause is in
[`hook_integrator.py`](https://github.com/microsoft/apm/blob/main/src/apm_cli/integration/hook_integrator.py):

- `_HOOK_EVENT_MAP` has entries for `claude`, `gemini`, and `kiro`, but **no
  `copilot` entry** — while `_HOOK_EVENT_EXPECTED_CASING["copilot"] =
  "camelCase"`, so `_emit_hook_event_diagnostics` flags `PostToolUse` as an
  unmappable "casing mismatch."
- The Copilot deploy path also calls the diagnostic with a hard-coded empty map
  (`_emit_hook_event_diagnostics(..., "copilot", {})`) and writes the events
  through unchanged, so it never applies a mapping even if one existed.

A fix needs both: a `copilot` mapping (`PreToolUse`→`preToolUse`,
`PostToolUse`→`postToolUse`) and for the Copilot path to apply it during merge.

**To Reproduce**
1. Use a package with a hook authored in the Claude shape
   (`{ "hooks": { "PostToolUse": [...] } }`) and `copilot` in `targets`.
2. Run `apm install`.
3. See the casing warning; `.github/hooks/<pkg>-<file>.json` keeps `PostToolUse`.

```
Hook events for target 'copilot' may not be recognized: PostToolUse. Target
expects camelCase (e.g. preToolUse). Rename events to match the camelCase
convention, then reinstall.
WARNING apm_cli.integration.hook_integrator target copilot: hook event casing mismatch (no mapping): PostToolUse
```

(In the same run the `gemini` target *is* renamed correctly to `AfterTool`,
confirming the per-target rename machinery works for targets that have a mapping.)

**Expected behavior**

The Copilot target renames `PostToolUse` → `postToolUse` automatically and emits
no warning.

**Environment**
 - OS: Linux (Arch Linux)
 - Python Version: 3.12
 - APM Version: 0.23.1

**Logs**

See the warning block above.

**Additional context**

Minimal reproduction: https://github.com/sproott/apm-bugs (`make copilot-rename` in `hooks/`).
