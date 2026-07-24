.PHONY: clean project user

# Regenerate all installed state. Never touches the checked-in apm.yml files.
clean:
	@rm -rf \
		consumer/apm_modules/ \
		consumer/apm.lock.yaml \
		consumer/.claude/ \
		consumer/.gitignore \
		fake-home/

# BUG (project scope): `apm deps list` renders the local dep as `_local/mypkg`,
# but `apm uninstall _local/mypkg` reports "not found in apm.yml" (a silent
# exit-0 no-op). The only accepted key is the declared `../mypkg` path.
project: clean
	cd consumer && apm install
	@echo '--- apm deps list (shows key `_local/mypkg`) ---'
	cd consumer && apm deps list
	@echo '--- apm uninstall _local/mypkg (the displayed key -> not found) ---'
	cd consumer && apm uninstall _local/mypkg

# BUG (user scope): same mismatch under --global. Installed by ABSOLUTE path
# (relative paths are rejected at user scope), so `apm deps list -g` shows
# `_local/mypkg` while the only accepted uninstall key is the machine-specific
# absolute path -- which is displayed nowhere. HOME is isolated to fake-home/
# so the real ~/.apm is untouched.
user: clean
	HOME=$(CURDIR)/fake-home apm install -g $(CURDIR)/mypkg
	@echo '--- apm deps list -g (shows key `_local/mypkg`) ---'
	HOME=$(CURDIR)/fake-home apm deps list -g
	@echo '--- apm uninstall -g _local/mypkg (the displayed key -> not found) ---'
	HOME=$(CURDIR)/fake-home apm uninstall -g _local/mypkg
