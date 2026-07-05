.PHONY: clean audit

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

audit: clean
	apm install
	apm audit --ci

prune: clean
	apm install
	apm prune
	@echo "Dev dependencies incorrectly pruned"
	apm install
	@echo "Dev dependencies installed again"
