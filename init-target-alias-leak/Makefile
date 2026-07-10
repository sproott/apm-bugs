.PHONY: clean control multi agents reproduce

# apm.yml is *generated* by `apm init` in every scenario here -- it is the
# artifact under test, so it is git-ignored rather than checked in. Only the
# .apm/ primitive is checked in; it exists so `apm install` gets far enough to
# validate the manifest's targets: field.
clean:
	@rm -rf \
		*/apm.yml \
		*/apm.lock.yaml \
		*/apm_modules/ \
		*/.gitignore \
		*/.claude/ \
		*/.codex/ \
		*/.cursor/ \
		*/.gemini/ \
		*/.github/ \
		*/.opencode/ \
		*/.windsurf/

# Control: a single-token --target survives round-trip. install succeeds.
control: clean
	cd control && apm init --target copilot --yes
	cd control && grep -A2 '^targets:' apm.yml
	cd control && apm install

# `--target claude,copilot` is alias-resolved to the internal name `vscode`
# and written to apm.yml. `apm install` then rejects its own manifest.
multi: clean
	cd multi && apm init --target claude,copilot --yes
	cd multi && grep -A3 '^targets:' apm.yml
	-cd multi && apm install

# `--target agents` is passed through verbatim. `agents` is a CLI-accepted
# alias but not a manifest-canonical target, so install rejects it too.
agents: clean
	cd agents && apm init --target agents --yes
	cd agents && grep -A2 '^targets:' apm.yml
	-cd agents && apm install

reproduce: control multi agents
