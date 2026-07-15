# [BUG] Copilot user-scope hook commands are repo-relative, unresolvable from any cwd ≠ $HOME

**Describe the bug**

A user-scope (`apm install -g`) Copilot install deploys a hook file to
`~/.copilot/hooks/`, but leaves the hook `command` a repo-relative path such as
`node .copilot/hooks/scripts/<pkg>/scripts/notify.js`. A hook command is resolved
against the process working directory, not the file that declares it, so this only
works when Copilot happens to run from `$HOME` — from any project directory the
script is never found.

Every other user-scope target rewrites hook commands to absolute paths for exactly
this reason. `integrate_hooks_for_target` threads `user_scope` into the merge-based
targets (Claude, Codex, Cursor, …) and Kiro, which pass it down to
[`_rewrite_command_for_target`](https://github.com/microsoft/apm/blob/main/src/apm_cli/integration/hook_integrator.py)
as a `deploy_root`; when `deploy_root` is set the rewritten script path is resolved
to an absolute path. The Copilot branch is the sole outlier: it dispatches to
`integrate_package_hooks`
([hook_integrator.py](https://github.com/microsoft/apm/blob/main/src/apm_cli/integration/hook_integrator.py))
without `user_scope`, and that method calls `_rewrite_hooks_data` with no
`deploy_root`, so Copilot commands stay relative at every scope.

This is the same defect class as #1310 / #1354 (`--target claude` needed absolute
hook paths in `~/.claude/settings.json`); the fix there never covered Copilot. The
opposite constraint from #1394 — project-scope checked-in configs must stay
relative — is preserved by keying on scope, so the fix is to thread `user_scope`
through the Copilot branch exactly as the merge targets already do.

**To Reproduce**

Steps to reproduce the behavior:

1. `cd copilot-user-hook-relative-path`
2. `make copilot` — install a package with a script-backed hook to user scope,
   Copilot target, under an isolated fake `$HOME`
3. See the emitted `command` in `~/.copilot/hooks/pkg-notify.json` is a repo-relative
   path (`node .copilot/hooks/scripts/pkg/scripts/notify.js`)
4. `make claude` — install the same package/hook to the Claude target at user scope
5. See its emitted `command` is absolute

**Expected behavior**

A hook installed at user scope runs regardless of the working directory Copilot is
launched from. The deployed `command` points at an absolute script path, matching
every other user-scope target.

**Environment (please complete the following information):**
 - OS: Linux (Arch Linux)
 - Python Version: 3.14
 - APM Version: 0.25.0 (d73e6ac)

**Logs**

```
--- install demo-hook-pkg to user scope, Copilot target ---
[*] Installed 1 APM dependency in 1.7s.

--- deployed hook file under ~/.copilot/hooks/ ---
~/.copilot/hooks/pkg-notify.json
~/.copilot/hooks/scripts/pkg/package.json

--- the emitted command (note: repo-relative, not absolute) ---
    "command": "node .copilot/hooks/scripts/pkg/scripts/notify.js"

--- install the same package to user scope, Claude target ---
[*] Installed 1 APM dependency in 1.7s.

--- the emitted command in ~/.claude/settings.json (absolute) ---
    "command": "node /home/.../fake-home/.claude/hooks/pkg/scripts/notify.js"
```

**Additional context**

Minimal reproduction: https://github.com/sproott/apm-bugs
(`make copilot` / `make claude` / `make reproduce` in
`copilot-user-hook-relative-path/`).
