.PHONY: clean tree audit

# Remove the lockfile, installed apm_modules/, and generated target dirs.
# The checked-in apm.yml files are never touched.
clean:
	@rm -rf \
		apm.lock.yaml \
		apm_modules/ \
		.claude/ \
		.codex \
		.cursor/ \
		.gemini/ \
		.github/ \
		.opencode/ \
		.windsurf/

# Reproduce: install, then render the tree. The remote fixture's local
# sub-packages appear self-nested (root -> inner -> leaf, repeated) up to the
# depth-5 cap.
tree: clean
	apm install
	apm deps tree

audit: clean
	apm install
	apm deps list
	apm audit --ci
