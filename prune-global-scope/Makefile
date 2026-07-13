.PHONY: clean flag orphan reproduce

# All user-scope (`-g`) commands are pinned to an isolated fake HOME so the
# demo never touches the real ~/.apm/ or ~/.claude/. User scope resolves its
# manifest, modules, and deploy root from $HOME, so overriding HOME is enough.
HOME_DIR := $(CURDIR)/fake-home
PKG      := $(CURDIR)/pkg
APM      := env HOME=$(HOME_DIR) apm

# Remove the isolated home (manifest, apm_modules/, deployed mirror, caches).
clean:
	@rm -rf $(HOME_DIR)

# The headline gap: prune has no user-scope switch at all.
flag:
	@echo "--- apm prune -g (user scope) ---"
	@$(APM) prune -g 2>&1 || true
	@echo
	@echo "--- for contrast, the sibling destructive command has it ---"
	@apm uninstall --help 2>&1 | grep -E '\-g, --global' || true

# Install a package to user scope, orphan it (drop it from ~/.apm/apm.yml),
# then show the only workaround -- `cd ~/.apm && apm prune` -- is incomplete:
# it removes the apm_modules/ tree but leaks the deployed primitive, because
# the user-scope deploy root is $HOME while a cwd-scoped prune only cleans
# files under ~/.apm/.
orphan: clean
	@echo "--- install demo-global-pkg to user scope ---"
	@$(APM) install -g $(PKG) 2>&1 | tail -3
	@echo
	@echo "--- installed under ~/.apm/, deployed under ~/ ---"
	@find $(HOME_DIR) -path '*/apm_modules/*/apm.yml' -o -path '*skills/demo*' -type f 2>/dev/null | sed 's#$(HOME_DIR)#~#' | sort
	@echo
	@echo "--- orphan it: rewrite ~/.apm/apm.yml with no dependencies ---"
	@printf 'name: fake-home\nversion: 1.0.0\ndescription: APM project for fake-home\nauthor: Developer\ndependencies:\n  apm: []\n  mcp: []\nincludes: auto\nscripts: {}\n' > $(HOME_DIR)/.apm/apm.yml
	@echo
	@echo "--- workaround: cd ~/.apm && apm prune ---"
	@cd $(HOME_DIR)/.apm && $(APM) prune 2>&1 | sed 's/^/    /'
	@echo
	@echo "--- result: apm_modules pruned, but deployed skill survives orphaned ---"
	@echo -n "  ~/.apm/apm_modules pkg : "; \
		if [ -z "$$(find $(HOME_DIR)/.apm/apm_modules -name apm.yml 2>/dev/null)" ]; then echo "REMOVED"; else echo "still present"; fi
	@echo -n "  deployed skill under ~/: "; \
		find $(HOME_DIR) -path '*skills/demo*' -type f 2>/dev/null | sed 's#$(HOME_DIR)#~#' | head -1; \
		if [ -n "$$(find $(HOME_DIR) -path '*skills/demo*' -type f 2>/dev/null)" ]; then echo "  <- SURVIVES (orphaned in user deploy root)"; fi

reproduce: flag orphan
