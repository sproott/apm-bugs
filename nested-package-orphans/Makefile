.PHONY: clean audit prune

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

audit: clean
	apm install
	apm deps list
	apm audit --ci

prune: clean
	apm install
	apm prune
	apm install
