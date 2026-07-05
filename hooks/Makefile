.PHONY: clean copilot-rename audit

clean:
	@rm -rf \
		apm.lock.yaml \
		.claude/ \
		.codex \
		.cursor/ \
		.gemini/ \
		.github/ \
		.opencode/ \
		.windsurf/

copilot-rename: clean
	apm install

audit: clean
	apm install
	apm audit --ci || exit 0
	@echo "Try again on existing lockfile..."
	apm install
	apm audit --ci
