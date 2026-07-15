.PHONY: clean reproduce copilot claude

# All user-scope (`-g`) commands are pinned to an isolated fake HOME so the
# demo never touches the real ~/.apm/, ~/.copilot/, or ~/.claude/. User scope
# resolves its manifest, modules, and deploy root from $HOME, so overriding
# HOME is enough.
HOME_DIR := $(CURDIR)/fake-home
PKG      := $(CURDIR)/pkg
APM      := env HOME=$(HOME_DIR) apm

# Remove the isolated home (manifest, apm_modules/, deployed mirror, caches).
clean:
	@rm -rf $(HOME_DIR)

# The bug: user-scope Copilot install leaves the hook command repo-relative.
# The deployed hook lands in ~/.copilot/hooks/, but its command is
# `node .copilot/hooks/scripts/...`, resolved against the process cwd -- so
# it only works when Copilot happens to run from $HOME.
copilot: clean
	@echo "--- install demo-hook-pkg to user scope, Copilot target ---"
	@$(APM) install -g $(PKG) 2>&1 | tail -3
	@echo
	@echo "--- deployed hook file under ~/.copilot/hooks/ ---"
	@find $(HOME_DIR)/.copilot/hooks -name '*.json' 2>/dev/null | sed 's#$(HOME_DIR)#~#'
	@echo
	@echo "--- the emitted command (note: repo-relative, not absolute) ---"
	@grep -h '"command"' $(HOME_DIR)/.copilot/hooks/*.json 2>/dev/null | sed 's/^[[:space:]]*/    /'

# For contrast: the same package, same hook, installed to the Claude target at
# user scope emits an absolute path -- the behaviour Copilot should mirror.
claude: clean
	@echo "--- install the same package to user scope, Claude target ---"
	@$(APM) install -g --target claude $(PKG) 2>&1 | tail -3
	@echo
	@echo "--- the emitted command in ~/.claude/settings.json (absolute) ---"
	@grep -h '"command"' $(HOME_DIR)/.claude/settings.json 2>/dev/null | sed 's/^[[:space:]]*/    /'

reproduce: copilot claude
