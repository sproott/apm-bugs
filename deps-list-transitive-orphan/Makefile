.PHONY: clean deps-list prune

# Remove the lockfile, installed apm_modules/, and generated target dirs.
# The checked-in apm.yml files are never touched.
clean:
	@rm -rf \
		apm.lock.yaml \
		apm_modules/ \
		.claude/

# BUG: deps list flags the transitive dep _local/sub-package as orphaned,
# even though the lockfile records it as a legitimate depth-2 install.
deps-list: clean
	apm install
	apm deps list

# Contrast: prune correctly reports NO orphans for the same tree.
prune: clean
	apm install
	apm prune --dry-run
