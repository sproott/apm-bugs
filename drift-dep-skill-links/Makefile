.PHONY: clean audit

# Remove the lockfile, installed apm_modules/, and generated target dirs.
# The checked-in apm.yml files are never touched.
clean:
	@rm -rf \
		apm.lock.yaml \
		apm_modules/ \
		.claude/

audit: clean
	apm install
	apm audit --ci
