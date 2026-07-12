.PHONY: clean stale audit reproduce

SENTINEL = packages/pkg/.apm/skills/demo/SKILL.md

# Remove lockfiles, installed apm_modules/, generated mirrors, and the
# generated sentinel skill source. The checked-in apm.yml files are never
# touched.
clean:
	@rm -rf \
		apm.lock.yaml \
		apm_modules/ \
		.claude/ \
		packages/pkg/apm.lock.yaml \
		packages/pkg/apm_modules/ \
		packages/pkg/.claude/ \
		packages/pkg/.apm/

# Write the sentinel skill source at a given marker version (sentinel-v1 /
# sentinel-v2). The v1->v2 rewrite stands in for any real edit to the nested
# package's authored primitives.
sentinel-v%:
	@mkdir -p packages/pkg/.apm/skills/demo
	@printf -- '---\nname: demo\ndescription: Sentinel skill for the recursive install/audit demo.\n---\n\nmarker: v%s\n' '$*' > $(SENTINEL)
	@echo "sentinel source at marker: v$*"

# Root `apm install` refreshes only the root mirror; the nested package's own
# mirror silently stays at v1.
stale: clean
	$(MAKE) sentinel-v1
	cd packages/pkg && apm install
	apm install
	$(MAKE) sentinel-v2
	apm install
	@echo
	@echo "--- deployed mirrors after root apm install ---"
	@grep -rH 'marker:' .claude packages/pkg/.claude

audit: stale
	@echo
	@echo "--- apm audit --ci at root (blind to the stale nested mirror) ---"
	apm audit --ci
	@echo
	@echo "--- apm audit --ci inside packages/pkg (catches the drift) ---"
	cd packages/pkg && apm audit --ci

reproduce: audit
